# InfraReady : Ansible sur ops01

## Objet

Playbooks Ansible du projet InfraReady, exécutés sur ops01 :
- nginx (bloc D) : installation, démarrage et activation au boot ;
- administration Linux (bloc C) : comptes, groupes, permissions, disque de test, mise à jour contrôlée, journaux.

Windows n'est pas couvert par ce dépôt pour l'instant.

## Prérequis

- Machine : ops01, Ubuntu Server 26.04.1 LTS constaté (le dossier technique indique 24.04, voir Écarts).
- Ansible core 2.20.1, Python 3.14 (`/usr/bin/python3`).
- Utilisateur `ops01` avec sudo. Le mot de passe sudo est saisi à l'invite à chaque exécution, jamais écrit dans un fichier.

## Structure

```
inventory.ini          cible : groupe linux, ops01 en connexion locale
group_vars/linux.yml   variables du groupe (élévation, interpréteur Python, paquet et service nginx)
group_vars/windows.yml  connexion WinRM vers WIN-DC01 (mot de passe lu dans l'environnement)
windows.yml             playbook Windows : unité InfraReady, groupes, comptes de test, journal Système
windows-exporter.yml    playbook Windows : Windows Exporter 0.32.0 sur WIN-DC01 (port 9182)
nginx.yml              playbook du bloc D : installe, démarre et active nginx
c-linux.yml            playbook du bloc C : comptes, groupes, permissions, disque de test, mise à jour, contrôles
f-supervision.yml      playbook du bloc F : Node Exporter, Prometheus, Grafana, règles d'alerte, tableau de bord
maintenance.yml        playbook du bloc E : mise à jour sans suppression, journal de maintenance, contrôles
.gitignore             exclut clés, *.vault, secrets et .env
README.md              ce fichier
.github/workflows/ci.yml  pipeline GitHub Actions : yamllint, ansible-lint, syntaxe des playbooks
requirements-ci.txt, requirements.yml  versions figées des outils et des collections
.yamllint, .ansible-lint  configuration des contrôles de qualité
```

Hors du dépôt, à ne pas versionner : le résumé des changements (`/resume.txt`) et le journal de travail (`/home/ops01/infraready-journal.txt`).

## Commandes

Depuis `~/infraready-ansible` :

```
# Vérifier la syntaxe, sans rien modifier
ansible-playbook -i inventory.ini nginx.yml --syntax-check
ansible-playbook -i inventory.ini c-linux.yml --syntax-check
ansible-playbook -i inventory.ini f-supervision.yml --syntax-check
ansible-playbook -i inventory.ini maintenance.yml --syntax-check

# Vérifier qu'Ansible atteint la cible
ansible -i inventory.ini linux -m ping

# Exécuter (mot de passe sudo demandé à l'invite)
sudo ansible-playbook -i inventory.ini nginx.yml
sudo ansible-playbook -i inventory.ini c-linux.yml
sudo ansible-playbook -i inventory.ini f-supervision.yml
sudo ansible-playbook -i inventory.ini maintenance.yml

# Windows : mots de passe saisis dans le terminal, jamais écrits dans un fichier
read -rs -p "Mot de passe admin : " WIN_ADMIN_PASSWORD; echo; export WIN_ADMIN_PASSWORD
read -rs -p "Mot de passe des comptes de test : " TEST_USER_PASSWORD; echo; export TEST_USER_PASSWORD
ansible -i inventory.ini windows -m ansible.windows.win_ping
ansible-playbook -i inventory.ini windows.yml
ansible-playbook -i inventory.ini windows-exporter.yml
```

Relancer un playbook immédiatement après doit donner `changed=0` : c'est le test d'idempotence.

Pourquoi `sudo ansible-playbook` et non `ansible-playbook -K` : avec `-K`, Ansible a expiré sur le prompt sudo (« Timed out waiting for become success or become password prompt »). La commande avec `sudo` contourne ce point, au prix d'un playbook exécuté entièrement en root (voir Écarts).

## Cible

Le groupe `linux` ne contient que `ops01`. Ansible agit en local : aucune connexion SSH n'est utilisée.

## Bloc D : nginx (`nginx.yml`)

Installe nginx, le démarre et l'active au démarrage.

Contrôles :

```
systemctl is-active nginx     # attendu : active
systemctl is-enabled nginx    # attendu : enabled
curl -I http://localhost      # attendu : HTTP/1.1 200 OK
```

Ce que ces contrôles prouvent : le service tourne, il démarre au boot (preuve : après le redémarrage de ops01 du 22 octobre 2026 à 08:22 UTC, nginx a redémarré sans commande manuelle), et il répond en HTTP en local.
Test depuis un autre poste : `curl -I http://192.168.50.20` depuis la WSL répond `HTTP/1.1 200 OK`.
Ce qu'ils ne prouvent pas : le contenu servi (la page n'a pas été lue).

## Bloc C : administration Linux (`c-linux.yml`)

- **Comptes et groupes** : groupe `devs` ; compte `testdev1` membre de `devs` ; compte `testext` hors du groupe. Aucun mot de passe n'est défini pour ces comptes.
- **Permissions** : `/srv/projet`, propriétaire `root`, groupe `devs`, droits `2770` (setgid, pour que les nouveaux fichiers héritent du groupe).
- **Disque de test** : image de 512 Mo `/var/lib/disque-test.img`, formatée en ext4, montée sur `/mnt/disque-test` et ajoutée à `/etc/fstab`. Écart : image fichier montée en boucle, faute de disque virtuel supplémentaire.
- **Mise à jour contrôlée** : un seul paquet, `tmux` (3.6a-2ubuntu0.1 vers 3.6a-2ubuntu0.2). Les autres paquets en attente ne sont pas appliqués. Écart.
- **Journaux** : consultés en lecture seule (`journalctl -p warning -b`, `journalctl -u nginx`). L'utilisateur `ops01` appartient au groupe `adm`, ce qui permet de lire les journaux sans root.
- **Contrôles intégrés au playbook** : `testdev1` peut écrire dans `/srv/projet`, `testext` ne le peut pas, le disque est monté, nginx est actif. Un `assert` final échoue si l'un de ces contrôles est faux.

## Bloc F : supervision Linux (`f-supervision.yml`)

- **Node Exporter** (paquet Ubuntu `prometheus-node-exporter`) : expose les métriques de la machine sur le port 9100.
- **Prometheus** (paquet Ubuntu `prometheus`) : collecte la cible `node` (`localhost:9100`) toutes les 15 secondes. La métrique `up{job="node"}` vaut 1 quand le serveur répond.
- **Règles d'alerte** (`/etc/prometheus/rules/infraready.yml`) : `ServeurInjoignable` (`up == 0` pendant 1 minute, critique) et `DisqueRacinePlein` (système de fichiers `/` au-delà de 80 % pendant 5 minutes, avertissement).
- **Grafana** (dépôt officiel Grafana, car le paquet n'est pas dans les dépôts Ubuntu) : source de données Prometheus provisionnée, et tableau de bord « InfraReady - Linux ops01 » (cinq panneaux, seuil à 80 % sur la jauge du disque).

Ce que ça prouve : la machine est supervisée, Prometheus évalue les seuils et Grafana affiche les données. Pendant les tests, aucune alerte n'était déclenchée (disque racine à 53 %).
Ce que ça ne prouve pas : la notification (aucun Alertmanager n'est configuré) et la partie Windows (`windows_exporter`, qui dépend du transport Windows).

## Bloc E : maintenance et mises à jour (`maintenance.yml`)

- Rafraîchit l'index APT, avec de nouvelles tentatives car le miroir Ubuntu peut être instable, puis applique les mises à jour **sans suppression de paquets** (`upgrade: safe`).
- Compte les paquets en attente avant et après, et signale si un redémarrage est requis. Le playbook ne redémarre jamais la machine.
- Vérifie ensuite nginx, Prometheus, Node Exporter et Grafana.
- Écrit `/var/log/infraready-maintenance.log` seulement si des paquets ont été mis à jour : une relance ne change rien.

Ce que ça prouve : la machine est à jour à la date du passage, les services sont vérifiés après la mise à jour, et le passage est idempotent.
Ce que ça ne prouve pas : la politique de mise à jour (calendrier, paquets exclus), qui reste à décider.

## Bloc B et D Windows (`windows.yml`) et supervision Windows (`windows-exporter.yml`)

Cible : WIN-DC01 (`win-q2qrglap1ur`, 192.168.50.10), domaine `infraready.test`. Connexion WinRM en HTTP (port 5985), authentification NTLM, compte `INFRAREADY\Administrateur`. Les mots de passe ne sont jamais dans les fichiers : ils sont saisis dans le terminal.

- **Unité d'organisation** : `OU=InfraReady,DC=infraready,DC=test`.
- **Groupes de sécurité globaux** : `GRP_Helpdesk` (support niveau 1) et `GRP_Comptables` (comptabilité).
- **Comptes de test** : `test.helpdesk` dans `GRP_Helpdesk`, `test.compta` dans `GRP_Comptables`. Comptes actifs, mot de passe qui n'expire pas. Le mot de passe est défini à la création seulement (`update_password: on_create`), donc une relance ne le modifie pas.
- **Droits** : les deux groupes ne reçoivent aucun droit d'administration et n'entrent dans aucun groupe privilégié du domaine. Ils n'ont que les droits par défaut d'un utilisateur du domaine.
- **Journaux Windows** : le journal Système est consulté à chaque passage (5 dernières erreurs, lecture seule). Aucune erreur n'était présente dans les 200 derniers événements au moment du passage.
- **Windows Exporter 0.32.0** : installé sur WIN-DC01, port 9182, collecteurs `cpu,logical_disk,memory,net,os,service,system`. Pare-feu : le port n'est ouvert que pour `192.168.50.0/24`, profil Domaine. Le MSI est téléchargé sur ops01, puis copié sur le serveur.

Ce que ça prouve : les objets Active Directory existent et sont cohérents, l'exporter répond, et les passages sont idempotents (`changed=0` à la relance, pour `windows.yml` et `windows-exporter.yml`).
Ce que ça ne prouve pas : les droits métier (aucun partage ni GPO n'est configuré), ni la connexion depuis un compte de test (mot de passe aléatoire non consigné).

## Qualité et intégration continue

La pipeline `.github/workflows/ci.yml` s'exécute à chaque push et pull request. Elle :
- vérifie le YAML avec `yamllint` (`.yamllint`) ;
- lance `ansible-lint` (`.ansible-lint`) ;
- vérifie la syntaxe des six playbooks.

Les versions sont figées : `requirements-ci.txt` (ansible-core 2.20.1, ansible-lint 26.1.1, yamllint 1.38.0) et `requirements.yml` (ansible.windows 3.3.0, community.windows 3.1.0, microsoft.ad 1.10.0).

Pour reproduire localement, depuis `~/infraready-ansible` : `yamllint .`, puis `ansible-lint`, puis `ansible-playbook -i inventory.ini <playbook> --syntax-check` pour chaque playbook.

Règles de lint ignorées, avec leur justification :
- `command-instead-of-module` : `systemctl is-active` sert à vérifier l'état, en lecture seule.
- `package-latest` : la mise à jour contrôlée de `tmux` demande explicitement la dernière version.
- `fqcn[canonical]` : `ansible.builtin.mount` est disponible dans ansible-core 2.20.
- `run-once[task]` : les plays sont linéaires, et le MSI n'est téléchargé qu'une fois.
- `no-handler` : le journal de maintenance n'est écrit que si des paquets ont changé.

Résultat de la reproduction locale, avec les mêmes versions : `yamllint .` OK, `ansible-lint` OK (code 0), syntaxe des six playbooks OK.
Ce que ça ne prouve pas : la pipeline n'a pas encore tourné sur GitHub, car le dépôt n'est pas encore poussé.

## Résultats constatés

- nginx, second passage : `ok=3 changed=0 failed=0 unreachable=0`.
- nginx, contrôles : `active`, `enabled`, `HTTP/1.1 200 OK` en local, et `HTTP/1.1 200 OK` depuis la WSL sur 192.168.50.20.
- c-linux, troisième passage : `ok=15 changed=0 failed=0 skipped=1`, contrôle final « Controles du bloc C reussis ».
- Historique de c-linux : le premier passage a été interrompu par une erreur du miroir Ubuntu (index `dep11` de taille incohérente, cause externe). Le second passage a mis à jour `tmux` (`changed=1`). Le troisième passage est propre.
- f-supervision, premier passage : tâches réussies, mais contrôle des règles en échec, car Prometheus avait été démarré avant la configuration (entrée 044 du journal). Un redémarrage manuel de Prometheus puis une relance ont corrigé le problème.
- f-supervision, relance : `ok=22 changed=0 failed=0 unreachable=0`, contrôle final « Controles du bloc F reussis ».
- f-supervision, vérification : services actifs et activés au boot ; règles `ServeurInjoignable` et `DisqueRacinePlein` chargées ; `up{job="node"}` = 1 ; Grafana `database: ok` ; ports 3000, 9090 et 9100 en écoute.
- maintenance, premier passage : 44 paquets mis à jour à 10:21 UTC (historique APT), 44 en attente avant, 0 après ; aucun redémarrage requis ; services actifs.
- maintenance, relance : `ok=11 changed=0 failed=0`, contrôle final « Controles du bloc E reussis ».
- windows (AD DS), premier passage : `ok=9 changed=4 failed=0` (unité, groupes, comptes et appartenances créés).
- windows (AD DS), relance après `update_password: on_create` : `ok=9 changed=0 failed=0`, idempotence prouvée.
- windows-exporter : relance `ok=11 changed=0 failed=0 skipped=1`. Le premier passage avait échoué sur le collecteur invalide `cs`, corrigé.
- f-supervision, passage avec la cible Windows : `ok=24 changed=3 failed=0`. Prometheus redémarré par le handler ; cibles `prometheus`, `node` et `windows` à `up = 1`.
- f-supervision, relance : `ok=23 changed=0 failed=0 unreachable=0`, contrôle final « Controles du bloc F reussis ». Idempotence du bloc F prouvée.

## Écarts par rapport au dossier technique

- **Version d'Ubuntu** : 26.04.1 LTS constatée, alors que le dossier technique indique 24.04. La version constatée est retenue.
- **Connexion locale** : `ansible_connection=local` remplace la connexion SSH vers soi-même.
- **Élévation** : les playbooks sont exécutés en root via `sudo ansible-playbook`. Le prompt `-K` n'a pas fonctionné en connexion locale. Piste d'amélioration : rétablir `become` avec une connexion SSH vers ops01, pour n'élever les droits que tâche par tâche.
- **Compte d'administration** : `ops01` avec mot de passe sudo, au lieu d'un compte technique dédié.
- **Disque de test** : simulé par une image fichier montée en boucle.
- **Mise à jour** : limitée à un paquet (`tmux`) sur 45 en attente.
- **Interpréteur Python** : fixé à `/usr/bin/python3` pour éviter la découverte automatique.
- **Redondance** : `become: true` est déclaré dans `nginx.yml` et dans `group_vars/linux.yml`, sans effet néfaste.
- **Avertissements** : lors du montage, Ansible core affiche des avertissements de dépréciation (`module_utils`). Ils n'affectent pas le résultat.
- **Grafana** : le mot de passe par défaut `admin` doit être changé à la première connexion. Le changement reste à confirmer ; aucun mot de passe n'est écrit dans les fichiers.
- **Ports ouverts** : Prometheus (9090), Grafana (3000) et Node Exporter (9100) écoutent sur toutes les interfaces. À restreindre au réseau du lab, à documenter dans `docs/securite.md`.
- **Clé du dépôt Grafana** : téléchargée depuis `apt.grafana.com` sans vérification d'empreinte. À vérifier.
- **Alertes** : aucun Alertmanager, donc aucune notification. Les alertes ne sont visibles que dans Prometheus.
- **Mises à jour automatiques** : `unattended-upgrades` est actif et installe des correctifs hors Ansible (par exemple à 08:24 le 22 octobre). À documenter comme décision de politique de mise à jour.
- **WinRM en HTTP** : port 5985 avec authentification NTLM, sans TLS (5986 fermé). Les messages NTLM sont chiffrés, mais le transport n'est pas TLS. À durcir avec un certificat sur WIN-DC01.
- **Droits des groupes** : aucun droit spécifique n'est attribué aux groupes. À valider avec le formateur si des droits métier sont attendus.
- **Comptes de test** : mot de passe aléatoire, non consigné. Pour une démonstration en soutenance, il faudra le réinitialiser.
- **Compte admin localisé** : `INFRAREADY\Administrateur` (nom en français), et non `Administrator`.
