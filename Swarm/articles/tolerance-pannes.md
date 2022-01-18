# Elément pour la tolérance aux pannes dans un swarm

## Quelques rappels

Un Swarm est constitué de managers et de workers:
- les managers sont responsables de l'état du cluster
- les workers sont dédiés à faire tourner les containers (tasks) des services

Par défaut un manager est également un worker, la réciproque n'étant pas vraie.

Un seul des managers a le rôle de *leader*, celui-ci consigne l'ensemble des évènements du cluster dans des fichiers de logs (utilisés par l'algorithme de consensus distribué Raft). Ces logs sont ensuite répliqués sur les autres managers.

Si un Swarm ne compte qu'un manager, et que celui-ci devient indisponible, les services déployés sur les workers continuerons à tourner mais aucune action d'administration ne pourra plus être effectuée (déploiement de nouveaux services, mise à jour de service existant, ...). C'est la raison pour laquelle il est important, dans un environnement de production, de déployer plusieurs managers.

De plus, chaque mise à jour de l'état du swarm nécessite qu'un quorum, c'est à dire la majorité des membres, soit disponible.

Le tableau suivant indique le quorum nécessaire en fonction du nombre de managers. Le nombre de pannes supportées est également spécifié.

Nombre de managers | Quorum | Nombre de tolérances aux pannes
------------------ | ------ | -------------------------------
1 | 1 | 0
2 | 2 | 0
3 | 2 | 1
4 | 3 | 1
5 | 3 | 2
6 | 4 | 2

Exemple: si un Swarm a 3 managers et donc un quorum égal à 2, il ne peut supporter la défaillance que d'un seul manager. Si 2 managers sont défaillant en même temps il n'est plus possible d'obtenir le quorum dans les décisions de vote et de mise à jour de l'état du cluster.

> Il est recommandé d'utiliser un nombre impair de manager. La tolérance aux pannes avec un nombre pair de managers est la même que pour le nombre impair qui précéde. En d'autres mots: utiliser 4 managers au lieu de 3 ne changera rien.

Pour assurer une redondance géographique et éviter de dépendre d'un seul prestataire d'hébergement (Amazon AWS, Google GCP, Microsoft Azure, DigitalOcean, Packet, ...), il est possible de déployer un Swarm sur plusieurs datacenters.

Le tableau ci-dessous prend l'exemple d'un Swarm déployé dans 3 datacenters et indique la répartition optimale des managers en fonction du nombre de managers.

Nombre de managers | Répartition des managers
------------------ | ------------------------
3 | 1-1-1
5 | 2-2-1
7 | 3-2-2
9 | 3-3-3

Exemple: en considérant un Swarm contenant 7 managers, la répartition optimale est d'avoir 3 managers dans le premier datacenter, et 2 managers dans les 2 autres.
