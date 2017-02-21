# Configuration VPS Debian 8

## Installation serveur

### Connexion au serveur
	ssh root@SERVER_NAME
   
### Modification du mot de passe **root**
	passwd root

### Mise à jour du serveur
    apt-get update
    apt-get upgrade

### Mise à jour de la zone géographique (si besoin)
    dpkg-reconfigure tzdata

### Installation d'un éditeur de texte
	apt-get install nano

### Création nouvel utilisateur
	useradd -m -s /bin/bash USERNAME
	passwd USERNAME

### Configuration SSH
	nano /etc/ssh/sshd_config

#### Changer le port
	Port NEW_PORT
#### Interdire l'accès **root**
	# Interdire l'acces en root
	PermitRootLogin no
	# Nom de l'utilisateur
	AllowUsers USERNAME
#### Redémarrer le service ssh
	service ssh restart

Pour se connecter on utilisera maintenant

	ssh USERNAME@SERVER_NAME -p NEW_PORT

Puis pour passer en root

	su

#### Bonus : Authentification par clé SSH
**Sur le serveur**
   
	cd /home/USERNAME
	mkdir -p /home/USERNAME/.ssh
	chown USERNAME:USERNAME /home/USERNAME/.ssh
	chmod 700 /home/USERNAME/.ssh
	touch /home/USERNAME/.ssh/authorized_keys
	chown USERNAME:USERNAME /home/USERNAME/.ssh/authorized_keys
	chmod 600 /home/USERNAME/.ssh/authorized_keys

**Sur le client**

	ssh-keygen -t rsa
	
***Mac OS***

	ssh-add -K ~/.ssh/SSH_KEY_NAME

***Debian***

	apt-get install keychain
	nano /home/USERNAME/.bashrc
	
On ajoute la ligne suivante :

	eval `keychain --eval --agents ssh SSH_KEY_NAME`

**Copie de la clé sur le serveur**

_Via ssh-copy-id_

	ssh-copy-id -i ~/.ssh/SSH_KEY_NAME.pub USERNAME@SERVER_NAME -p NEW_PORT

_Manuellement_
Copier contenu de la clé publique dans **/home/USERNAME/.ssh/authorized_keys**
Vérifier les droits :

	chmod 600 /home/USERNAME/.ssh/authorized_keys
	chown USERNAME:USERNAME /home/USERNAME/.ssh/authorized_keys

### Firewall
On met en place un firewall pour bloquer toutes les connexions et n'autoriser que celles dont nous avons besoin.

On installe d'abord **iptables**

	apt-get install iptables

On crée le fichier **/etc/init.d/firewall**. et on met le code suivant :

	#!/bin/sh 
	### BEGIN INIT INFO
	# Provides: firewall
	# Required-Start: $remote_fs $syslog
	# Required-Stop: $remote_fs $syslog
	# Default-Start: 2 3 4 5
	# Default-Stop: 0 1 6
	# Short-Description: Démarre les règles iptables
	# Description: Charge la configuration du pare-feu iptables
	### END INIT INFO
	
	# Réinitialise les règles
	iptables -t filter -F 
	iptables -t filter -X 
	 
	# Bloque tout le trafic
	iptables -t filter -P INPUT DROP 
	iptables -t filter -P FORWARD DROP 
	iptables -t filter -P OUTPUT DROP 
	  
	# Autorise les connexions déjà établies et localhost
	iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 
	iptables -A OUTPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 
	iptables -t filter -A INPUT -i lo -j ACCEPT 
	iptables -t filter -A OUTPUT -o lo -j ACCEPT 
	   
	# ICMP (Ping)
	iptables -t filter -A INPUT -p icmp -j ACCEPT 
	iptables -t filter -A OUTPUT -p icmp -j ACCEPT 
	    
	# SSH
	iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT    # Attention, si vous avez changé le port SSH dans le fichier /etc/ssh/sshd_config
	iptables -t filter -A OUTPUT -p tcp --dport 22 -j ACCEPT 
	     
	# DNS
	iptables -t filter -A OUTPUT -p tcp --dport 53 -j ACCEPT 
	iptables -t filter -A OUTPUT -p udp --dport 53 -j ACCEPT 
	iptables -t filter -A INPUT -p tcp --dport 53 -j ACCEPT 
	iptables -t filter -A INPUT -p udp --dport 53 -j ACCEPT 
	     
	# NTP (horloge du serveur) 
	iptables -t filter -A OUTPUT -p udp --dport 123 -j ACCEPT
	     
	# HTTP
	iptables -t filter -A OUTPUT -p tcp --dport 80 -j ACCEPT 
	iptables -t filter -A INPUT -p tcp --dport 80 -j ACCEPT 
	# HTTP Caldav
	iptables -t filter -A OUTPUT -p tcp --dport 8008 -j ACCEPT
	iptables -t filter -A INPUT -p tcp --dport 8008 -j ACCEPT
	      
	# HTTPS
	iptables -t filter -A OUTPUT -p tcp --dport 443 -j ACCEPT
	iptables -t filter -A INPUT -p tcp --dport 443 -j ACCEPT
	# HTTPS Caldav
	iptables -t filter -A OUTPUT -p tcp --dport 8008 -j ACCEPT
	iptables -t filter -A INPUT -p tcp --dport 8443 -j ACCEPT
	      
	# FTP 
	iptables -t filter -A OUTPUT -p tcp --dport 20:21 -j ACCEPT 
	iptables -t filter -A INPUT -p tcp --dport 20:21 -j ACCEPT 
	
	# MySQL
	iptables -A INPUT -p tcp -s IP_DU_SERVEUR_AUTORISE --dport 3306 -j ACCEPT
	iptables -A INPUT -p tcp --dport 3306 -j DROP
	
	# RSYNC
	iptables -t filter -A INPUT -p tcp --dport 873 -m state --state NEW,ESTABLISHED -j ACCEPT
	iptables -t filter -A OUTPUT -p tcp --dport 873 -m state --state NEW,ESTABLISHED -j ACCEPT
	      
	# Mail SMTP 
	iptables -t filter -A INPUT -p tcp --dport 25 -j ACCEPT 
	iptables -t filter -A OUTPUT -p tcp --dport 25 -j ACCEPT 
	iptables -t filter -A INPUT -p tcp --dport 587 -j ACCEPT
	iptables -t filter -A OUTPUT -p tcp --dport 587 -j ACCEPT
	iptables -t filter -A INPUT -p tcp --dport 465 -j ACCEPT
	iptables -t filter -A OUTPUT -p tcp --dport 465 -j ACCEPT
	       
	# Mail POP3
	iptables -t filter -A INPUT -p tcp --dport 110 -j ACCEPT
	iptables -t filter -A OUTPUT -p tcp --dport 110 -j ACCEPT
	iptables -t filter -A INPUT -p tcp --dport 995 -j ACCEPT 
	iptables -t filter -A OUTPUT -p tcp --dport 995 -j ACCEPT 
	        
	# Mail IMAP
	iptables -t filter -A INPUT -p tcp --dport 993 -j ACCEPT 
	iptables -t filter -A OUTPUT -p tcp --dport 993 -j ACCEPT 
	iptables -t filter -A INPUT -p tcp --dport 143 -j ACCEPT
	iptables -t filter -A OUTPUT -p tcp --dport 143 -j ACCEPT
	        
	# Anti Flood / Deni de service / scan de port
	iptables -A FORWARD -p tcp --syn -m limit --limit 1/second -j ACCEPT
	iptables -A FORWARD -p udp -m limit --limit 1/second -j ACCEPT
	iptables -A FORWARD -p icmp --icmp-type echo-request -m limit --limit 1/second -j ACCEPT
	iptables -A FORWARD -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT

On commente les lignes dont nous n'avons pas besoin (Caldav, FTP...).

On rend le fichier exécutable

	chmod +x /etc/init.d/firewall

On le rajoute au démarrage du serveur

	update-rc.d firewall defaults

On lance le script

	/etc/init.d/firewall

### Fail2ban
On installe **fail2ban**

	apt-get install fail2ban

On ajoute le script de firewall :

	nano /etc/init.d/fail2ban

On modifie la ligne :

	# Required-Start:	$local_fs $remote_fs

Avec :
	
	# Required-Start:	$local_fs $remote_fs $firewall

On configure les jails :

	nano /etc/fail2ban/jail.conf

On redémarre le service :

	service fail2ban restart

### Amavisd-new, SpamAssassin et Clamav
On installe **Amavisd-new, SpamAssassin et Clamav**

	apt-get install amavisd-new spamassassin clamav clamav-daemon zoo unzip bzip2 arj nomarch lzop cabextract apt-listchanges libnet-ldap-perl libauthen-sasl-perl clamav-docs daemon libio-string-perl libio-socket-ssl-perl libnet-ident-perl zip libnet-dns-perl

On configure clamav :

	nano /etc/clamav/clamd.conf

On modifie/ajoute la ligne suivante :

	AllowSupplementaryGroups true

----------


### Compléter le bash
	apt-get install bash-completion

### Synchronisation de l'heure système
	apt-get install ntp ntpdate
	service ntp stop
	ntpdate 0.fr.pool.ntp.org

Mise à jour de la configuration

	nano /etc/ntp.conf

Et on ajoute les lignes suivantes au dessus de la liste des serveurs

	server 0.fr.pool.ntp.org
	server 1.fr.pool.ntp.org
	server 2.fr.pool.ntp.org
	server 3.fr.pool.ntp.org

On redémarre le service

	service ntp start

### CuRL
	apt-get install curl

## Serveur Web

### Apache
	apt-get install apache2 apache2-utils
	
On sécurise pour que le serveur soit moins bavard

	nano /etc/apache2/conf-enabled/security.conf

On modifie les lignes avec la configuration suivante :

	ServerTokens Prod
	ServerSignature Off
	TraceEnable Off

### PHP
	apt-get install libapache2-mod-php5 php5 php5-common php5-curl php5-dev php5-gd php5-intl php-pear php5-imagick php5-imap php5-json php5-mcrypt php5-memcache php5-mysql php5-pspell php5-recode php5-snmp snmp php5-sqlite php5-tidy php5-xmlrpc php5-xsl php5-apcu

On modifie quelques réglages de PHP

	nano /etc/php5/apache2/php.ini

On modifie les lignes avec la configuration suivante :

	;Taille maximum des fichiers à uploader (2MB par défaut)
	upload_max_filesize = 20M
	post_max_size = 20M

	;Activation de l'UTF-8 par défaut
	mbstring.language = UTF-8
	mbstring.internal_encoding = UTF-8
	mbstring.http_input = UTF-8
	mbstring.http_output = UTF-8
	mbstring.detect_order = auto

On active la réécriture

	a2enmod rewrite

On redémarre apache

	service apache2 restart

### MySQL
	apt-get install mysql-server mysql-client
 
On continue l'installation 

	mysql_install_db
On sécurise

	/usr/bin/mysql_secure_installation

On répond **n** pour le changement de mot de passe et **y** à toutes les autres questions.

On configure

	nano /etc/mysql/my.cnf

On modifie les paramètres suivants :

	[client]
	port = NOUVEAU_PORT_MYSQL
	default-character-set = utf8
	
	[mysqld]
	port = NOUVEAU_PORT_MYSQL
	default_storage_engine = MYISAM
	collation-server = utf8_unicode_ci
	init-connect='SET NAMES utf8'
	character-set-server = utf8
	#log_bin = /var/log/mysql/mysql-bin.log
	#expire_logs_days = 10
	
	[mysql]
	default-character-set=utf8

On redémarre le serveur

	service mysql restart

### PhpMyAdmin
	apt-get install phpmyadmin

On configure l'url

	nano /etc/phpmyadmin/apache.conf

On modifie la ligne suivante

	Alias /NOUVEAU_NOM_PHPMYADMIN /usr/share/phpmyadmin

----------

### Git
On installe :

	apt-get install git


### Java
On crée le sources.list pour java :

	nano /etc/apt/sources.list.d/java-8-debian.list

On ajoute les lignes suivante :

	deb http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main
	deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main

On ajoute la clé du repository :

	apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys EEA14886

On met à jour les repositories et on installe java :

	apt-get update
	apt-get install oracle-java8-installer

On vérifie la version de java installée :

	javac -version


