# Container's layer

La layer d’un container, est la layer read-write créé lorsqu’un container est lancé. C’est la layer dans laquelle tous les changements effectués dans le container sont sauvegardés. Cette layer est supprimée avec le container et ne doit donc pas être utilisée comme un stockage persistant.

## Lancement d'un container

Utilisez la commande suivante pour lancer un shell intéractif dans un container basé sur l’image *ubuntu:18.04*.

```
$ docker container run -ti ubuntu:18.04
```

## Installation d'un package

*figlet* est un package qui prend un texte en entrée et le formatte de façon amusante. Par défaut ce package n’est pas disponible dans l’image ubuntu. Vérifiez le avec la commande suivante:

```
# figlet
```

La commande devrait donner le résultat suivant.

```
bash: figlet: command not found
```

Installez le package *figlet* avec la commande suivante:

```
# apt-get update -y
# apt-get install figlet
```

Vérifiez que le binaire fonctionne

```
# figlet Hola
```

Ce qui devrait donner le résultat suivant

```
_   _       _
| | | | ___ | | __ _
| |_| |/ _ \| |/ _` |
|  _  | (_) | | (_| |
|_| |_|\___/|_|\__,_|

```

Sortez du container.

```
# exit
```

## Lancement d'un nouveau container

Lancez un nouveau container basé sur *ubuntu:18.04*.

```
$ docker container run -ti ubuntu:18.04
```

Vérifiez si le package figlet est présent.

```
# figlet
```

Vous devriez obtenir l’erreur suivante:

```
bash: figlet: command not found
```

Comment expliquez-vous ce résultat ?

Chaque container lancé à partir de l’image ubuntu est différent des autres. Le second container est différent de celui dans lequel figlet a été installé. Chacun correspond à une instance de l’image ubuntu et a sa propre layer, ajoutée au dessus des layers de l’image, et dans laquelle tous les changements effectués dans le container sont sauvegardés.

Sortez du container

```
# exit
```

## Redémarrage du container


Listez les containers (en exécution ou non) sur la machine hôte

```
$ docker container ls -a
```

Depuis cette liste, récuperez l’ID du container dans lequel le package figlet a été installé et redémarrez le avec la commande suivante.

Note: la commande *start* permet de démarrer un container se trouvant dans l'état **Exited**.

```
$ docker container start CONTAINER_ID
```

Lancez un shell intéractif dans ce container en utilisant la commande exec.

```
$ docker container exec -ti CONTAINER_ID bash
```

Vérifez que figlet est présent dans ce container.

```
# figlet Hola
 _   _       _
| | | | ___ | | __ _
| |_| |/ _ \| |/ _` |
|  _  | (_) | | (_| |
|_| |_|\___/|_|\__,_|

```

Vous pouvez maintenant sortir du container.

```
# exit
```

## Nettoyage

Listez les containers (en exécution ou non) sur la machine hôte

```
$ docker container ls -a
```

Pour supprimer tous les containers, nous pouvons utiliser les commandes *rm* et *ls -aq* conjointement. Nous ajoutons l’option -f afin de forcer la suppression des containers encore en exécution. Il faudrait sinon arrêter les containers et les supprimer.

```
$ docker container rm -f $(docker container ls -aq)
```

Tous les containers ont été supprimés, vérifiez le unee nouvelle fois avec la commande suivante:

```
$ docker container ls -a
```
