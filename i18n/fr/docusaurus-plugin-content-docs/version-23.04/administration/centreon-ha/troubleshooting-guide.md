---
id: troubleshooting-guide
title: Dépanner Centreon-HA
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

## Une ressource ne démarre pas 

Si une ressource (par exemple une ressource du groupe centreon) ne démarre pas correctement, des *failed actions* apparaîtront dans `crm_mon`. Par exemple ci-dessous, `centreontrapd` a rencontré une erreur :

<Tabs groupId="sync">
<TabItem value="RHEL 8 / Oracle Linux 8" label="RHEL 8 / Oracle Linux 8">

```bash
Cluster Summary:
  * Stack: corosync
  * Current DC: @CENTRAL_MASTER_NAME@ (version 2.0.5-9.0.1.el8_4.1-ba59be7122) - partition with quorum
  * Last updated: Wed Sep 15 16:35:47 2021
  * Last change:  Wed Sep 15 10:41:50 2021 by root via crm_attribute on @CENTRAL_MASTER_NAME@
  * 2 nodes configured
  * 14 resource instances configured
Node List:
  * Online: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
Full List of Resources:
  * Clone Set: ms_mysql-clone [ms_mysql] (promotable):
    * Masters: [ @CENTRAL_MASTER_NAME@ ]
    * Slaves: [ @CENTRAL_SLAVE_NAME@ ]
  * Clone Set: php-clone [php]:
    * Started: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
  * Clone Set: cbd_rrd-clone [cbd_rrd]:
    * Started: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
  * Resource Group: centreon:
    * vip       (ocf::heartbeat:IPaddr2):        Started @CENTRAL_MASTER_NAME@
    * http      (systemd:httpd):         Started @CENTRAL_MASTER_NAME@
    * gorgone   (systemd:gorgoned):      Started @CENTRAL_MASTER_NAME@
    * centreon_central_sync     (systemd:centreon-central-sync):         Started @CENTRAL_MASTER_NAME@
    * cbd_central_broker        (systemd:cbd-sql):       Started @CENTRAL_MASTER_NAME@
    * centengine        (systemd:centengine):    Started @CENTRAL_MASTER_NAME@
    * centreontrapd     (systemd:centreontrapd):         Stopped
    * snmptrapd (systemd:snmptrapd):     Stopped

Failed Resource Actions:
* centreontrapd_start_0 on @CENTRAL_MASTER_NAME@ 'not running' (7): call=82, status=complete, exitreason='',
    last-rc-change='Wed Sep 15 13:42:19 2021', queued=1ms, exec=2122ms
```

</TabItem>
</Tabs>

Afin d'avoir plus d'informations sur la cause de cette panne, se connecter via SSH au nœud maître (celui où la ressource devrait tourner) et lancer la commande suivante :

```bash
systemctl status centreontrapd -l
```

Si jamais la sortie de cette commande ne suffit pas, il est possible d'en savoir plus via la commande 

```bash
pcs resource debug-start centreontrapd
```

Une fois la cause identifiée et corrigée, la commande à lancer pour que le cluster "oublie" ces erreurs et relance le service est :

```bash
pcs resource cleanup centreontrapd
```

## Une ressource ou un groupe de ressources ne démarre sur aucun des nœuds

Si suite à une bascule, qu'elle soit manuelle ou à cause d'une panne ou de l'arrêt d'un serveur, la situation suivante se produit :

<Tabs groupId="sync">
<TabItem value="RHEL 8 / Oracle Linux 8" label="RHEL 8 / Oracle Linux 8">

```bash
Cluster Summary:
  * Stack: corosync
  * Current DC: @CENTRAL_MASTER_NAME@ (version 2.0.5-9.0.1.el8_4.1-ba59be7122) - partition with quorum
  * Last updated: Wed Sep 15 16:35:47 2021
  * Last change:  Wed Sep 15 10:41:50 2021 by root via crm_attribute on @CENTRAL_MASTER_NAME@
  * 2 nodes configured
  * 14 resource instances configured
Node List:
  * Online: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
Full List of Resources:
  * Clone Set: ms_mysql-clone [ms_mysql] (promotable):
    * Slaves: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
  * Clone Set: php-clone [php]:
    * Started: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
  * Clone Set: cbd_rrd-clone [cbd_rrd]:
    * Started: [ @CENTRAL_MASTER_NAME@ @CENTRAL_SLAVE_NAME@ ]
```

</TabItem>
</Tabs>

Aucune erreur n'est remontée, mais le groupe centreon n'apparaît plus, et aucune de ses ressources n'est donc démarrée. Ce cas de figure est généralement dû à un enchainement de deux bascules (`pcs resource move ...`) sans avoir supprimé la contrainte par la suite. Pour le vérifier lancer:

```bash
pcs constraint show
```

<Tabs groupId="sync">
<TabItem value="RHEL 8 / Oracle Linux 8" label="RHEL 8 / Oracle Linux 8">

```bash
Location Constraints:
    Disabled on: @CENTRAL_SLAVE_NAME@ (score:-INFINITY) (role: Started)
    Disabled on: @CENTRAL_MASTER_NAME@ (score:-INFINITY) (role: Started)
Ordering Constraints:
Colocation Constraints:
  centreon with ms_mysql-clone (score:INFINITY) (rsc-role:Started) (with-rsc-role:Master)
  ms_mysql-clone with centreon (score:INFINITY) (rsc-role:Master) (with-rsc-role:Started)
Ticket Constraints:
```

</TabItem>
</Tabs>

On constate que le groupe centreon n'est plus "autorisé" à démarrer sur aucun des nœuds.

Pour libérer le groupe de ressources de ces contraintes, il faut lancer la commande :

```bash
pcs resource clear centreon
```

Les ressources devraient alors immédiatement se lancer.
