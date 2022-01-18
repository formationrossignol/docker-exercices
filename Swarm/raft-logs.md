# Raft logs

Dans cet exercice, nous allons voir d'un peu plus près comment l'état du cluster est sauvegardé dans les logs générés par l’algorithme Raft. Ces logs sont crées sur le Leader des managers et répliqués sur chaque manager.

# Swarm architecture

Le schéma suivant met en évidence un "Internal Distributed Data Store" utilisé pour la gestion du cluster. Il s'agit en fait des logs qui sont répliqués entre les différents managers d'un cluster Swarm.

![Architecture](./images/swarm1.png)

# Création d’un swarm

> si vous avez déjà créé un cluster swarm, vous pouvez passer directement à la section suivante.

Une solution simple est d'utiliser [Multipass](https://multipass.run) pour créer 2 machines virtuelles et mettre en place un cluster Swarm sur celles-ci.

:fire: nous utilisons Multipass dans cet exercice mais vous pouvez bien sur utiliser un autre solution afin de créer ces VMs.

## Création des VMs

Une fois Multipass installé, utilisez la commande suivante pour créer 2 VMs:

```
for i in 1 2; do
  multipass launch -n node$i
done
```

Vérifier ensuite que ces VMs sont dans l'état *Running*:

```
$ multipass list 
Name                    State             IPv4             Image
node1                   Running           192.168.64.11    Ubuntu 20.04 LTS
node2                   Running           192.168.64.12    Ubuntu 20.04 LTS
```

Installez ensuite Docker sur chacune de ces VMs:

```
for i in 1 2; do
  multipass exec node$i -- /bin/bash -c "curl -sSL https://get.docker.com | sh"
done
```

Vérifiez ensuite que l'installation s'est déroulée correctement:

```
for i in 1 2; do
  multipass exec node$i -- sudo docker version
done
```

Avant de passer à l'étape suivante, récupérez les adresses IP de chacune de ces VMs, vous en aurez besoin dans la suite:

```
IP1=$(multipass info node1 | grep IPv4 | cut -d':' -f2 )
IP2=$(multipass info node2 | grep IPv4 | cut -d':' -f2 )
```

## Initialisation du swarm

Depuis un hôte Docker qui n'est pas en mode Swarm, le répertoire */var/lib/docker/swarm* est vide comme le confirme la commande suivante:

```
$ multipass exec node1 -- ls /var/lib/docker/swarm
```

Depuis node1, lancez la commande suivante afin d'initialiser le Swarm:

```
$ multipass exec node1 -- /bin/bash -c "sudo docker swarm init --advertise-addr $IP1"
```

Vous obtiendrez un résultat proche de celui ci-dessous:

```
Swarm initialized: current node (otqo5vn8i3n2tp03jkvgb80to) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-40uyzsm8bu99sp5vdavyknucg5snvkxfst8nc8h2r48m0pa6cw-7yvb02jk4ja7dezb6bxptvv7h 192.168.64.11:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Depuis *node2* lancez la commande de *join* telle qu'elle est précisée par la commande précédente.

```
$ multipass exec node2 -- /bin/bash -c "sudo docker swarm join --token SWMTKN-1-40uyzsm8bu99sp5vdavyknucg5snvkxfst8nc8h2r48m0pa6cw-7yvb02jk4ja7dezb6bxptvv7h 192.168.64.11:2377
```

Listez ensuite les nodes de votre Swarm, vous devriez obtenir un résultat proche de celui ci-dessous:

```
$ multipass exec node1 -- sudo docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
otqo5vn8i3n2tp03jkvgb80to *   node1      Ready     Active         Leader           20.10.5
ssr9pn2x2kbb1otcq0mn1yxac     node2      Ready     Active                          20.10.5
```

A partir du moment ou le daemon Docker est en mode Swarm (suite à la commande d’initialisation précédente), plusieurs éléments sont présents dans le répertoire swarm.

```
$ multipass exec node1 -- sudo find /var/lib/docker/swarm
/var/lib/docker/swarm
/var/lib/docker/swarm/docker-state.json
/var/lib/docker/swarm/raft
/var/lib/docker/swarm/raft/wal-v3-encrypted
/var/lib/docker/swarm/raft/wal-v3-encrypted/0.tmp
/var/lib/docker/swarm/raft/wal-v3-encrypted/0000000000000000-0000000000000000.wal
/var/lib/docker/swarm/raft/snap-v3-encrypted
/var/lib/docker/swarm/state.json
/var/lib/docker/swarm/worker
/var/lib/docker/swarm/worker/tasks.db
/var/lib/docker/swarm/certificates
/var/lib/docker/swarm/certificates/swarm-node.crt
/var/lib/docker/swarm/certificates/swarm-root-ca.crt
/var/lib/docker/swarm/certificates/swarm-node.key
```

Les logs sont situés dans le répertoire raft, les clés d’encryption dans le répertoire certificates.

# Création d’un secret

Lancez un shell dans la VM node1:

```
$ multipass shell node1
```

Créez ensuite un secret avec la commande suivante. Celle-ci permet de créer un secret nommé passwd et contenant une chaine de caractère.

```
ubuntu@node1:~$ echo 'A2e5bc21' | docker secret create passwd -
```

Si nous listons les secrets existants sur notre swarm, seul le secret précédemment créé apparait, son contenu n’est plus visible.

```
ubuntu@node1:~$ docker secret ls
ID                          NAME      DRIVER    CREATED         UPDATED
mzxhj2130dvjrz4oxv1zo0tl2   passwd              8 seconds ago   8 seconds ago
```

Nous verrons par la suite que ce secret est en clair dans les logs de Raft.

## A propos des logs de Raft

La version 1.13 de la plateforme Docker a introduit la gestion des secrets dans le contexte d’un swarm. Ce sont des information sensibles, par exemple des identifiants de connexion à des services tiers. Les secrets sont stockées en clair dans les logs utilisés par l’implémentation de l’algorithme Raft et c’est notamment pour cette raison que logs sont cryptés, afin d’assurer la confidentialité de ces informations.

Les secrets sont généralement créés par les Ops lors du lancement de l’application puis fournis au service qui en ont besoin. Ils seront alors accessibles, dans les containers du service, depuis un système de fichiers temporaire sous */run/secrets/NOM_DU_SECRET*.

## Décryptage des logs

Nous allons utiliser ici l’utilitaire swarm-rafttool, un binaire qui se trouve dans la librairie SwarmKit utilisée par Docker pour la gestion des clusters Swarm.

### Swarm Rafttool

Afin d'éviter l'installation de cet utilitaire, nous le lançons directement depuis un container depuis un shell sur node1:

```
ubuntu@node1:~$ docker run --rm -ti --entrypoint='' \
  -v /var/lib/docker/swarm/:/var/lib/docker/swarm:ro \
  -v $(pwd):/tmp/ lucj/swarm-rafttool:1.0 sh
```

Nous pouvons alors voir les différentes options possibles:

```
/go # swarm-rafttool
Tool to translate and decrypt the raft logs of a swarm manager

Usage:
  swarm-rafttool [command]

Available Commands:
  decrypt       Decrypt a swarm manager's raft logs to an optional directory
  dump-wal      Display entries from the Raft log
  dump-snapshot Display entries from the latest Raft snapshot
  dump-object   Display an object from the Raft snapshot/WAL

Flags:
  -h, --help                help for swarm-rafttool
  -d, --state-dir string    State directory (default "/var/lib/swarmd")
      --unlock-key string   Unlock key, if raft logs are encrypted

Use "swarm-rafttool [command] --help" for more information about a command.
```

Dans la suite, nous utiliserons la command dump-wal afin de décrypter et visualiser les entrées du fichier de logs.

### Décryptage

Afin de décrypter le fichier de log, nous créons un script shell qui va tout d’abord copier le fichier puis lancer le binaire swarm-rafttool sur cette copie. L’étape de copie est nécessaire car swarm-rafttool ne permet pas de décrypter les logs en cours d’usage.

Créer un fichier dump.sh dans le container lancé précédemment:

```
/go # cat<<'EOF' | tee dump.sh
d=$(date "+%Y%m%dT%H%M%S")
SWARM_DIR=/var/lib/docker/swarm
WORK_DIR=/tmp
DUMP_FILE=$WORK_DIR/dump-$d
STATE_DIR=$WORK_DIR/swarm-$d
cp -r $SWARM_DIR $STATE_DIR
$GOPATH/bin/swarm-rafttool dump-wal --state-dir $STATE_DIR > $DUMP_FILE
echo $DUMP_FILE
EOF
```

Lancez ce script et observer le contenu des logs.

```
/go # chmod +x ./dump.sh
/go # ./dump.sh | xargs cat
```

La sortie est relativement verbeuse, et peux être décomposée en plusieurs *Entry*. Je vous invite à examiner les premières qui sont relatives à la mise en place du Swarm..

La dernière $Entry* (ci-dessous) concerne la création du secret passwd.

```
Entry Index=17, Term=2, Type=EntryNormal:
id: 132335301136655
action: <
  action: STORE_ACTION_CREATE
  secret: <
    id: "mzxhj2130dvjrz4oxv1zo0tl2"
    meta: <
      version: <
        index: 16
      >
      created_at: <
        seconds: 1616446789
        nanos: 117813739
      >
      updated_at: <
        seconds: 1616446789
        nanos: 117813739
      >
    >
    spec: <
      annotations: <
        name: "passwd"
      >
      data: "A2e5bc21\n"
    >
  >
>
```

Comme nous pouvons le voir, le contenu du secret est en clair dans le log. L’encryption des logs est donc obligatoire pour préserver la sécurité de ces informations sensibles.

Sortez ensuite du container

```
/go # exit
```

## Autolock

Si un manager est compromis, les logs cryptés et les clés d’encryption sont récupérables. Il est alors facile pour un hacker de décrypter les logs et d’avoir ainsi accès aux données sensibles, comme nous venons de le faire. Pour empêcher cela, un Swarm peut être locké. Une clé d’encryption est alors générée et utilisée pour encrypter les clés publique / privée (celles servant à encrypter / décrypter les logs).

Cette nouvelle clé, appelée Unlock key, doit être sauvegardée offline et fournie manuellement au daemon Docker après un restart.

Nous visualisons le contenu de la clé située dans le sous répertoire certificates.

```
ubuntu@node1:~$ sudo cat /var/lib/docker/swarm/certificates/swarm-node.key
-----BEGIN PRIVATE KEY-----
kek-version: 0
raft-dek: EiD7edBc+9jsI0lJEH63AMCPsOp7jMargnFZOB31PDmopg==

MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQga5k0N//QHIsAdsOd
r0VMCYUBOj9j5lX4zYYsvwpqFluhRANCAAQi0wveBqEjA7D+3GxBq85+gQMBFxF/
REgfOoHpKUvlS2Hn/jn5CeqHKjM9gu4ethvbaKUDXJWdzSvOtjRn9RsF
-----END PRIVATE KEY-----
```

La commande suivante met à jour le Swarm et active la fonctionnalité d’Autolock.

Note: il est également possible d’activer l’Autolock lors de la création du Swarm.

```
$ docker swarm update --autolock=true
Swarm updated.
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-/OV8LnYcw7QGhnwGAs4CIjM0v1I39biZVwSM+2E4x6U

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.
```

Si nous observons une nouvelle fois le contenu de la clé, nous pouvons voir qu’elle a été encryptée.

```
ubuntu@node1:~$ sudo cat /var/lib/docker/swarm/certificates/swarm-node.key
-----BEGIN ENCRYPTED PRIVATE KEY-----
kek-version: 17
raft-dek: CAESMINEe3+FIiQxS0MnippCDXEl+deTJF0OzE1yHQAuYTvMO1BvDj+Rp0EHPy+MfEITPxoYk2hJlimLi8TAa+J/SuNggLT99wjm3N/s

MIHeMEkGCSqGSIb3DQEFDTA8MBsGCSqGSIb3DQEFDDAOBAj6szwVT23QvwICCAAw
HQYJYIZIAWUDBAEqBBAMi04962zn1hHQBhDHht9cBIGQVERJNp2ol2yjkSJo1Si2
RpPZ5vp4Ukei5YdLD+O7d/+FjeK6WCvMiszNjJ2ZwBKIpCIWYKWWReVDTrfkzl+Y
RM37mFeBr6mq/PAlNRRN20vR77Zgx74fHejFlA8yMKr0EkVVWlQ5YEDdxq6DsFNx
6mEGEVeopHgMMf/fdHC1HSxydZmrj2MTnvlbUU6fOXdh
-----END ENCRYPTED PRIVATE KEY-----
```

### Redémarage du daemon

Reperez le processus dockerd:

```
$ sudo systemctl restart docker
```

Une fois le daemon redémarré, il ne vous sera pas possible d'interagir avec celui-ci car le Swarm est vérouillé.

```
ubuntu@node1:~$ sudo docker node ls
Error response from daemon: Swarm is encrypted and needs to be unlocked before it can be used. Please use "docker swarm unlock" to unlock it.
```

Nous déverouillons alors le manager en lui précisant le token récupéré précédement:

```
$ docker swarm unlock
Please enter unlock key:
```

Vérifez ensuite que vous pouvez bien lister les nodes:

```
ubuntu@node1:~$ sudo docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
otqo5vn8i3n2tp03jkvgb80to *   node1      Ready     Active         Leader           20.10.5
ssr9pn2x2kbb1otcq0mn1yxac     node2      Unknown   Active                          20.10.5
```

## En résumé

Nous avons donné ici un aperçu des logs générés par l’algorithme de consensus Raft. Nous l’avons illustré en nous basant sur un Swarm simplement composé d’un seul manager. Vous trouverez des détails supplémentaires dans cet [article](https://medium.com/lucjuggery/raft-logs-on-swarm-mode-1351eff1e690).
