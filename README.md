# Installation-Freeradius-
FreeRADIUS est utilisé comme serveur externe RADIUS (Remote Authentication Dial-In User Service).

### Installation Apache
####On installe Apache dans le cas où nous utilisons Daloradius pour la gestion d'utilisateurs Freeradius par interface graphique.

    apt -y install apache2
    systemctl enable --now apache2

#### Installation MySQL/MariaDB
#Pour installer une base de données pour notre serveur Radius. 

    apt -y install mariadb-server

###Sécurisez MariaDB  

    mysql_secure_installation

#A partir d'ici, les instructions de la console devraient nous guider tout au long du processus.

#### Installer PHP et des modules PHP supplémentaires

    apt -y install php libapache2-mod-php php-{gd,common,mail,mail-mime,mysql,pear,mbstring,xml,curl}

#### Installation paquet Freeradius 

###FreeRADIUS est installé avec deux modules supplémentaires , freeradius-utils et  freeradius-mysql :

    apt -y install freeradius freeradius-mysql freeradius-utils -y

### Autoriser FreeRADIUS dans le pare-feu

##Dans le cas un pare-feu est utilisé , il est nécessaire d’autoriser l’accès aux ports 1812 et 1813, qui sont utilisés par RADIUS.

    ufw allow to any port 1812 proto udp
    ufw allow to any port 1813 proto udp

####  Tester le serveur RADIUS

#Nous utilisons le serveur RADIUS en mode debug pour voir s'il y a des erreurs.
systemctl stop freeradius
freeradius -X

###Le résultat devrait être suffisamment long et se terminer par quelque chose comme ça :

    Listening on auth address * port 1812 bound to server default
    Listening on acct address * port 1813 bound to server default
    Listening on auth address :: port 1812 bound to server default
    Listening on acct address :: port 1813 bound to server default
    Listening on auth address 127.0.0.1 port 18120 bound to server inner-tunnel
    Listening on proxy address * port 52025
    Listening on proxy address :: port 42807
    Ready to process requests

###Configuration FreeRadius pour utiliser Mysql/MariaDB

##Par défaut, FreeRADIUS n'est pas paramétré pour l'utilisation de MySQL ou MariaDB.
Il faut commencer par créer une nouvelle base de données et les informations d'identification, que FreeRADIUS utilisera.

    mysql -u root -p

#la base de données et les informations d’identification qui seront utilisées par FreeRADIUS:

    MariaDB [(none)]> CREATE DATABASE radius_db;
    MariaDB [(none)]> CREATE USER 'radius'@'localhost' IDENTIFIED BY 'PASSWORD';
    MariaDB [(none)]> GRANT ALL ON radius_db.* TO radius@localhost IDENTIFIED BY "PASSWORD";
    MariaDB [(none)]> FLUSH PRIVILEGES;
    MariaDB [(none)]> quit;

#Ensuite, il faut importer le schéma RADIUS MySQL fourni par FreeRADIUS.

    mysql -u root -p radius_db < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql

##Voici la commande pour vérifier la table de base de donnée Radius: 

    mysql -u root -p -e "use radius_db;show tables;"

#### Ensuite il faut créer un lien symbolique à partir du module SQL dans le répertoire /etc/freeradius/3.0/mods-enabled :

    ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/

#### Puis on configure FreeRADIUS pour utiliser MySQL / MariaDB.
Pour ce faire, il faut éditer  /etc/freeradius/3.0/mods-available/sql   

    apt install nano
    nano /etc/freeradius/3.0/mods-enabled/sql

### Il faut apporter les modifications suivantes :

    Changer driver = « rlm_sql_null » À  driver = « rlm_sql_mysql »
    Changer  dialecte = « sqlite » À  dialecte = « mysql »
    il faut commenter  la section TLS, il est décommenté par défaut

output: 
    mysql {
    # If any of the files below are set, TLS encryption is enabled
    # tls {
    #       ca_file = "/etc/ssl/certs/my_ca.crt"
    #       ca_path = "/etc/ssl/certs/"
    #       certificate_file = "/etc/ssl/certs/private/client.crt"
    #       private_key_file = "/etc/ssl/certs/private/client.key"
    #       cipher = "DHE-RSA-AES256-SHA:AES128-SHA"
    #       tls_required = yes
    #       tls_check_cert = no
    #       tls_check_cert_cn = no
    #}
    # If yes, (or auto and libmysqlclient reports warnings are
    # available), will retrieve and log additional warnings from
    # the server if an error has occured. Defaults to 'auto'
    warnings = auto
}
A partir d'ici il faut rechercher des informations de connexion, juste en dessous,  décommenter le serveur, le port, la connexion, le mot de passe et les remplacer par les informations d'identification MySQL créées précédemment pour FreeRADIUS :
# Connection info:
    server = "localhost"
    port = 3306
    login = "radius"
    password = "PASSWORD"

## Ensuite, il faut chercher  radius_db et remplacez la valeur par défaut par le nom de base de données

# Database table configuration for everything except Oracle

    radius_db = "radius_db"

décommentez la ligne read_clients = yes

# Set to 'yes' to read radius clients from the database ('nas' table)

# Clients will ONLY be read on server startup.
read_clients = yes

     Après ces modifications, enregistrez le paramètre et quitter.
   
# Il faut modifier les droits de groupe du fichier que nous venons de modifier:

    chgrp -h freerad /etc/freeradius/3.0/mods-available/sql
    chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql

Et redémarrez le service FreeRADIUS :

    systemctl restart freeradius.service

#### Pour finir, il faut lancer FreeRADIUS en mode débogage pour assurer que tout fonctionne.
Dans un premier temps, il faut arrêter le service en cours :

    systemctl stop freeradius.service

##### Ensuite exécutez la commande suivante pour exécuter FreeRADIUS en mode débogage :
freeradius -X

output:

    Listening on auth address * port 1812 bound to server default
    Listening on acct address * port 1813 bound to server default
    Listening on auth address :: port 1812 bound to server default
    Listening on acct address :: port 1813 bound to server default
    Listening on auth address 127.0.0.1 port 18120 bound to server inner-tunnel
    Listening on proxy address * port 52025
    Listening on proxy address :: port 42807
    Ready to process requests

####### Quitter le mode de débogage et redémarrez le service FreeRADIUS.

    systemctl start freeradius.service

### Configuration du controleur WLC en tant que client d'authentification, d'autorisation et de comptabilité (AAA) sur FreeRadius:

### Étape 1. Editez nano /etc/freeradius/3.0/clients.conf afin de définir la clé partagée pour WLC.

    nano /etc/freeradius/3.0/clients.conf

Étape 2. En bas, ajoutez l’adresse IP du contrôleur et la clé partagée.

client X.X.X.X {

        secret = passwd

        shortname = wifi
}

##### Configurer FreeRadius en tant que serveur RADIUS sur le controleur (WLC)

###Étape 1. Ouvrez l’interface graphique du WLC et accédez à SECURITY > RADIUS > Authentication > New 

#### Étape 2. Renseignez les informations du serveur RADIUS
#### mettre l'ip du serveur Radius
notre ip : x.x.x.x
pour la clé partagée, mettre celle du client Radius (dans notre cas, le controleur WLC).

### Étape 3. Affectez le serveur RADIUS au WLAN.

Accédez à Security > AAA Servers (Serveurs de sécurité AAA et choisissez le serveur RADIUS souhaité, 
puis cliquez sur Apply (Appliquer). Dans notre cas nous avons choisir notre serveur Radius "x.x.x.x"

V. Installation de daloRadius 1.3 

### Daloradius est une interface web open source qui facilite la gestion et l'administration du serveur FreeRADIUS. 
Il offre une interface conviviale pour gérer les utilisateurs, les comptes, les sessions, les journaux, les autorisations d'accès.

######## Configuration daloRadius

    wget https://github.com/lirantal/daloradius/archive/refs/tags/1.3.tar.gz

#Une fois l’archive téléchargée, exécutez la commande ci-dessous pour l’extraire.

    mkdir /var/www/html/daloradius

    tar xzf 1.3.tar.gz -C /var/www/html/daloradius --strip-components=1

### Importation du schéma de base de données daloRadius

##Il est nécessaire d'importer ces tables dans la base de données FreeRADIUS que nous avons créée avant lors d'installation de Freeradius.

    mysql -u root -p radius_db < /var/www/html/daloradius/contrib/db/fr2-mysql-daloradius-and-freeradius.sql
    mysql -u root -p radius_db < /var/www/html/daloradius/contrib/db/mysql-daloradius.sql

 ### Configurer la propriété des fichiers  de configuration web daloRadius à l'utilisateur web Apache

    chown -R www-data.www-data /var/www/html/daloradius/
    Configurer les permissions du fichier de configuration principal daloRADIUS sur 664;

    cp /var/www/html/daloradius/library/daloradius.conf.php{.sample,}

########Configuration des paramètres de connexion de la base de données DaloRADIUS

######Ouvrir le fichier de configuration daloRADIUS pour modifier et définir les paramètres de connexion de la base de données.

    nano /var/www/html/daloradius/library/daloradius.conf.php

######## A partir d'ici il faudra modifier les lignes suivantes et mettre les paramètres de bases de données Freeradius: 

    $configValues['FREERADIUS_VERSION'] = '2';
    configValues['CONFIG_DB_ENGINE'] = 'mysqli';
    $configValues['CONFIG_DB_HOST'] = 'localhost';
    $configValues['CONFIG_DB_PORT'] = '3306';
    $configValues['CONFIG_DB_USER'] = 'radius';
    $configValues['CONFIG_DB_PASS'] = '***';
    $configValues['CONFIG_DB_NAME'] = 'radius_db';

###Enregistrer le fichier de configuration et redémarrer Freeradius

    systemctl restart freeradius

##### Accès à l’interface Web daloRADIUS

#### Il faut ouvrir le port 80 avec cette commande:

    ufw allow 80/tcp

#####Tout d'abord il est nécessaire d'installer les modules DB et MDB2 en utilisant PEAR :

    pear install DB
    pear install MDB2

### accédez à daloRADIUS en utilisant l’adresse https://x.x.x.x/daloradius.
