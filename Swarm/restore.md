# Objectif

Dans cet exercice nous allons voir comment faire le backup et la restauration d'un cluster Swarm

## Quelques rappels

Comme nous l'avons vu, les nodes de type *Manager* sont les garants de l'état du swarm. Chaque action effectuée dans le swarm (création d'un service, mise à jour d'un service, création d'un network, ....) est sauvegardée dans un fichier de log sur le leader des managers. Ce fichier étant répliqué sur les autres managers.

Sur chaque manager les clés servant à chiffrer ces logs, et d'autres fichiers relatifs à l'état du swarm, sont stockés dans le répertoire */var/lib/docker/swarm*. Ce répertoire est créé lors de l'initialisation du swarm (docker swarm init).

## Création de 2 clusters swarm

Dans cet exercice, nous allons utiliser [Multipass](https://multipass.run). Une fois Multipass installé, lancez la commande suivante afin de créer 2 VMs nommées *swarm1* et *swarm2*:

```
for i in 1 2; do
  multipass launch -n swarm$i
done
```

Installez Docker sur chacune de ces VMs en utilisant la commande suivante:

```
for i in 1 2; do
  multipass exec swarm$i -- /bin/bash -c "curl -sSL https://get.docker.com | sh"
done
```

Créez ensuiten un service sur le premier cluster:

```
$ multipass exec swarm1 -- docker service create --replicas=2 nginx:1.18
```

## Backup du premier cluster (swarm1)

Afin de réaliser le backup d'un swarm, effectuez les opérations suivantes:

- arrêtez le daemon Docker depuis le manager:

```
$ multipass exec swarm1 -- sudo systemctl stop docker
```

- copiez le répertoire /var/lib/docker/swarm à l'extérieur du cluster. Nous utilisons pour cela un répertoire de la machine local que nous montons dans le système de fichier du manager

```
$ mkdir /tmp/backup
$ multipass mount /tmp/backup swarm1:/tmp/backup
$ multipass exec swarm1 -- sudo cp -r /var/lib/docker/swarm /tmp/backup/
```

- redémarrez le daemon Docker avec la commande suivante:

```
$ multipass exec swarm1 -- sudo systemctl start docker
```

## Restauration du backup sur le second cluster (swarm2)

Afin de réaliser la restauration d'un backup, effectuez les opérations suivantes:

- arrêtez le daemon Docker sur le manager

```
$ multipass exec swarm2 -- sudo systemctl stop docker
```

- supprimez le répertoire */var/lib/docker/swarm*

```
$ multipass exec swarm2 -- sudo rm -rf /var/lib/docker/swarm 
```

- copiez le contenu du backup dans */var/lib/docker/swarm*. Nous utilisons pour cela le répertoire local dans lequel le backup a été créé:

```
$ multipass mount /tmp/backup swarm2:/tmp/backup
$ multipass exec swarm2 -- sudo cp -r /tmp/backup/swarm /var/lib/docker/
```

- redémarrez le daemon

```
$ multipass exec swarm2 -- sudo systemctl start docker
```

- forcez la ré-initialisation du swarm avec la commande suivante, celle-ci force la création d'un nouveau cluster à partir de l'état courant (les éléments présents dans le répertoire */var/lib/docker/swarm*)

```
$ multipass exec swarm2 --  docker swarm init --force-new-cluster
```

Vérifiez ensuite que le service contenant 2 replicas de nginx a bien été lancé sur ce nouveau cluster:

```
$ multipass exec swarm2 -- docker service ls
ID             NAME                 MODE         REPLICAS   IMAGE        PORTS
sh2uiyoaml6m   nervous_tereshkova   replicated   2/2        nginx:1.18
```
