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
