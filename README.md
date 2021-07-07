# SELinux (Security Enhanced Linux)

Sorti en 2003, initialement créé pour la NSA.
Il s'agit d'un système de sécurité renforcé se trouvant dans les distributions déclinées de *Red Hat*, comme *CentOS*.

## Discretionary Access Control (DAC) 

Les systèmes **Linux et Unix** ont un contrôle d'accès discrétionaire (DAC) :

- La **propriété** des données (fichiers, dossiers...) est réparti entre *l'utilisateur propriétaire*, les *groupes* et les *autres utilisateurs*.
Chaque catégorie d'utilisateur est restrieint par un système de **permissions** (lecture, écriture, exécution). Ce qui est une forme de sécurité

- Les utilisateurs ont la possibilité (*"la discrétion"*) de modifier les permissions sur leurs **propres fichiers**. N'importe quel utilisateur peut exécuter la commande *chmod +rwx* sur son dossier personnel par exemple, sans qu'il en soit empêché. Aucun processus ni utilisateur ne pourra s'empêcher également d'accéder **à ce dossier**.

**L'utilisateur *root* outrepasse toutes ces contraintes.**

## Mandatory Access Control (MAC)

**Il s'agit du mode de fonctionnement de SELinux.**

Sur un système possèdant un contrôle d'accès mandataire (MAC), il existe une **politique d'accès**. Le changement d'accès à un répertoire à l'aide de paramètres DAC (cf. ci-dessus), **peut être empêché à l'aide d'une politique**, si cette dernière restreint le droit d'accès d'un utilisateur ou d'un processus par exemple.

La politique d'accès du MAC peut donc s'appliquer ressources, pas uniquement des dossiers ou des fichiers :

- Les utilisateurs
- La mémoire
- Les sockets
- Les ports TCP/UDP
- etc...

### Les politiques

Les politiques d'accès de SELinux qui sont majoritairement utilisées sont pour *Red Had Entreprise Linux* :

- "ciblé" (*"targeted"*), il s'agit de la politique par défaut. Elle vise une centaine de processus au sein de *RHEL*.

- "multi-niveau/multi-catégorie" (*"mls"*), il s'agit d'une autre politique qui est très complexe, et est utilisée dans les agences gouvernementales américaines (CIA, FBA, NSA...)

## Le fonctionnement de SELinux

### L'état de SELinux
Il est possible de déterminer **la politique de son système** en regardant :

- Le contenu de `/etc/selinux/config`

Qui possède principalement les valeurs suivantes :
```sh
SELINUX=enforcing
SELINUXTYPE=targeted
```
Signifiant la politique de SELinux au moment du boot de la machine. **Ici, SELinux est activé (*enforcing*).**

- L'exécution de `/usr/sbin/sestatus`

Qui donne l'exemple de contenu suivant :
```
SELinux status:		 enabled
SELinuxfs mount:	    /sys/fs/selinux
SELinux root directory: /etc/selinux
Loaded policy name:	 targeted
...
```
- L'exécution de `/usr/sbin/getenforce`
Affichant directement l'état de SELinux (*enforcing*,*permissive*, ou *disabled*)

Il existe ensuite de concepts importants dans SELinux, il s'agit de **l'étiquettage (*labeling*)**, et **la mise en vigueur de types (*type enforcement*)**

### L'étiquettage

Tous les ressources manipulables par SELinux (fichiers, processus, ports, etc.) possèdent tous ***une étiquette* de SELinux**
 
Plus particulièrement, pour les **fichiers et les dossiers**, les étiquettes sont stockés comme étant une extension d'attributs sur le système de fichier.

Quant aux processus, aux ports, etc., c'est **le noyau Linux** qui s'occupe de l'étiquettage.

#### Formattage

Une étiquette se présente de la manière suivante :
`user:role:type:niveau (optionnel, le niveau de la politique)`

C'est le **type** qui nous intéresse le plus, car SELinux met en vigueur sa politique avec cet élément (*type enforcement* pour rappel). Donc l'étiquette est le moyen permettant à SELinux de vérifier si des ressources ont le droit de communiquer entre elles.

### Exemple avec le serveur Apache
Le fichier exécutable de Apache se trouvant sur `/usr/sbin/httpd`, nous utilisons la commande `ls -lZ`.
Nous avons un affichage similaire à celui-ci :
```
[root@centos ~]# ls -lZ /usr/sbin/httpd
-rwxr-xr-x. 1 root root system_u:object_r:httpd_exec_t:s0 536888 Jan 4 00:17 /usr/sbin/httpd
```
Le dossier de configuration de Apache, `/etc/httpd` possède comme type d'étiquette `httpd_config_t` (pour `r:httpd_config_t:s0`)

Rappelons que SELinux peut aussi gérer les ***processus***, il est possible d'avoir des informations sur le processus de Apache (`httpd`) via `ps axZ`. Il est possible de faire pareil avec `netstat` pour examiner les ports de Apache, ou `semanage` qui est spécifique à SELinux :

```
[root@centos ~]# semanage port -l | grep [h]ttp
http_cache_port_t	tcp 	8080, 8118, 8123,...
http_cache_port_t	udp	 3130
http_port_t		  tcp 	80, 81, 443
...
```
Nous avons plusieurs types, `http_cache_port_t`, `http_port_t`...

### Mise en vigueur

Soit un processus qui fait tourner Apache / httpd, ce dernier aura comme type d'étiquette `httpd_t`.
Il est logique qu'il puisse communiquer avec un fichier, dont le type d'étiquette sera `httpd_config_t`.

Par contre ce même processus ne pourra pas communiquer avec le fichier `/etc/shadow` dont le type d'étiquette sera `shadow_t`. **Ce qui est une bonne chose en matière de sécurité : *shadow* étant le fichier contenant tous les utilisateurs avec leurs mots de passes !**. Même si un utilisateur fait un `chmod 777`, SELinux **bloquera quand même le processus *httpd* dans l'accès de *shadow***

### Manipuler les étiquettes

Comme indiqué ci-dessus, il est possible de vérifier les étiquettes avec des commandes :

- `ls -Z`
- `id -Z`
- `ps -Z`
- `netstat -Z` 

Il est possible également d'associer le drapeau `-Z` à `cp` ou `mkdir` 

### Les erreurs

**Lorsque SELinux signale une erreur, cela est dû qu'on fait quelque chose d'incorrect.**
Cela est dû à plusieurs facteurs :
b
- L'étiquettage est incorrect => **il faudra utiliser des outls pour corriger cela**
- La politique doit être réajustée => **il faudra passer des booléens ou des modules de politique**
- La politique peut être un bug => **signaler un ticket à Red Hat**
- Une instrusion a été effectuée

Désactiver SELinux n'est pas conseillé, car il ne pourra plus intervenir en cas de problème.

### Les booléens

Les booléens servent à activer / désactiver des paramètres de SELinux. Il est possible d'adapter la configuration de SELinux selon ses propres besoins.

Exemple : "*Doit-on autoriser le serveur FTP à accéder à nos dossiers personnels ?*"

Pour voir tous les booléens : `getsebool -a`

### Conseils et astuces

- Installer `setroubleshoot` et `setools` sur les machines où on souhaitera développer des modules de politique. De plus cela nous permettra d'avoir une **meilleure compréhension des erreurs signalées par SELinux.**

Redémarrer ensuite le service `auditd`

## Exemples dans la vie courante

### Mise en place d'un serveur Apache

#### Premier cas

Supposons que nous souhaitons rendre accessible le répertoire d'un utilisateur à Apache pour qu'il puisse afficher du contenu web :

- On active la **directive *UserDir*** dans `/etc/httpd/conf.d/userdir.conf`
- On donne la permission d'exécution à Apache : `chmod o+x`
- On redémarre le serveur Apache

On aura une erreur `Forbidden` dans le navigateur web.

Il se peut que SELinux **empêche l'accès au répertoire utilisateur** au processus de Apache.

Pour vérifier cela, il faut consulter les logs des démons (puisque le processus Apache, `httpd`, en est un) avec la commande `journalctl -xe`.

Il faut savoir que httpd refuse de lire les dossiers se trouvant dans un dossier utilisateur, puisqu'il n'en aura pas le droit.
C'est pour cela que les logs de `journalctl` proposent d'exécuter la commande `setsebool -P httpd_read_user_content 1` (dire à SELinux de permettre à httpd de consulter les répertoires utilisateurs)

#### Second cas

On décide cette fois-ci de placer notre site web dans le dossier par défaut, soit `/var/www/html`.
Jusqu'ici il n'y a pas de problème.

Le problème va apparaître lorsqu'on souhaite déplacer notre site web, qui se trouvait dans un autre dossier (dans $HOME par exemple), pour le mettre dans `/var/www/html` : un `Forbidden` apparaîtra aussi.

Comme on a appliqué un déplacement (`mv`), l'ancien propriétaire **est conservé**, alors que la copie applique au contenu copié, le propriétaire du dossier parent.

--> On applique un `chown`, **mais le problème sera toujours là**.

Si on creuse, `journalctl` nous propose d'exécuter la commande `setsebool` ci-dessus. Mais le `Forbidden` persiste. Faisons un `ls -lZ` du contenu de notre dossier html (on suppose que notre si:

```
[root@centos ~]# ls -ldZ /var/www/html/monsite
drw-rw-r--.  1  root root unconfined_u:object_r:user_home_t:s0 26 Jan 4 00:17 index.html
```

On peut s'apercevoir que le **type de l'étiquette / son contexte** n'est pas correcte : `user_home_t`.
Il faut lui donner la bonne valeur, on peut s'inspirer des **types des éléments de httpd**, comme `/var/www/html` : 

```
[root@centos ~]# ls -ldZ /var/www/html
drw-rw-r--.  1  root root unconfined_u:object_r:httpd_sys_content_t:s0 26 Jan 4 00:17 html
```
La valeur du type est `httpd_sys_content_t`.
On peut l'appliquer pour notre dossier `monsite`, il existe plusieurs méthodes possibles via `chcon` :

- On applique le nouveau type en le saisissant manuellement : `chcon -R -t httpd_sys_content_t /var/www/html/monsite`

- On s'inspire du type d'une structure valide (plus rapide) : `chcon -R --reference /var/www/html /var/www/html/monsite`

Si on souhaite **restaurer l'étiquette à son état initial**, on utilise `restorecon` : `restorecon -R /var/www/html/monsite`


Pour information, SELinux sait quel contexte associer à une ressource, car il pioche dans les fichiers du dossier `/etc/selinux/targeted/contexts/files/`, principalement dans le fichier `file_contexts` : ce dernier contient 4000 entrées !

### chcon vs semanage

Lorsqu'on souhaite modifier le contexte d'un répertoire, on peut aussi utiliser la commande `semanage` :

`semanage fcontext -a -t httpd_sys_content_t "/srv/www/website`

Cela va dire à SELinux que le contenu du dossier `/srv/www/website` devra porter comme contexte par défaut, `httpd_sys_content_t`.

Ou si on souhaite récupérer un contexte depuis un autre dossier (à l'instar de `chcon --reference`) :

`semanage fcontext -a -e /var/www/html /srv/www`

Mais quel est la différence entre les deux commandes ?

Puisqu'on inidique à SELinux, un contexte par défaut à un dossier, ce dernier ne sera pas affecté par `restorecon`, contrairement dans le cas de `chcon`.

Donc `semanage` permet d'affecter un nouveau contexte de manière **permanente**, tandis que pour `chcon`, cela est temporaire.**Toutes les informations ont été testées sur une VM Centos 7.**
