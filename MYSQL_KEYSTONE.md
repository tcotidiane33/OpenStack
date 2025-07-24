Pour initialiser la base de données de Keystone (et plus généralement pour préparer votre base de données MySQL/MariaDB pour OpenStack), voici les étapes à suivre :

### 1. Installer MySQL/MariaDB (si ce n'est pas déjà fait)
```bash
sudo apt install -y mariadb-server python3-pymysql
```

### 2. Configurer MySQL/MariaDB
Créez un fichier de configuration pour MySQL :
```bash
sudo nano /etc/mysql/mariadb.conf.d/99-openstack.cnf
```

Ajoutez ces configurations de base :
```ini
[mysqld]
bind-address = 0.0.0.0  # Ou l'IP de votre controller
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

### 3. Redémarrer MySQL et sécuriser l'installation
```bash
sudo systemctl restart mysql
sudo mysql_secure_installation
```
(Répondez aux questions de sécurité selon vos besoins)

### 4. Créer la base de données et les utilisateurs pour Keystone
Connectez-vous à MySQL :
```bash
sudo mysql -u root -p
```

Exécutez ces commandes dans le shell MySQL :
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
FLUSH PRIVILEGES;
EXIT;
```

Remplacez `KEYSTONE_DBPASS` par le mot de passe que vous souhaitez utiliser.

### 5. Maintenant vous pouvez exécuter la synchronisation de la base Keystone
Comme indiqué dans votre tutoriel :
```bash
sudo keystone-manage db_sync
```

### Vérification
Pour vérifier que les tables ont bien été créées :
```bash
sudo mysql -u keystone -p -e "USE keystone; SHOW TABLES;"
```

### Notes importantes :
1. Assurez-vous que le mot de passe (`KEYSTONE_DBPASS`) correspond à celui dans votre fichier `/etc/keystone/keystone.conf`
2. Si vous avez des erreurs de connexion, vérifiez :
   - Que MySQL écoute sur la bonne adresse (`netstat -tulnp | grep mysql`)
   - Que l'utilisateur a bien les privilèges (`SHOW GRANTS FOR 'keystone'@'%';`)
3. Pour les installations de production, utilisez des mots de passe plus complexes

Après ces étapes, vous pouvez continuer avec le reste de la configuration de Keystone comme indiqué dans votre tutoriel.


```bash
eureka@DESKTOP-40LBQG8:~$ sudo nano /etc/mysql/mariadb.conf.d/99-openstack.cnf
eureka@DESKTOP-40LBQG8:~$ sudo systemctl restart mysql
eureka@DESKTOP-40LBQG8:~$ sudo mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] Y
Enabled successfully!
Reloading privilege tables..
 ... Success!


You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] n
 ... skipping.

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] n
 ... skipping.

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] n
 ... skipping.

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
eureka@DESKTOP-40LBQG8:~$ sudo mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 34
Server version: 10.11.13-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.003 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keyst
one_db_pwd';
Query OK, 0 rows affected (0.012 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone_db_p
wd';
Query OK, 0 rows affected (0.031 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> EXIT;
Bye

```