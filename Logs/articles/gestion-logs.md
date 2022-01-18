Le daemon Docker utilise des drivers de logging pour collecter les logs des containers et services qu'il fait tourner. La commande docker info  permet de connaitre les drivers supportés ainsi que celui qui est actuellement utilisé par le daemon.

```
$ docker info | grep Log
Logging Driver: json-file
 Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
```

Dans le cas présent, le driver json-file est utilisé. Nous reviendrons sur les différents plugins supportés un peu plus bas.

## Le driver json-file

C'est celui qui est utilisé par défaut sur une installation de Docker CE. Cela signifie que pour chaque container un fichier de log au format json est créé dans le repertoire d'installation de Docker (usuellement /var/lib/docker).

Voyons cela sur un exemple et considérons un container qui envoie des requêtes ping en continu. Nous lançons ce container avec la commande suivante:

```
$ docker container run -d alpine ping 8.8.8.8
04446c13b167c70ecad87385687693aa41cb68eabbf6419b76ef5156722f0d70
```

A partir de l'identifiant retourné (que nous appèlerons CONTAINER_ID), nous pouvons accéder au fichier de log du container. Comme nous le voyons ci-dessous, une entrée est ajoutée à chaque requête ping envoyée par le container.

```
$ cat /var/lib/docker/containers/CONTAINER_ID/CONTAINER_ID-json.log
{"log":"PING 8.8.8.8 (8.8.8.8): 56 data bytes\n","stream":"stdout","time":"2017-10-02T06:29:01.228472527Z"}
{"log":"64 bytes from 8.8.8.8: seq=0 ttl=60 time=1.816 ms\n","stream":"stdout","time":"2017-10-02T06:29:01.230998953Z"}
{"log":"64 bytes from 8.8.8.8: seq=1 ttl=60 time=1.655 ms\n","stream":"stdout","time":"2017-10-02T06:29:02.230258483Z"}
{"log":"64 bytes from 8.8.8.8: seq=2 ttl=60 time=1.412 ms\n","stream":"stdout","time":"2017-10-02T06:29:03.23101734Z"}
{"log":"64 bytes from 8.8.8.8: seq=3 ttl=60 time=1.408 ms\n","stream":"stdout","time":"2017-10-02T06:29:04.230977496Z"}
{"log":"64 bytes from 8.8.8.8: seq=4 ttl=60 time=1.459 ms\n","stream":"stdout","time":"2017-10-02T06:29:05.23160492Z"}
{"log":"64 bytes from 8.8.8.8: seq=5 ttl=60 time=1.450 ms\n","stream":"stdout","time":"2017-10-02T06:29:06.231869715Z"}
{"log":"64 bytes from 8.8.8.8: seq=6 ttl=60 time=1.384 ms\n","stream":"stdout","time":"2017-10-02T06:29:07.232303965Z"}
...
```

## Changement du driver de logging

Le tableau suivant liste les différents drivers disponibles et la particularité de chacun d'entre eux.

Driver | Description     
------ | -----------
none      | Aucun log n'est généré     
json-file | Les logs sont formatés en json, c'est le driver utilisé par défaut
syslog    | Les logs sont envoyés à syslog, cela nécessite que le daemon syslog tourne sur la machine hôte     
journald  | Les logs sont envoyés à journald, cela nécessite que le daemon journald tourne sur la machine hôte     
gelf      | Les logs sont envoyés à un endpoint GELF (Graylog Extended Log Format), comme Graylog ou Logstash (bien connu dans le cadre d'une stack ELK)     
fluentd   | Les logs sont envoyés à fluend, cela nécessite que le daemon fluentd tourne sur la machine hôte     
awslogs   | Les logs sont envoyés à Amazon CloudWatch     
splunk    | Les logs sont envoyés à splunk     
etwlogs   | Gestionnaire de logs pour les plateformes Windows     
gcplogs   | Les logs sont envoyés à Google Cloud Platform (GCP)

Pour utiliser un autre driver, parmi les plugins suportés, il suffit de le configurer dans le fichier /etc/docker/daemon.json

Par exemple, si nous souhaitons utiliser le driver syslog, nous ajoutons l'entrée "log-driver": "syslog"  dans /etc/docker/daemon.json et redémarrons le daemon (il faudra au préalable s'assurer que le daemon syslog tourne sur la machine hôte).

```
{
  ...
  "log-driver": "syslog"
}
```

Note: pour des containers très verbeux les fichiers de log crées en local peuvent rapidement prendre beaucoup de place. Une approche intéressante est d'envoyer l'ensemble des logs dans un service tiers qui pourra indexer ces logs et également mettre à disposition des outils de filtrage et de visualisation, par exemple une stack ELK (Elasticsearch / Logstash, Kibana).

Il est possible de configurer le driver utilisé en fournissant des options supplémentaires. Par exemple, si nous souhaitons envoyer les logs vers un serveur syslog distant tournant sur la machine logs_server sur le port 1234, nous le précisons de la façon suivante:  

```
{
  ...
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "udp://logs_server:1234"
  }
}
```

Chaque driver a sa propre liste d'options disponibles, chacune sera spécifiée sous la forme key=value sous la clé log-opts.

## Driver de logging pour un container

Nous avons vu ci-dessus comment utiliser un driver de logging au niveau du daemon Docker. Ce driver est utilisé pour l'ensemble des containers géré par le daemon. Il est cependant possible de préciser le driver à utiliser au niveau du container, si par exemple nous ne souhaitons pas utiliser le driver par défaut.

Supposons que le daemon utilise le driver json-file. Si nous souhaitons que les logs d'un container soit gérés par un daemon syslog distant, nous pourrons démarrer le container en spécifiant les options suivantes:

```
docker container run --log-driver syslog --log-opt syslog-address=udp://log_server:1234 ...
```
