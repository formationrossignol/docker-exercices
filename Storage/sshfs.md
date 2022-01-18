# Utilisation du driver sshfs

Dans cet exercice, nous allons utiliser le driver *vieux/sshfs* afin de persister des données dans un volume créé sur une machine distante/

## Création de 2 VM Ubuntu

Nous allons utiliser ici l'outils [Multipass](https://multipass.run) pour créer facilement 2 machines virtuelles Ubuntu en local. Une fois Multipass installé, lancez les 2 commands suivantes pour créer:
- la machine *node1* sur laquelle nous installerons Docker dans la suite
- la machine *store* dans laquelle seront persistées les données via le driver *sshfs*

```
$ multipass launch -n node1
$ multipass launch -n store
```

## Répertoire de stockage

Sur la VM *store*, créez le répertoire */tmp/store*, celui-ci sera utilisé pour persister les données via un volume.

```
$ multipass exec store -- mkdir /tmp/store
```

## Clé ssh

Afin de faire en sorte d'avoir un accès sans mot de passe sur la VM *store* depuis *node1*, créez une paire de clés ssh sur *node1* et copiez la clé publique dans le fichier *authorized_keys* de l'utilisateur *ubuntu* sur la VM *store*.

- création des clés ssh sur node1
```
$ multipass exec node1 -- ssh-keygen -t rsa -P '' -f /home/ubuntu/.ssh/id_rsa
```

- copy de la clé publique dans *store*
```
$ KEY=$(multipass exec node1 -- cat /home/ubuntu/.ssh/id_rsa.pub)
$ multipass exec store -- /bin/bash -c "echo $KEY >> /home/ubuntu/.ssh/authorized_keys"
```

## Installation de Docker

Utilisez la commande suivante pour installer Docker sur *node1*

```
$ multipass exec node1 -- /bin/bash -c "curl -fsSL https://get.docker.com | sh -"
$ multipass exec node1 -- sudo usermod -aG docker ubuntu
```

## Installation du plugin sshfs

Lancez la commande suivante afin d'installer le plugin *vieux/sshfs*:

```
$ multipass exec node1 -- docker plugin install --grant-all-permissions vieux/sshfs sshkey.source=/home/ubuntu/.ssh/
```

Cela permettra d'utiliser le driver correspondant pour créer un volume sur une machine distante via ssh.

## Creation d'un volume

Maintenant que le plugin *vieuw/sshfs* est installé, créez un volume en utilisant le driver de même nom:

```
$ IP_STORE=$(multipass info store | grep IP | awk '{print $2}')
$ multipass exec node1 -- docker volume create -d vieux/sshfs -o sshcmd=ubuntu@$IP_STORE:/tmp/store -o IdentityFile=/root/.ssh/id_rsa sshvolume
```

## Utilisation du volume

Vous allez à présent lancer un container sur *node1* en montant le volume défini précédemment.

Utilisez la commande suivante pour lancer un shell sur *node1*

```
$ multipass shell node1
```

Lancez ensuite un container dans lequel le volume *sshvolume* est monté. Faites en sorte que la commande lancée dans le container permette de créer un fichier de test dans le volume en question:

```
ubuntu@node1:~$ docker run -ti -v sshvolume:/data alpine touch /data/TEST_FILE

ubuntu@node1:~$ exit
```

Vérifiez ensuite que le fichier à correctement été créé dans le répertoire */tmp/store* de la VM *store*

```
$ multipass exec store -- ls /tmp/store
```
