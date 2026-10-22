# InfraReady : Ansible, installation de nginx sur ops01

## Objet

Premier playbook du bloc C : installer nginx, vérifier qu'il tourne et qu'il démarre au boot, tester l'idempotence. Les autres volets (comptes, stockage, mises à jour, journaux) viendront plus tard.

## Prérequis

- Machine cible et contrôleur : ops01, Ubuntu Server 26.04.1 LTS (le dossier technique indique 24.04, voir Écarts).
- Ansible core 2.20.1, Python 3.14 (`/usr/bin/python3`).
- Utilisateur `ops01` avec sudo. Le mot de passe sudo est saisi à l'exécution, jamais écrit dans un fichier.

## Structure

```
inventory.ini          cible : groupe linux, ops01 en connexion locale
group_vars/linux.yml   variables du groupe (sudo, interpréteur Python, paquet et service web)
nginx.yml              playbook : installe et active nginx
.gitignore             exclut clés, *.vault, secrets et .env
README.md              ce fichier
```

Le journal des changements (`resume.txt`) est volontairement hors du dépôt, à la racine du système (`/resume.txt`).

## Commandes

Depuis `~/infraready-ansible` :

```
# Vérifier la syntaxe, sans rien modifier
ansible-playbook -i inventory.ini nginx.yml --syntax-check

# Vérifier qu'Ansible atteint la cible
ansible -i inventory.ini linux -m ping

# Exécuter le playbook (mot de passe sudo demandé)
sudo ansible-playbook -i inventory.ini nginx.yml
```

Relancer le playbook immédiatement après doit donner `changed=0` : c'est le test d'idempotence.

Pourquoi `sudo ansible-playbook` et non `ansible-playbook -K` : avec `-K`, Ansible a expiré sur le prompt sudo (« Timed out waiting for become success or become password prompt »). La commande avec `sudo` contourne ce point, au prix d'un playbook exécuté entièrement en root (voir Écarts).

## Cible

Le groupe `linux` ne contient que `ops01`. Ansible agit en local : aucune connexion SSH n'est utilisée.

## Contrôles de résultat

```
systemctl is-active nginx     # attendu : active
systemctl is-enabled nginx    # attendu : enabled
curl -I http://localhost      # attendu : HTTP/1.1 200 OK
```

Ce que ces contrôles prouvent : le service tourne, démarre au boot et répond en HTTP en local.
Ce qu'ils ne prouvent pas : l'accessibilité depuis un autre poste du réseau (non testé ici) ni le contenu servi.

## Résultats constatés

- Second passage du playbook : `ok=3 changed=0 failed=0 unreachable=0`, donc idempotent.
- Contrôles : `active`, `enabled`, `HTTP/1.1 200 OK` sur localhost.

## Écarts par rapport au dossier technique

- Version d'Ubuntu : 26.04.1 LTS constatée, alors que le dossier technique indique 24.04. La version constatée est retenue.
- Ansible tourne sur la machine qu'il configure : `ansible_connection=local` remplace la connexion SSH vers soi-même.
- Playbook exécuté en root via `sudo ansible-playbook` : le prompt `-K` n'a pas fonctionné en connexion locale. Piste d'amélioration : rétablir `become` avec une connexion SSH vers ops01, pour n'élever les droits que par tâche.
- Compte d'administration : `ops01` avec mot de passe sudo, au lieu d'un compte technique dédié.
- Interpréteur Python fixé à `/usr/bin/python3` pour éviter la découverte automatique.
- `become: true` est déclaré dans `nginx.yml` et dans `group_vars/linux.yml`. C'est redondant, sans effet néfaste.
