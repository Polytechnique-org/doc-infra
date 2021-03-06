Réplication MySQL
=================

Les serveurs de Polytechnique.org utilisent une réplication MySQL master-slave
pour synchroniser les modifications faites sur le site (qui modifie le master)
avec les serveurs qui ont besoin des données, par exemple pour transmettre les
emails aux adresses indiquées par les utilisateurs.


Mise en place de la réplication
-------------------------------

Toutes les données qui ont besoin d'être répliquées entre les serveurs se
situent dans la base de données ``x5dat``.

Les serveurs utilisent des tunnels stunnel (https://www.stunnel.org/) pour
communiquer entre eux, ce qui a pour conséquence que les connexions MySQL sont
toujours vues comme arrivant de ``localhost``.

Ainsi, l'utilisateur pour la réplication est créé sur le master par une commande
comme :

.. code-block:: mysql

    GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'localhost' IDENTIFIED BY 'password';
    FLUSH PRIVILEGES;

Comme décrit sur https://dev.mysql.com/doc/refman/5.5/en/replication-options-slave.html,
chaque serveur MySQL doit avoir un identifiant ``server-id`` unique (entier
positif de 32 bits).

Il faut aussi ajouter quelques lignes de configuration dans la section
``[mysqld]`` de la configuration des serveurs.

Sur le master, cette configuration peut être ajoutée dans un nouveau fichier
``/etc/mysql/conf.d/master.cnf``::

    [mysqld]
    log_bin                 = /var/log/mysql/mysql-bin.log
    expire_logs_days        = 7
    max_binlog_size         = 100M

Sur les slaves, la configuration équivalente peut avoir lieu dans le fichier
``/etc/mysql/conf.d/slave.cnf`` ::

    [mysqld]
    log_bin                 = /var/log/mysql/mysql-bin.log
    expire_logs_days        = 7
    max_binlog_size         = 100M
    report-host=@@HOST_FQDN@@
    replicate-do-db=x5dat

où ``@@HOST_FQDN@@`` est le fully-qualified domain name du slave. Il faut de
plus s'assurer que ``/etc/mysql/my.cnf`` contiennent la ligne suivante sur tous
les serveurs ::

    !includedir /etc/mysql/conf.d/

Alors, après avoir verrouillé la base de données maître en lecture et transféré
les données sur chaque slave, il est possible d'activer la réplication en
exécutant dans l'interface MySQL sur les slave la commande similaire à :

 .. code-block:: mysql

    mysql> CHANGE MASTER TO MASTER_HOST='127.0.0.1', MASTER_PORT=3307,
            MASTER_USER='replicator', MASTER_PASSWORD='password',
            MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=42;

Dans cette commande :

* ``MASTER_HOST`` doit indiquer une adresse IP au lieu de ``localhost`` pour
  forcer l'utilisation du réseau (et non de la socket Unix du serveur).
* ``MASTER_PORT`` correspond au port local de la connexion stunnel avec le
  master.
* ``MASTER_USER`` et ``MASTER_PASSWORD`` correspondent aux informations
  d'authentification du slave auprès du master (qui sont évidemment différents
  sur les serveurs en production).
* ``MASTER_LOG_FILE`` et ``MASTER_LOG_POS`` correspondent au informations
  données par ``SHOW MASTER STATUS`` sur le master au moment où les données
  ont été extraites pour être transférées sur le slave.


Vérification de l'état de la réplication
----------------------------------------

Sur le master, il est possible de vérifier l'état de la réplication MySQL avec :

 .. code-block:: mysql

    mysql> SHOW MASTER STATUS \G
    *************************** 1. row ***************************
                File: mysql-bin.002548
            Position: 509123
        Binlog_Do_DB: x5dat
    Binlog_Ignore_DB:

Sur le slave, une commande équivalente existe :

 .. code-block:: mysql

    mysql> SHOW SLAVE STATUS \G
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 127.0.0.1
                      Master_User: replicator
                      Master_Port: 3307
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.002548
              Read_Master_Log_Pos: 509123
                   Relay_Log_File: mysqld-relay-bin.000009
                    Relay_Log_Pos: 509269
            Relay_Master_Log_File: mysql-bin.002548
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB: x5dat
              Replicate_Ignore_DB:
               Replicate_Do_Table:
           Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
                       Last_Errno: 0
                       Last_Error:
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 509123
                  Relay_Log_Space: 509469
                  Until_Condition: None
                   Until_Log_File:
                    Until_Log_Pos: 0
               Master_SSL_Allowed: No
               Master_SSL_CA_File:
               Master_SSL_CA_Path:
                  Master_SSL_Cert:
                Master_SSL_Cipher:
                   Master_SSL_Key:
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error:
                   Last_SQL_Errno: 0
                   Last_SQL_Error:
      Replicate_Ignore_Server_Ids:
                 Master_Server_Id: 4

En particulier ``Seconds_Behind_Master`` doit toujours être ``0`` pour indiquer
que le slave est synchronisé avec le master.

Les fichiers de log binaire mentionnées se situent dans ``/var/log/mysql`` et
sont lisibles avec la commande ``mysqlbinlog``.


Réparer une réplication
-----------------------

Lorsque le master ou un des slave s'arrête pour une raison ou une autre (crash,
coupure de réseau, coupure d'alimentation électrique, etc.), au rétablissement
MySQL ne rétablit pas tout seul la synchronisation de la réplication. Ceci se
remarque en étudiant la sortie de la ``SHOW SLAVE STATUS``, qui indique une
erreur.

Relancer la réplication après arrêt violent du master
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Sur les slaves, regarder dans ``SHOW SLAVE STATUS`` le contenu de
  ``Master_Log_File``, qui est de la forme ``mysql-bin.XXXXXX``.
* Sur le master, regarder la dernière requête du fichier
  ``/var/log/mysql/mysql-bin.XXXXXX`` correspondant, avec par exemple :

  .. code-block:: sh

    mysqlbinlog /var/log/mysql/mysql-bin.XXXXXX | less

* Vérifier que cette requête a bien été exécutée sur chacun des slaves.
* Sur le master, regarder la première requête du fichier suivant,
  ``mysql-bin.YYYYYY``.
* Vérifier sur chacun des slaves que cette requête n'a pas été effectuée.
* Si ces vérifications d'intégrité réussissent, il est possible de relancer la
  synchronisation sur les slaves avec :

  .. code-block:: mysql

    mysql> STOP SLAVE;
    mysql> CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.YYYYYY', MASTER_LOG_POS=0;
    mysql> START SLAVE;

* Sinon, il faut réinstaller la réplication à partir d'un dump de la base de
  donnée effectué avec un read lock, ce qui induit un downtime des services
  utilisant la base de données sur le slave concerné.

Relancer la réplication après arrêt violent d'un slave
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Après l'arrêt violent d'un slave, la procédure est légèrement différente car le
master a continué dans le même fichier. Ainsi :

* ``SHOW SLAVE STATUS`` indique le ``Master_Log_File`` du master, mais
  ``Relay_Master_Log_File`` contient le nom du dernier fichier de log utilisé,
  de la forme ``mysql-bin.XXXXXX``, et ``Exec_Master_Log_Pos`` est la position
  dans ce fichier (et non ``Read_Master_Log_Pos``).
* En lisant sur le master ce fichier, il est possible de déterminer l'heure de
  la désynchronisation.  Par exemple ::

    # mysqlbinlog /var/log/mysql/mysql-bin.002548 |grep -C2 509123
    /*!*/;
    # at 509096
    #150719 12:09:02 server id 4  end_log_pos 509123 Xid = 30848121
    COMMIT/*!*/;
    # at 509123
    #150719 12:10:01 server id 4  end_log_pos 509200 Query thread_id=7970969 exec_time=0 error_code=0
    SET TIMESTAMP=1437300601/*!*/;

* Sur cet exemple, la synchronisation aurait été coupée après la requête
  exécutée à 12:09:02, et il est possible de vérifier l'état de la base de
  données en fonction.
* Si cette vérification montre que l'état de la base de données est bien
  cohérent avec celui attendu par rapport aux requêtes du fichier de log, alors
  il est possible de relancer le serveur avec une commande similaire à (dans
  l'exemple ici) :

  .. code-block:: mysql

    mysql> CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.002548', MASTER_LOG_POS=509123;

Une fois la réplication rétablie, il est possible de suivre la décroissance de
``Seconds_Behind_Master`` dans ``SHOW SLAVE STATUS \G`` jusqu'à la valeur ``0``.

Réinitialiser la réplication après une désynchronisation extrême et violente
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Il peut arriver suite à une succession de coupures brutales du master que les
slaves s'arrêtent avec le message d'erreur suivant dans ``SHOW SLAVE STATUS \G``
::

    Last_Errno: 1452
    Last_Error: Error 'Cannot add or update a child row: a foreign key
    constraint fails (`x5dat`.`group_event_participants`, CONSTRAINT
    `group_event_participants_ibfk_2` FOREIGN KEY (`eid`, `item_id`) REFERENCES
    `group_event_items` (`eid`, `item_id`) ON DELETE CASCADE ON UPDATE CA)' on
    query. Default database: 'x5dat'. Query: 'INSERT INTO
    group_event_participants (eid, uid, item_id, nb, flags, paid) ...

Lorsque cela est le signe qu'il y a des données sur le master qui manquent dans
les slaves, le plus simple consiste à réinitialiser la réplication. Cela
s'effectue en deux temps : tout d'abord il faut effectuer une sauvegarde
("dump") complète de la base de données ``x5dat`` sur le master, et ensuite
appliquer ce dump sur chacun des slaves. Voici une procédure plus détaillée :

* Sur le master :

  #. Basculer en mode lecture seule tout ce qui peut écrire dans la base de
     données. Il s'agit de modifier la configuration du site web pour le passer
     en lecture seule, en décommentant ``mode = "r"`` dans la section ``[Core]``
     de ``~web/prod/platal/configs/platal.conf``.

  #. Verrouiller la base de données en lecture seule
     (cf. https://dev.mysql.com/doc/refman/5.7/en/replication-solutions-backups-read-only.html) :

     .. code-block:: mysql

        mysql> FLUSH TABLES WITH READ LOCK;
        mysql> SET GLOBAL read_only = ON;

  #. Noter la position actuelle du master, position ``XXXX`` dans
     ``mysql-bin.YYYYYY`` ::

        mysql> SHOW MASTER STATUS \G
        *************************** 1. row ***************************
                  File: mysql-bin.000197
              Position: 3678385
          Binlog_Do_DB: x5dat
        Binlog_Ignore_DB:
        1 row in set (0.00 sec)

     Ici, ``XXXX`` est 3678385 et ``YYYYYY`` est 000197.

  #. Sauvegarder le contenu de la base de données :

     .. code-block:: sh

        mysqldump --opt -u admin -p x5dat > "$(date '+%Y-%m-%d')_x5dat_dump.sql"

  #. Vérifier que la position de réplication n'a pas changé dans ``SHOW MASTER STATUS \G``.

  #. Lever le verrou de la base de données :

     .. code-block:: mysql

        mysql> SET GLOBAL read_only = OFF;
        mysql> UNLOCK TABLES;

  #. Désactiver le mode lecture seule du site web.

* Puis sur chaque slave, un par un :

  #. Transférer le dump de la base de donnée du master (avec ``scp`` par exemple)
     et vérifier que le fichier n'a pas été corrompu lors des transferts (avec
     ``sha256sum`` par exemple).

  #. Arrêter vraiment toute tentative de reprise de réplication en MySQL :

     .. code-block:: mysql

        mysql> STOP SLAVE;

  #. Couper les services utilisant la base de donnée :

     .. code-block:: sh

        service postfix stop

  #. Appliquer le dump, avec une barre de progression :

     .. code-block:: sh

        pv "$(date '+%Y-%m-%d')_x5dat_dump.sql" | mysql -u admin -p x5dat

  #. Relancer la réplication à la position ``XXXX`` dans
     ``mysql-bin.YYYYYY``, en utilisant les nombres trouvés précédemment sur le
     master :

     .. code-block:: mysql

        mysql> CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.YYYYYY', MASTER_LOG_POS=XXXX;
        mysql> START SLAVE;

  #. Vérifier que la réplication se déroule sans erreur :

     .. code-block:: mysql

        mysql> SHOW SLAVE STATUS \G

  #. Démarrer les services qui ont été précédemment arrêtés :

     .. code-block:: sh

        service postfix start

  #. Vérifier que les services fonctionnent de nouveau correctement.
