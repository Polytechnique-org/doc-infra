Fonctionnement de Mailman
=========================

Mailman est un serveur de listes mails.
Il s'occupe de la gestion des abonnements, des archives ...

La gestion sur les serveurs mail
--------------------------------

Un seul serveur mail a mailman installé et configuré pour envoyer des mails : c'est celui qui héberge le site web (à cause du RPC qui tourne en local).
Les autres MXs peuvent (éventuellement) recevoir des mails à destination d'une liste ; dans ce cas, ils transmettront au plus tôt à svoboda.

Le serveur de développement peut aussi avoir mailman installé.

La gestion sur le Web
---------------------

XMLRPC
------

Le démon RPC tourne sous ``supervise``, un programme des daemontools, qui se charge de relancer un service (exécutable) s'il s'arrête. Quand tout se passe bien, on peut utiliser ``svc -[option]`` pour (re)lancer / arrêter / ... le démon RPC.

Le RPC reçoit des commandes depuis le site web, qui viennent par XMLRPC. (Détails)

Debug
-----

Mailman, comme postfix, possède plusieurs queues mail.
Elles sont dans plusieurs sous-dossiers de ``/var/lib/mailman/qfiles``.

On dispose de

* ``/var/lib/mailman/bin/show_qfiles`` pour inspecter le contenu d'un message (argument = fichier ``.pck`` du message) ;
* ``/var/lib/mailman/bin/unshunt`` pour libérer des messages.
* ``/var/lib/mailman/bin/discard`` pour supprimer un message en modération (dans le cas de spam vraiment massif).

Mailman logge dans ``/var/log/mailman``.

Archives
--------

Les archives de mailman se trouvent dans ``/var/lib/mailman/archives/private``. Par exemple la liste ma-liste@listes.example.org a ses archives dans un fichier MBOX situé dans ``/var/lib/mailman/archives/private/ma-liste.mbox/ma-liste.mbox``.
À Polytechnique.org, la visualisation de ces archives sur internet utilise Banana, qui communique avec mailman en NNTP pour obtenir les arborescences de messages.

Pour supprimer un message, il suffit d'éditer le fichier MBOX avec par exemple Mutt::

    cp ma-liste.mbox modif.mbox
    mutt -f modif.mbox

À la fin de l'édition du fichier mbox, Mutt a pu ajouter des entêtes suivants dans les messages::

    Status: O
    Content-Length: 357
    Lines: 15

Pour les supprimer, il suffit d'éxecuter la commande ``sed`` suivante::

    sed '/^Status: /{N;N;/Content-Length: .*Lines: /d}' < modif.mbox > modif-clean.mbox

À la fin du traitement, si ``ma-liste.mbox`` n'a pas été modifié (ie. si aucun message n'a été reçu), il est possible de régénérer les archives comme décrit sur http://wiki.list.org/pages/viewpage.action?pageId=4030681)::

    mv ma-liste.mbox OLD-ma-liste.mbox
    mv modif-clean.mbox ma-liste.mbox
    cd /var/lib/mailman
    ./bin/arch ma-liste

De plus, il faut vider le cache de banana concernant les archives de la liste::

    rm ~web/prod/platal/spool/banana/MLArchives/ma-liste/*
