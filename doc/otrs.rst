OTRS
====

OTRS est le système utilisé pour gérer les mails envoyés à notre équipe de support.

Installation
------------

On commence en installant le paquet debian ``otrs2``. Ensuite, il faudra probablement copier la base de données pour ne pas perdre l'historique des tickets traités.

Ensuite, on s'amuse :

Configuration
Il y a deux fichiers fondamentaux :

* ``/usr/share/otrs/Kernel/Config.pm``
* ``/usr/share/otrs/.procmailrc``

Le premier sert à faire toute la configuration du service, dès les mots de passes aux adresses mail, passant par l'encodage des données.

Le deuxième nous sert pour filtrer les mails à destination d'OTRS selon leur provenance, ce qui permet de les placer dans des queues de support différentes selon qu'ils ont été envoyés à ``contact@``, ``support@`` ou ``tresorier@``.

Vérifier que le site est en mode sécurisé
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Théoriquement, avoir la variable
::

    $Self->{'SecureMode'} = 1;

dans le fichier ``/etc/otrs/Kernel/Config.pm`` suffit à mettre le site en mode sécurisé (c'est à dire, empêcher l'utilisation de l'installateur web, ce qui risque d'écraser les données pré-existantes).

Pour vérifier que le site est en mode sécurisé, il suffit de regarder la page http://support.polytechnique.org/otrs/installer.pl, et si elle ne donne pas d'erreur, c'est parce que le site n'est pas sécurisé et n'importe qui peut détruire notre BDD...

Mail de sortie avec BCC hotliners
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pour que toute l'équipe soit au courant des mails qu'un membre du support envoie via OTRS, nous effectuons un BCC vers ``hotliners@staff``. Comme il ne faut pas que les autres mails d'OTRS (différentes actions sur des tickets, l'arrivée des tickets qui notifie les abonnés) soient aussi copiés vers hotliners (pour éviter de flooder les support-men), on ne peut pas utiliser la variable ``SendmailBcc`` que propose OTRS.

Pour avoir le comportement désiré il est donc nécessaire de changer le fichier ``/usr/share/otrs/Kernel/System/Ticket/Article.pm``, à la fonction ``ArticleSend()`` n'importe où après que ``$Param{Bcc}`` ait été récupéré, et avant l'envoi du mail (via ``SendmailObject->Send()``::

    1548a1551
    >     if ($Self->{ConfigObject}->{XorgBcc}) {
    >        $Param{Bcc}  = "$Self->{ConfigObject}->{XorgBcc}, ".$Param{Bcc};
    >     }

et d'ajouter la nouvelle variable dans ``/etc/otrs/Kernel/Config.pm``::

    $Self->{'XorgBcc'} = 'hotliners@staff.polytechnique.org';

Problème avec le cron ``otrs.cleanup``

Message d'erreur typique dans le cron::

    Sujet: Cron <otrs@svoboda> [ -x $HOME/bin/otrs.cleanup ] && $HOME/bin/otrs.cleanup > /d[..]
    cat: /usr/share/otrs/var/spool/bounces: Is a directory
    cat: /usr/share/otrs/var/spool/spam: Is a directory

Pour corriger cela, il suffit de changer ``/usr/share/otrs/bin/otrs.cleanup`` ::

    34c34
    < for i in $OTRS_SPOOLDIR/* ; do
    ---
    > for i in `find $OTRS_SPOOLDIR -type f` ; do

pour que ce script trouve les bons fichiers.
Cela est dû à notre configuration procmail, peut-être qu'en la changeant vers une mbox et non des maildirs on n'aurait plus ce problème.

OTRS n'arrive pas à lire des mails avec un Content-type mal défini
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Si on regarde ``/usr/share/otrs/var/spool/``, on peut trouver des fois des messages qui n'ont pas été
délivrées à OTRS lorsque le script ``/usr/share/otrs/bin/otrs.cleanup`` est lancé (par un cron).

Cela se traduit par::

    otrs@svoboda: ~ % bin/otrs.cleanup
    Checking otrs spool dir...
    Starting otrs PostMaster... (/usr/share/otrs/var/spool/2) failed.

À une époque, nous laissions ces mails dans le ``[...]/spool`` sans jamais s'en occuper, en supposant qu'il n'y avait que du spam. Cette position a été changée, il faut donc corriger le Content-type mal formé de ces mails (par exemple on avait *Content-type: text/html; charset= iso-8859-1* pour un des mails que posaient un problème) pour retirer une espace superflue qui trouble l'interpréteur d'OTRS. On fait alors ::

    ``/usr/share/otrs/bin/otrs.cleanup``
    38c38
    <         if cat $i | $OTRS_POSTMASTER >> /dev/null 2>&1; then
    ---
    >         if sed -e "s/\(Content-type: .*charset=\) \(.*\)/\1\2/" $i | $OTRS_POSTMASTER >> /dev/null 2>&1; then

Récupération après une panne d'OTRS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pendant une panne d'OTRS, les messages ne sont pas acheminés entre Postfix et OTRS. Plus exactement, procmail peut échouer à exécuter ``/usr/share/otrs/bin/otrs.PostMaster.pl``. Dans ce cas, ``/usr/share/otrs/.procmailrc`` indique de transmettre les messages dans ``$SYS_HOME/var/spool/.``, c'est à dire ``/var/lib/otrs/spool/{1,2,3,...}``. Pour récupérer ces messages, il suffit d'exécuter à la main ``otrs.cleanup`` ou de faire une boucle sur les message en les donnant à ``otrs.PostMaster``.

crons
~~~~~

TODO : en gros, les crons de récupération mail peuvent être omis puisqu'on utilise procmail.

Enlever le pop-up de confirmation qu'un ticket a été sélectionné pour une action groupée
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Comme il est très utile de pouvoir utiliser des actions groupées pour fermer les spams, et il est très peu productif de, à chaque fois qu'un ticket est sélectionné pour une action groupée, devoir fermer une boîte que te l'informe, il est essentiel de pouvoir demander à OTRS de ne pas montrer cela.

Pour le faire il, on ajoute à ``/etc/otrs/Kernel/Config.pm``::

    $Self->{'Ticket::Frontend::BulkFeatureJavaScriptAlert'} = 0;

Autres variables de conf
~~~~~~~~~~~~~~~~~~~~~~~~

Il faut dé-blacklister le domaine "@me", qui est utilisé par beaucoup de monde. Actuellement, il est dans ``~otrs/Kernel/Config/Files/ZZZAuto.pm``. (configuré via l'interface web)

Ajouter un correcteur orthographique en français
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
TODO

À faire toujours lors d'une mise à jour
---------------------------------------

Les mises à jour d'OTRS sont plutôt compliquées parce qu'il faudra de toutes façons refaire une partie des changements du code (et le vérifier), et qu'en plus la BDD d'OTRS change souvent d'une version à l'autre. Il est donc très conseillé de se documenter (en cherchant sur Internet) sur ce qui peut se passer...

Mettre à jour les bases de données
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Lors des mises à jour de version mineure (a.b vers a.c), il est courant qu'OTRS change un peu les tables de la base de données (le plus souvent pour rajouter des colonnes avec informations supplémentaires).

Un bon commencement est donc de faire un backup de la base de données OTRS, puis d'installer la nouvelle version. Ensuite, on fait tourner les scripts qui font le *DBUpdate*, en général dans ``~otrs/scripts``. Normalement, cela se passe sans problèmes. Et au pire, on a de quoi recommencer (à n'importe quelle étape de ce qui suit) !

Reprendre la partie de l'installation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Lancer le site et :

* Vérifier la configuration (OTRS accède à la base de données, les tickets sont là, les bulk actions, ...).
* Vérifier que le site est en mode sécurisé

Ensuite, on passe aux hacks

* Tester le Bcc:hotliners (Normalement, il suffit de remettre le morceau de code dans ``Article.pm`` et vérifier que les Bcc marchent !)
* Corriger le cron otrs.cleanup
* Corriger les mauvais charset des Content-type

Et finalement,

* Relire les crons
* Ajouter un correcteur orthographique en français


Quelques notes de la migration 2.0.4 -> 2.2.7
---------------------------------------------

Dans la suite, on note quelques changements et vérifications que se sont avérées nécessaires lors de la dernière mise à jour d'OTRS (du 31/10/2010, version 2.0.4 -> 2.2.7), qui est venue dans le passage de etch à lenny sur svoboda.

L'encodage de la BDD a changé depuis la dernière version
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dans la version 2.0.4 d'OTRS que nous avions, la BDD sql était UTF-8 double-encoded pour que l'interface affiche correctement les mails. Visiblement, ce "problème" a été résolu avec la nouvelle version d'OTRS, mais les scripts de mise à jour de la BDD n'ont pas corrigé cela.

Pour dé-double-encoder, on a utilisé la méthode décrite dans http://www.blueboxgrp.com/news/2009/07/mysql_encoding et ça a bien marché (même si ça a pris 3h30 à le faire, en vérifiant beaucoup de choses et scriptant les commander à taper, mais les rentrant au fur et à mesure pour vérifier leur bon fonctionnement...).

Erreurs de prise en compte des configurations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On a réussi à faire marcher les bulk actions en changeant cette option via l'interface d'administration d'OTRS (Administrateur -> Configuration Système, recherche de l'option BulkFeatureJavaScriptAlert), ce qui a écrit dans le fichier ``/usr/share/otrs/Kernel/Config/Files/ZZZAuto.pm`` la ligne que nous avions mis dans ``/etc/otrs/Kernel/Config.pm``.

Le site refusait de passer en mode sécurisé, la variable dans ``Config.pm`` n'étant pas suffisante. Pour le mettre en mode sécurisé (pour de vrai !) nous avons aussi changé cette configuration dans les fichiers
``/usr/share/otrs/Kernel/Config/Files/ZZZAuto.pm``
``/usr/share/otrs/Kernel/Config/Defaults.pm``
ce qui est probablement un bug d'OTRS, mais en tout cas voici des endroits où chercher.

Problème avec le cron GenericAgent.pl
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Message d'erreur dans le cron::

    Sujet: Cron <otrs@svoboda> test -x $HOME/bin/GenericAgent.pl && $HOME/bin/GenericAgent.[..]
    Need Ticket::ViewableStateType in Kernel/Config.pm!

Apparemment, il s'est corrigé tout seul (on n'a pas vu de message d'erreur à part une fois). Cela dit, on a rajouté la valeur conseillée de cette variable de configuration pour que les scripts de mise à jour vers la version 2.4 marchent. Il peut avoir encore des choses à vérifier...

Quelques notes de la migration 2.2.7 -> 2.4.9
---------------------------------------------

Changement de la BDD
~~~~~~~~~~~~~~~~~~~~

Avec une migration OTRS il faut toujours exécuter les scripts de mise à jour de la BDD. Visiblement cela n'est pas automatiquement fait par debian, donc il faut penser de le faire à la main.

Apparition du DashBoard
~~~~~~~~~~~~~~~~~~~~~~~

Avec la nouvelle version de OTRS un DashBoard a été introduit en regroupant plein d'informations. Cette page est la page par défaut juste après le login. Il est par contre plus intéressant chez nous d'arriver directement à la page pour traiter des tickets. Pour le faire il suffit de changer dans http://support.polytechnique.org/otrs/index.pl?Action=AdminSysConfig&Subaction=Edit&SysConfigSubGroup=Frontend::Agent&SysConfigGroup=Ticket&#|Options de configuration: Ticket → Frontend::Agent la valeur de ``Frontend::CommonParam###Action:`` à ``AgentTicketQueue``.


Quelques outils
---------------

Lors de la migration 1.3 -> 2.0 d'OTRS, une certaine quantité de scripts ont été faits pour traiter la base de données ``otrs``, essentiellement pour nettoyer un peu (``flush*``) et faire la conversion des charsets (lors de 1.3, on était en latin1). Ils se trouvent dans ``svoboda:~bernardofpc/otrs2/``.

Export MailBox
~~~~~~~~~~~~~~

Pour exporter tout une file en un fichier MailBox, il suffit de concaténer tous les messages présents dans base de donnée. Cela peut se faire en une requête SQL exécutée avec la commande ``mysql -rsN`` (options ``--raw``, ``--silent`` et ``--skip-column-names``. Par exemple, la commande suivante exporte toute la file Support::

    mysql -rsN -u otrs -p otrs -e \
        'SELECT  ap.body \
           FROM  queue AS q \
      LEFT JOIN  ticket AS t ON (t.queue_id = q.id) \
      LEFT JOIN  article AS a ON (a.ticket_id = t.id) \
      LEFT JOIN  article_plain AS ap ON (ap.article_id = a.id) \
          WHERE  q.name = "Support" AND t.ticket_state_id=1;' > otrs-support.mbox
