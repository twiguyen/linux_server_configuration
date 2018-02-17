## Synopsis

The goal of this project is to learn how to install a Linux server and prepare it to host a web application. The web application to be hosted is called "My Wardrobe", a catalog-type application created for Udacity. The server should be configured to be secure, have a database server, and successfully deploying the catalog app.


## 1) Start up a new server

A new Linux server instance was created using [Amazon Lightsail](https://lightsail.aws.amazon.com/)

SSH Port: 2200

Public IP(static): 35.172.18.23

## 2) Update all currently installed packages

```
$ sudo apt-get update
$ sudo apt-get upgrade
```

## 3) Change SSH port and configure Lightsail firewall to allow it.

1. Edit config file
```
$ sudo nano /etc/ssh/sshd_config
```
2. Find 'Port' and change it to 2200
3. Restart
```
$ sudo service ssh restart
```
4. On Lightsail, click under Firewall and add "Custom" port, "2200".

## 4) Configure UFC (Uncomplicated Fire)

This will allow incoming connections for SSH required by the Udacity project.

1. Allow SSH port 2200
```
$ sudo ufw allow 2200/tcp
```
2. Allow HTTP port 80
```
$ sudo ufw allow 80/tcp
```
3. Allow NTP port 123
```
$ sudo ufw allow 2200/udp
```
4).
3. Confirm and enable port settings
```
$ sudo ufw enable
```

## 4) Give 'grader' access
1. Log in into remote VM as root user
```
$ ssh ubuntu@35.172.18.23
```
2. Create a new user named 'grader'
```
$ sudo adduser grader
```
3. Create a file named 'grader' in the sudoers directory
```
$ sudo nano /etc/sudoers.d/grader
```
4. Paste the following and save the file
```
grader ALL=(ALL:ALL) ALL
```
This will allow the grader to have permission to sudo commands.

## 5) Create SSH key pair for 'grader'
1. On the local machine, create a new SSH key pair
```
$ ssh-keygen -f ~/.ssh/udacity_key.rsa
```
3. Log into remove VM as root user
2. Create a file for authorized keys
```
$ touch /home/grader/.ssh/authorized_keys
```
3. Copy file content from 'udacity_key.pub' from your local machine to the 'authorized_keys' file.
4. Grant 'grader' permissions and make them owner
```
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys
$ sudo chown -R grader:grader /home/grader/.ssh
```
5. Log in as grader
```
$ ssh -i ~/.ssh/udacity_key.rsa grader@35.172.18.23
```

## 6) Configure local timezone to UTC
1. Open time configuration
```
$ sudo dpgk-reconfigure tzdata
```
2. Choose 'None of the above'
3. Choose 'UTC'

## 7) Install and configure Apache
1. Install Apache
```
$ sudo apt-get install apache2
```
2. Install Mod_WSGI would will be used to serve the Flash application.
```
$ sudo apt-get install libapache2-mod-wsgi python-dev
sudo service apache2 restart
```
3. Enable mod_wsgi
```
$ sudo a2enmod wsgi
```

## 8) Install and configure PostgreSQL
1. Install Python for PostgreSQL
```
$ sudo apt-get install python-psycopg2
```
2. Install= PostgreSQL
```
$ sudo apt-get install postgresql postgresql-contrib
```
3. Open application's database set up file
```
$ sudo nano database_setup.py
```
4. Change 'sqlite' to 'postgresql'
```
$ pythan engine = create_engine('postgresql://catalog:[PASSWORD]@localhost/catalog')
```
[PASSWORD] will be the database user's password
5. Rename myWardrobe.py
```
$ mv myWardrobe.py __init__.py
```
6. Create new database user
```
$ sudo adduser catalog
```
7. Change to default user 'postgres' and connect
```
$ sudo su - postgres
$ sudo psql
```
7. Create catalog user with password
```
# CREATE USER catalog WITH PASSWORD '[PASSWORD]';
```
8. Allow 'catalog' user to create database tables
```
# ALTER USER catalog CREATEDB;
```
9. Create database
```
# CREATE DATABASE catalog WITH OWNER catalog;
```
10. Connect to database catalog
```
# \c catalog
```
11. Revoke all rights from public
```
# REVOKE ALL ON SCHEMA public FROM public;
```
12. Grant access to 'catalog' user
```
# GRANT ALL ON SCHEMA public TO catalog;
```
12. Exit PostgreSQL and create database schema
```
# \q
$ exit
$ python database_setup.py
```

## 9) Install Git
1. Install git
```
$ sudo apt-get install git
```
2. Configure username (replace [USERNAME] with your choice)
```
$ git config --global user.name [USERNAME]
```
3. Configure email (replace [EMAIL] with your choice)
```
$ git config --global user.email [EMAIL]
```

## 10) Deploy the Catalog Project
1. Create a new catalog directory
```
$ cd /var/www
$ sudo mkdir catalog
```
2. Change owner to grader
```
$ sudo chown -R grader:grader catalog
cd /catalog
```
3. Clone Github catalog repository
```
$ git clone https://github.com/twiguyen/my-wardrobe
```
4. Create .wsgi file
```
$ cd /var/www/catalog
$ sudo nano catalog.wsgi
```
5. Paste the following into file (replace [SECRET_KEY] with your app's secret key)
```
  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0,"/var/www/catalog/")
  
  from catalog import app as application
  application.secret_key = '[SECRET_KEY]'
```
6. Restart Apache
```
$ sudo service apache2 restart
```
7. Install anything neccessory for the project
```
$ sudo apt-get install python-pip
$ pip install Flash
$ pip install bleach
$ pip install httplib2
$ pip install request
$ pip install oauth2client
$ pip install sqlalchemy
$ pip install python-psycopg2
```

## 10) Configure and enable a new virtual host
1. Create new config file:
```
$ sudo nano /etc/apache2/sites-available/catalog.conf
```
2. Paste and save the following:
```
<VirtualHost *:80>
    ServerName 35.172.18.23
    ServerAlias ec2-35-172-18-23.us-east-1.compute.amazonaws.com
    ServerAdmin admin@35.172.18.23
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## 11) Run the application
1. Restart Apache
```
$ sudo service apache2 restart
```
2. Open browser and put in the public ip-address (35.172.18.23)
3. If it doesn't work, check the error log
```
$ sudo tail -20 /var/log/apache2/error.log
```


## Extra Steps

### Removing remote login from root user
1. Configure 'sshd_config' file
```
$ sudo nano /etc/ssh/sshd_config
```
2. Find 'PermitRootLogin' and set to 'no'

### Enforcing key-based authentication
1. Configure 'sshd_config' file
```
$ sudo nano /etc/ssh/sshd_config
```
2. Find 'PasswordAuthentication' and set to 'no'
3. Restart
```
$ sudo service ssh restart
```


## Motivation

This mini project was created for [Udacity](https://www.udacity.com/) and submitted for the **Full 

## Contributors

Jennifer Nguyen  
Udacity  [[Website]](https://www.udacity.com/) 
Udacity Forum Boards [[]](https://discussions.udacity.com/)
Stueken [[GitHub]](https://github.com/stueken/FSND-P5_Linux-Server-Configuration)