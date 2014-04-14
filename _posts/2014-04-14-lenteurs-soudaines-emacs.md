---
layout: post
title: Lenteurs soudaines d'Emacs
---

Si comme moi, vous utilisez emacs un peu partout. Vous avez surement été confronté à des problèmes de lenteur lors de son lancement (5-15 secondes).
Ses lenteurs surviennent généralement quand internet fait défaut (ou netsoul) ou bien quand votre distribution linux préférée est fraichement installée et mal/non configurée.

Correction
----------
Pour rapidement fixer le problème, il vous suffit de modifier le fichier /etc/hosts afin que votre hostname puisse être "résolu" facilement (sans l'aide de serveurs dns externe qui ne seront, de toute façon, d'aucune aide):

```bash
[steven@vm-arch ~]$ cat /etc/hostname
vm-arch
```

Si l'on suit le contenu du fichier hostname, vous devez ajouter la ligne "127.0.0.1 *votre_hostname*" (et "::1 ..." pour la version ipv6) au fichiers hosts :

```bash
[root@vm-arch steven]# cat /etc/hosts
#
# /etc/hosts: static lookup table for host names
#

#<ip-address>   <hostname.domain.org>   <hostname>
127.0.0.1       localhost
::1             localhost

127.0.0.1       vm-arch
::1             vm-arch


# End of file
```

A partir de ce moment, votre emacs va retrouver un temps de lancement normal. Si vous souhaitez connaitre les raisons de ce problème, je vous invite à lire la suite du post.

Pourquoi
--------

Cette solution a été donnée sur quelques sites, dont stackoverflow, mais sans jamais fournir de réels raisons. Voici donc mes investigations.

La première chose a consisté à récupérer le code source d'emacs, disponible sur [Github emacs].

Puis à rechercher tous les appels effectués à des fonctions de conversion *ip <> nom de domaine* comme gethostbyname, gethostbyaddr; getaddrinfo pouvant potentiellement bloquer emacs (ma première hypothèse):

Un rapide *find* m'a permis de restreindre les recherches à un seul fichier "src/sysdep.c" (un second candidat était "src/gnutls.c" mais rapidement éliminé à cause de son nom portant sur une possible utilisation de tls - penser ssl, certificat, etc. ... - qui servirait à rien au lancement d'emacs).

```
./src/sysdep.c:1352:  char *hostname_alloc = NULL;
./src/sysdep.c:1353:  char hostname_buf[256];
./src/sysdep.c:1354:  ptrdiff_t hostname_size = sizeof hostname_buf;
./src/sysdep.c:1355:  char *hostname = hostname_buf;
./src/sysdep.c:1358:     again.  Apparently, the only indication gethostname gives of
./src/sysdep.c:1363:      gethostname (hostname, hostname_size - 1);
./src/sysdep.c:1364:      hostname[hostname_size - 1] = '\0';
./src/sysdep.c:1367:      if (strlen (hostname) < hostname_size - 1)
./src/sysdep.c:1370:      hostname = hostname_alloc = xpalloc (hostname_alloc, &hostname_size, 1,
./src/sysdep.c:1374:  /* Turn the hostname into the official, fully-qualified hostname.
./src/sysdep.c:1380:    if (! strchr (hostname, '.'))
./src/sysdep.c:1394:            if ((ret = getaddrinfo (hostname, NULL, &hints, &res)) == 0
[...]
./src/sysdep.c:1435:        hp = gethostbyname (hostname);
./src/sysdep.c:1463:        hostname = fqdn;
./src/sysdep.c:1468:  Vsystem_name = build_string (hostname);
./src/sysdep.c:1469:  xfree (hostname_alloc);
```

#### init_system_name (void) l.1345

Ces appels sont contenus dans une fonction nommée "init_system_name (void)" (source code: [init_system_name])
Cette fonction à pour objectif, semble-t-il, de déterminer le véritable nom de la machine (n'oubliez pas qu'emacs n'est pas un simple éditeur, mais embarque un système de plugin très complet basé sur le Lisp).

La première partie du code est dédié à la recherche du hostname (uname, ou gethostname).
La seconde à pour but de, je cite, *"Turn the hostname into the official, fully-qualified hostname."* et tente d'appeler plusieurs fois getaddrinfo ou gethostbyname:

```c
       for (count = 0;; count++)
          {
            if ((ret = getaddrinfo (hostname, NULL, &hints, &res)) == 0
                || ret != EAI_AGAIN)
              break;
            if (count >= 5)
              break;
            Fsleep_for (make_number (1), Qnil);
          }
[...]
        struct hostent *hp;
        for (count = 0;; count++)
          {

#ifdef TRY_AGAIN
            h_errno = 0;
#endif
            hp = gethostbyname (hostname);
#ifdef TRY_AGAIN
            if (! (hp == 0 && h_errno == TRY_AGAIN))
#endif
              break;
            if (count >= 5)
              break;
            Fsleep_for (make_number (1), Qnil);
          }
```

Nous pouvons ainsi voir nos appels "bloquant" suivis d'appels à une fonction nommée "Fsleep_for" s'occupant, quant à elle de mettre en pause emacs pendant n secondes -ici 1 par tour- (*"Pause, without updating display, for SECONDS seconds."*).
Sans compter le blocage générer par gethostbyname ou getaddrinfo - généralement 1 à 2 secondes par appel-, nous pouvons rester bloquer jusqu'à 5 secondes rien qu'en *attente*.

Cette fonction semble confirmer mon hypothèse de blocage dû aux appels à des services dns. Reste à déterminer si cette fonction est bien appelé au démarrage d'emacs.

##### Stack d'appels

J'ai décidé de rechercher qui appelait cette fonction:

```
$ find . -name "*.c" | xargs grep -n "init_system_name"
./src/editfns.c:101:  init_system_name ();
./src/sysdep.c:1345:init_system_name (void)
./src/w32proc.c:2614:        is to call init_system_name, saving and restoring the
./src/w32proc.c:2619:     init_system_name ();
```

En dehors de l'implémentation windows, cette fonction est appelée qu'une seule fois dans "editfns.c" à la ligne 101 [editfns.c:101] dans "init_editfns". Puis cette dernière est appelée dans le [main d'emacs] à la ligne [1562].
Tout cela nous confirme bien la tentative de résolution de noms par emacs lors de son démarrage.

#### Confirmation avec strace

Afin de confirmer l'ordre d'execution, j'ai décidé d'utiliser *strace* pour vérifier l'ordre des appels systèmes (C'est surtout qu'un appel système assez rare, "uname", est effectué au début de la fonction l.[1349]).

``` 
uname({sys="Linux", node="vm-arch", ...}) = 0                                = 0
open("/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 3
[...]
munmap(0x7f7b00aa7000, 4096)            = 0
open("/etc/host.conf", O_RDONLY|O_CLOEXEC) = 3
[...]
open("/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 3
[...]
open("/usr/lib/libnss_files.so.2", O_RDONLY|O_CLOEXEC) = 3
[...]
open("/usr/lib/libnss_dns.so.2", O_RDONLY|O_CLOEXEC) = 3
[...]
socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.5.2")}, 16) = 0
poll([{fd=3, events=POLLOUT}], 1, 0)    = 1 ([{fd=3, revents=POLLOUT}])
rollen=0, msg_flags=0}, 37}, \{\{msg_name(0)=NULL, msg_iov(1)=[{"\245\356\1\0\0\1\0\0\0\0\0\0\7vm-arch\vlocaldomain"\
..., 37}], msg_controllen=0, msg_flags=0}, 37}}, 2, MSG_NOSIGNAL) = 2
poll([{fd=3, events=POLLIN}], 1, 5000)  = 1 ([{fd=3, revents=POLLIN}])
ioctl(3, FIONREAD, [37])                = 0
recvfrom(3, "|\363\204\3\0\1\0\0\0\0\0\0\7vm-arch\vlocaldomain"..., 2048, 0, {sa_family=AF_INET, sin_port=htons(53\
), sin_addr=inet_addr("192.168.5.2")}, [16]) = 37
poll([{fd=3, events=POLLIN}], 1, 4997)  = 1 ([{fd=3, revents=POLLIN}])
ioctl(3, FIONREAD, [112])               = 0
recvfrom(3, "\245\356\201\203\0\1\0\0\0\1\0\0\7vm-arch\vlocaldomain"..., 2011, 0, {sa_family=AF_INET, sin_port=hto\
ns(53), sin_addr=inet_addr("192.168.5.2")}, [16]) = 112
close(3)                                = 0
socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.5.2")}, 16) = 0
poll([{fd=3, events=POLLOUT}], 1, 0)    = 1 ([{fd=3, revents=POLLOUT}])
sendmmsg(3, \{\{\{msg_name(0)=NULL, msg_iov(1)=[{"\17\267\1\0\0\1\0\0\0\0\0\0\7vm-arch\0\0\1\0\1", 25}], msg_controll\
en=0, msg_flags=0}, 25}, \{\{msg_name(0)=NULL, msg_iov(1)=[{"5\222\1\0\0\1\0\0\0\0\0\0\7vm-arch\0\0\34\0\1", 25}], m\
sg_controllen=0, msg_flags=0}, 25}}, 2, MSG_NOSIGNAL) = 2
poll([{fd=3, events=POLLIN}], 1, 5000)  = 1 ([{fd=3, revents=POLLIN}])
ioctl(3, FIONREAD, [100])               = 0
[...]
```

Cet extrait montre l'appel à uname, puis à la tentative de récupération du nom de domaine complet en utilisant la bibliothèque nss (name service switch) et ses plugins: dans un premier temps "_files" qui regarde dans /etc/hosts, puis "_dns" qui effectue des requêtes auprès des serveurs dns contenus dans /etc/resolv.conf.
Ici poll est bloquant (avec un timeout de 5 secondes), il est chargé d'attendre la possible résolution du nom.



[Github emacs]:https://github.com/mirrors/emacs
[sysdep.c]:https://github.com/mirrors/emacs/blob/master/src/sysdep.c
[init_system_name]:https://github.com/mirrors/emacs/blob/master/src/sysdep.c#L1345
[editfns.c:101]:https://github.com/mirrors/emacs/blob/master/src/editfns.c#L101
[1562]:https://github.com/mirrors/emacs/blob/master/src/emacs.c#L1562
[main d'emacs]:https://github.com/mirrors/emacs/blob/master/src/emacs.c#L705
[1349]:https://github.com/mirrors/emacs/blob/master/src/sysdep.c#L1349
