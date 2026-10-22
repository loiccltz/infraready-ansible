# Rapport d'incident : indisponibilité de WIN-DC01

## Résumé

Le 22 octobre 2026, le contrôleur de domaine WIN-DC01 (192.168.50.10) s'est arrêté de façon non prévue vers 13:42 UTC. Il est resté injoignable pendant environ 5,5 minutes, selon Prometheus, jusqu'à son redémarrage manuel. La supervision a détecté la panne grâce à la cible `windows` et à l'alerte `ServeurInjoignable`. Après le redémarrage, WinRM et l'exporter Windows ont repris sans intervention d'Ansible. Les journaux Windows montrent d'autres arrêts non prévus, dont un coïncide avec un redémarrage d'ops01 : la cause probable se situe au niveau de l'hôte de virtualisation, à confirmer.

## Chronologie (UTC)

Les événements Windows sont convertis depuis l'heure locale (UTC+2).

| Heure | Événement | Source |
|---|---|---|
| 13:42:49 | Arrêt non prévu de WIN-DC01 (événement 6008) | Journal Système Windows |
| 13:42:54 | Dernière mesure à `up = 1` de la cible `windows` | Prometheus |
| 13:43:47 à 13:43:54 | Première mesure à `0` (l'heure exacte dépend de l'échantillonnage de la requête) | Prometheus |
| Environ une minute après | Alerte `ServeurInjoignable` en état `firing` (règle `for: 1m`) | Prometheus |
| 13:45:57 | Tentative d'arrêt contrôlé de `windows_exporter` : `UNREACHABLE`, aucune modification | Ansible |
| 13:47:59 | Redémarrage de WIN-DC01 sans arrêt propre (événement 41), après intervention manuelle | Journal Système Windows |
| 13:48:10 | Démarrage du journal d'événements, la machine est revenue | Journal Système Windows |
| 13:49:17 | Premier retour à `up = 1` | Prometheus |
| Après le retour | Alerte résolue ; WinRM authentifié (`pong`) ; `windows_exporter` en état `running`, démarrage automatique | Prometheus, Ansible |

## Détection

La cible `windows` est passée à `0`. La règle `ServeurInjoignable` (`up{job=~"node|windows"} == 0` pendant 1 minute) a déclenché l'alerte. Il n'existe pas de notification : l'alerte n'est visible que dans l'interface de Prometheus.

## Diagnostic

1. **Prometheus** : la cible `windows` est à `0`, l'alerte est en `firing`, les cibles `prometheus` et `node` restent à `1`. La panne est donc ciblée sur la machine Windows.
2. **Couche réseau** : aucune réponse au ping depuis la WSL et depuis ops01. Les ports 5985 (WinRM), 22 (SSH) et 9182 (exporter) sont fermés. Depuis ops01, la connexion renvoie « No route to host ». La résolution ARP de 192.168.50.10 sur `enp0s8` est en `FAILED` : aucune machine ne répond sur le lien local.
3. **Journaux Windows** : l'événement 6008 indique un arrêt non prévu à 13:42:49 UTC, et l'événement 41 indique un redémarrage sans arrêt propre à 13:47:59 UTC. La machine ne s'est donc pas arrêtée proprement.
4. **Conclusion** : WIN-DC01 s'est arrêté brutalement, puis a été redémarré à la main. Le service WinRM n'est pas en cause.

## Action corrective

- Redémarrage de la VM WIN-DC01 par l'utilisateur, dans l'hyperviseur (action hors Ansible).
- Vérifications après le redémarrage : ping, WinRM (`win_ping` réussi), service `windows_exporter` en état `running` et démarrage automatique, cible `up = 1`, alerte résolue.

## Cause

Arrêt brutal de la VM WIN-DC01 à 13:42:49 UTC, qui ne répondait plus, puis redémarrage forcé. La raison de cet arrêt n'est pas identifiée. Pistes :
- un plantage ou un blocage de la VM ;
- un problème au niveau de l'hôte de virtualisation : alimentation, ressources, ou redémarrage de l'hôte.

Les journaux de l'hyperviseur permettront de trancher.

## Historique des arrêts non prévus

| Heure (UTC) | Machine | Événement | Source |
|---|---|---|---|
| 20 octobre, 14:56:53 | WIN-DC01 | Arrêt non prévu (6008) | Journal Système Windows |
| 22 octobre, 08:22:19 | WIN-DC01 | Redémarrage sans arrêt propre (41) | Journal Système Windows |
| 22 octobre, 08:22:11 | ops01 | Redémarrage inattendu (journald signale un fichier de journal corrompu après un arrêt non propre, à 08:25) | Journaux ops01 |
| 22 octobre, 13:42:49 | WIN-DC01 | Arrêt non prévu (6008) : cet incident | Journal Système Windows |

Deux machines ont redémarré à quelques secondes d'intervalle le 22 octobre à 08:22 UTC. Cela suggère un événement au niveau de l'hôte de virtualisation.

## Impact

- **Supervision** : cible `windows` absente pendant environ 5,5 minutes, selon Prometheus.
- **Domaine** : l'impact sur l'authentification et le DNS du contrôleur n'a pas été mesuré dans cette session.
- **Données** : aucune perte constatée dans cette session.

## Action tentée sans effet

Un arrêt contrôlé de `windows_exporter` a été tenté à 13:45:57, pour simuler un incident. Il a échoué (`UNREACHABLE`) : la machine était déjà hors ligne. Aucune modification n'a été faite.

## Prévention (propositions)

- Vérifier les journaux de l'hyperviseur, pour établir si l'arrêt vient de la VM ou de l'hôte.
- Configurer le démarrage automatique des VM avec l'hôte, et vérifier l'alimentation et les ressources de l'hôte.
- Ajouter un Alertmanager pour envoyer une notification. Aujourd'hui, l'alerte n'est visible que dans Prometheus.
- Surveiller l'hôte de virtualisation, ou au moins la disponibilité des VM. Aujourd'hui, WIN-DC01 est un point de défaillance unique.
- Ajouter un second contrôleur de domaine, si le sujet le permet.

## Enseignements

Prometheus a détecté la panne en quelques minutes. Le diagnostic par couches (Prometheus, réseau, ARP, journaux Windows) a établi que la machine s'était arrêtée brutalement, sans toucher à son service WinRM. Les journaux Windows ont aussi révélé des arrêts non prévus antérieurs, qui n'avaient pas été repérés.
