---
id: upgrade-double-run-from-21.04
title: Montée de version de Centreon 22.04 depuis Centreon 21.04 en double-run
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';


Ce document décrit l'ensemble des étapes pour réaliser la migration d'une plateforme existante Centreon en 21.04 vers une nouvelle plateforme(Server, OS, etc.) en 22.04 en utilisant la méthode du double run.

L'architecture ici migré se compose d'un serveur Central avec base de données déportée et d'un Poller.
Le fonctionnement est la même pour la base de donnée sur le Central. Les opérations des bases sera entre les Centraux.

## Prérequis

### Version de l’OS

Tous les serveurs de la nouvelle plateforme doivent être fraichement installés avec les paquets ou à l'aide de l'ISO Centreon.
Un accès aux repos Centreon est nécéssaire surtout pour les différents modules.

### Comptes et droits

Les accès root sur toutes les machines sont nécéssaire ainsi que l'accès à l'ancienne et nouvelle base de données.

### Espace disque

Prendre en compte l'espace disque nécéssaire au stockage temporaire des dumps, ainsi que l'accroissement de la taille de la base lié à l'éventuel changement de moteur (MyIsam vers InnoDB, environs 1,5 fois)

### Conventions

Nous utiliserons les règles de nommage suivante :

- `@OLD-CENTRAL@` signifie que la commande est exécuté sur l’ancien serveur central
- `@NEW-CENTRAL@` signifie que la commande est exécuté sur le nouveau serveur central

- `@OLD-DB@` signifie que la commande est exécutée sur l'ancien serveur de base de données
- `@NEW-DB@` signifie que la commande est exécutée sur le nouveau serveur de base de données

- `@NEW-DB-MariaDB@` signifie que la commande est exécutée sur la base de données (une fois connecté)
- `@OLD_DATABASE_IP@` signifie qu'il s'agit de l'adresse IP de l'ancienne base de données

- `@OLD-POLLER@` signifie que la commande est exécutée sur l'ancien poller
- `@NEW-POLLER@` signifie que la commande est exécutée sur le nouveau poller

## Installation de la nouvelle plateforme

Réaliser l'installation complète de la nouvelle plateforme Centreon (central + bdd dans notre cas) sur les nouveaux serveurs en suivant la [procédure suivante](../installation/introduction.md).

## Sauvegarde avant migration

Sauvegarder les bases vierges.

- **@NEW-DB@** :

```bash
mysqldump -u root -p <rootPassword> centreon > db_centreon_fresh_install_`date +%Y%m%d-%H%M`.sql
mysqldump -u root -p <rootPassword> centreon_storage > db_centreon_storage_fresh_install_`date +%Y%m%d-%H%M`.sql
```

## Récupération des données

Récupérer les données de l’ancienne plate-forme vers la nouvelle en ignorant les tables volumineuses data_bin et logs que nous récupérerons dans une autre étape afin de minimiser l’interruption de service.

- **@NEW-DB@** :

```bash
ssh -C root@'@OLD_DATABASE_IP@' "mysqldump -u root -p <rootPassword> centreon" > old_db_centreon_`date +%Y%m%d%H%M`.sql
ssh -C root@'@OLD_DATABASE_IP@' "mysqldump -u root -p <rootPassword> centreon_storage --ignore-table=centreon_storage.data_bin --ignore-table=centreon_storage.logs" > old_db_centreon_storage_`date +%Y%m%d%H%M`.sql
```

## Import des données

Arrêter tous les services sur le nouveau serveur Central.

- **@NEW-CENTRAL@** :

```bash
systemctl stop centengine cbd gorgoned crond
```

Se connecter à la nouvelle base de donnée @NEW-DB-MariaDB@, supprimer les bases existantes puis les recréer :

- **@NEW-DB-MariaDB@** :

```sql
DROP DATABASE centreon;
DROP DATABASE centreon_storage;
CREATE DATABASE centreon;
CREATE DATABASE centreon_storage
```

Injecter les données du dump précédent :

- **@NEW-DB@** :

```sql
mysql -u root -p <rootPassword> centreon < old_db_centreon_<AAMMDDHHMM>.sql
mysql -u root -p <rootPassword> centreon_storage < old_db_centreon_storage_<AAMMDDHHMM>.sql
```

Copier les fichier RRD depuis l’ancien serveur central vers le nouveau. Cette action peut être parallélisée avec l’insertion des dump précédents. En fonction du nombre de métriques compter jusqu’à 1,5 h de synchronisation.

- **@OLD-CENTRAL@** :

```bash
rsync -av /var/lib/centreon/metrics/ root@'@NEW_CENTRAL_IP@':/var/lib/centreon/metrics/
rsync -av /var/lib/centreon/status/ root@'@NEW_CENTRAL_IP@':/var/lib/centreon/status/
```

Créer les tables exclue du dump de la base **centreon_storage** en utilisant les schéma ci dessous :

- **@NEW-DB-MariaDB@** :

```sql
CREATE TABLE `data_bin` (
  `id_metric` int(11) DEFAULT NULL,
  `ctime` int(11) DEFAULT NULL,
  `value` float DEFAULT NULL,
  `status` enum('0','1','2','3','4') DEFAULT NULL,
  KEY `index_metric` (`id_metric`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ;
```

Creation de la table logs:

- **@NEW-DB-MariaDB@** :

```sql
CREATE TABLE `logs` (
  `log_id` int(11) NOT NULL AUTO_INCREMENT,
  `ctime` int(11) DEFAULT NULL,
  `host_id` int(11) DEFAULT NULL,
  `host_name` varchar(255) DEFAULT NULL,
  `instance_name` varchar(255) NOT NULL,
  `issue_id` int(11) DEFAULT NULL,
  `msg_type` tinyint(4) DEFAULT NULL,
  `notification_cmd` varchar(255) DEFAULT NULL,
  `notification_contact` varchar(255) DEFAULT NULL,
  `output` text,
  `retry` int(11) DEFAULT NULL,
  `service_description` varchar(255) DEFAULT NULL,
  `service_id` int(11) DEFAULT NULL,
  `status` tinyint(4) DEFAULT NULL,
  `type` smallint(6) DEFAULT NULL,
  PRIMARY KEY (`log_id`),
  KEY `host_name` (`host_name`(64)),
  KEY `service_description` (`service_description`(64)),
  KEY `status` (`status`),
  KEY `instance_name` (`instance_name`),
  KEY `ctime` (`ctime`),
  KEY `rq1` (`host_id`,`service_id`,`msg_type`,`status`,`ctime`),
  KEY `rq2` (`host_id`,`msg_type`,`status`,`ctime`),
  KEY `host_id` (`host_id`,`service_id`,`msg_type`,`ctime`,`status`),
  KEY `host_id_2` (`host_id`,`msg_type`,`ctime`,`status`)
) ENGINE=InnoDB AUTO_INCREMENT=63993 DEFAULT CHARSET=utf8 ;
```

## Mise à jour graphique et partitioning

Réaliser la mise à jour graphique de l’interface web en déplacent le dossier d’installation dans le répertoire web :
> **INFO :** Il faut bien supprimier les fichiers temporaires de la précédente installation pour ne pas créer de problème de l'exécution des scripts de mise à jour.

- **@NEW-CENTRAL@** :

```bash
rm -f /var/lib/centreon/installs/install-22.04.*/tmp/Update-*
mv /var/lib/centreon/installs/install-22.04.* /usr/share/centreon/www/install/
```

Mettre en place le partitioning des tables data_bin et logs. Attention, vérifier dans l'interface que la rétention configuré pour les partitions (Administration  >  Parameters  >  Options) convient :

- **@NEW-CENTRAL@** :

```bash
/opt/rh/rh-php72/root/usr/bin/php /usr/share/centreon/bin/centreon-partitioning.php -m data_bin
/opt/rh/rh-php72/root/usr/bin/php /usr/share/centreon/bin/centreon-partitioning.php -m logs
```

## Reprise des acquittements

Nous utilisons le fichier retention.dat de l'ancienne plateforme pour reprendre les acquittements :

- **@OLD-CENTRAL@** :

```bash
scp /var/log/centreon-engine/retention.dat root@'@NEW_CENTRAL_IP@':/var/log/centreon-engine/
```

- **@OLD-POLLER@** :

```bash
scp /var/log/centreon-engine/retention.dat root@'@NEW_POLLER_IP@':/var/log/centreon-engine/
```

## Lancement de la nouvelle infrastructure

Redémarrer tous les processus du central en relevant l'heure en timestamp qui servira pour la reprise des données.

- **@NEW-CENTRAL@** :

```bash
systemctl start crond cbd gorgoned centengine
```

Depuis l’interface web, générer et déplacer les fichiers de configuration du serveur central et des Pollers puis redémarrer les engines (central + pollers).

## Reprise des données historiques

Plutôt que d'utiliser la méthode traditionnel par mysqldump (dump de l'ancienne table, intégration dans une table temporaire sur la nouvelle plateforme puis déversement de la table temporaire dans la table en production), nous utilisons ici un export / import des données brutes via fichier. Cette méthode est beaucoup plus rapide.

### Prérequis

Positionner le `max_allowed_packet` à **128M** : `max_allowed_packet = 128M` dans le fichier /etc/my.cnf.d/centreon.cnf + restart de MariaDB

L'espace disque suffisant pour héberger les fichiers de données.

Il est conseillé de commencer par un petit export (ex 7j) pour vérifier la taille du fichier et ainsi calculer l'espace disque nécessaire pour les prochains exports. Cela permet également de vérifier l'impact sur la production lors de l'import (ralentissement possible).

### Première reprise de données

#### Extraction 1

Exemple : la nouvelle plateforme à été mise en production le 10/02/2019 à 15h10. Le timestamp correspondant relevé précédemment est 1549811400. Nous voulons récupérer 7 jours de données, soit une borne inférieur au 03/02/2019. Nous prenons cette date à minuit, cela simplifie les calcul pour la suite. Le timestamp correspondant est donc 1549152000, ce qui nous donne la requête suivante :

- **@OLD-DB-MariaDB@** :

```sql
SELECT * INTO OUTFILE '/var/lib/mysql/data_bin_1549152000-1549811400.txt'   FIELDS TERMINATED BY ';' OPTIONALLY ENCLOSED BY '"'   LINES TERMINATED BY '\n'   FROM centreon_storage.data_bin WHERE ctime BETWEEN '1549152000' AND '1549811400';
```

Transférer le fichier data_bin_1549152000-1549811400.txt sur le nouveau serveur de base de données :

- **@OLD-CENTRAL@** :

```bash
scp /var/lib/mysql/data_bin_1549152000-1549811400.txt root@'@NEW_CENTRAL_IP@':/var/lib/mysql/
```

Puis procéder à l'import

#### Insert 1

- **@NEW-DB-MariaDB@** :

```sql
LOAD DATA INFILE '/var/lib/mysql/data_bin_1549152000-1549811400.txt' INTO TABLE centreon_storage.data_bin  FIELDS TERMINATED BY ';' OPTIONALLY ENCLOSED BY '"'  LINES TERMINATED BY '\n';
```

Si l'export / import s'est bien passé, procéder à la reprise de données suivante.

> Exemple, nous voulons maintenant récupérer un mois. Nous utilisons la borne inférieur précédente (le 03/02/2019 minuit, soit 1549152000) comme pivot. Enlever 1 seconde a ce timestamp car l'opérateur BETWEEN est inclusif.

Nos bornes sont donc :

`1549152000 - 1 = 1549151999` et `1549152000 - (86400*NBJOURAEXPORTER) = 1549152000 - (86400*30) = 1546560000` (soit le 04/01/2019 à minuit).

Cette borne peut être prise arbitrairement. On aurait pu ici prendre le 01/01/2019 pour plus de simplicité pour la suite (pour ensuite récupérer des mois complet)

#### Extraction 2

- **@OLD-DB-MariaDB@** :

```sql
SELECT * INTO OUTFILE '/var/lib/mysql/data_bin_1546560000-1549151999.txt'   FIELDS TERMINATED BY ';' OPTIONALLY ENCLOSED BY '"'   LINES TERMINATED BY '\n'   FROM centreon_storage.data_bin WHERE ctime BETWEEN '1546560000' AND '1549151999';
```

Transférer le fichier data_bin_1546560000-1549151999.txt sur le nouveau serveur de base de données puis procéder à l'import :

#### Insert 2

- **@NEW-DB-MariaDB@** :

```sql
LOAD DATA INFILE '/var/lib/mysql/data_bin_1546560000-1549151999.txt' INTO TABLE centreon_storage.data_bin  FIELDS TERMINATED BY ';' OPTIONALLY ENCLOSED BY '"'  LINES TERMINATED BY '\n';
```

Si tout s'est également bien passé, continuer l'export / import des mois restant à reprendre puis faire de même pour la table logs.