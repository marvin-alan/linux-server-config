#Linux-server-configuration

### Project Description

A Linux distribution on a virtual machine and prepare to host a web application, includes installing updates, securing it from a number of attack vectors and installing/configuring database server.

- IP address: 34.222.140.232

- Accessible SSH port: 2200

- URL: http://34.222.140.232.xip.io

### Walkthrough

1. Create new user named grader and give it the permission to sudo
  - `sudo adduser grader` to create a new user named grader
  - Create a new file in the sudoers directory `sudo vi /etc/sudoers.d/grader`
  - Paste this code:
  ```
  grader ALL=(ALL:ALL) NOPASSWD:ALL
  ```

2. Set ssh login using keys

  - `su - grader`
  - `mkdir .ssh`
  - `touch .ssh/authorized_keys`
  - `vi .ssh/authorized_keys`

  Copy the public key (one with the extension .pub) generated on your local machine to this file and save

  - `chmod 700 .ssh`
  - `chmod 644 .ssh/authorized_keys`
  - Restart SSH using `service ssh restart`

3. Update all currently installed packages
  - Download package lists with `sudo apt-get update`
  - Fetch new versions of packages with `sudo apt-get upgrade`

4. Change SSH port from 22 to 2200
  - Run `sudo nano /etc/ssh/sshd_config`
  - Change the Port from 22 to 2200
  - Note: Remember to add and save port 2200 with Application as Custom and Protocol as TCP in the Networking section of your instance on Amazon Lightsail.
  
5. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

  - `sudo ufw allow 2200/tcp`
  - `sudo ufw allow 80/tcp`
  - `sudo ufw allow 123/udp`
  - `sudo ufw enable`

6. Configure the local timezone to UTC
  - Run `sudo dpkg-reconfigure tzdata` and then choose UTC
 
7. Disable ssh login for root user
  - Run `sudo nano /etc/ssh/sshd_config`
  - Change `PermitRootLogin without-password` line to `PermitRootLogin no`
  - Restart ssh with `sudo service ssh restart`
  - Now you are only able to login using `ssh grader@34.222.140.232  -p 2200 -i ~/.ssh/grader`
 
8. Install Apache
  - `sudo apt-get install apache2`

9. Install mod_wsgi
  - Run `sudo apt-get install libapache2-mod-wsgi python-dev`
  - Enable mod_wsgi with `sudo a2enmod wsgi`
  - Start the web server with `sudo service apache2 start`

  
10. Clone the Catalog app from Github
  - Install git using: `sudo apt-get install git`
  - `cd /var/www`
  - `sudo mkdir catalog`
  - Change owner of the newly created catalog folder `sudo chown -R grader:grader catalog`
  - `cd /catalog`
  - Clone your project from github `git clone https://github.com/marvin-alan/joke-catalog.git catalog`
  - Rename application.py to __init__.py `mv application.py __init__.py`
  - `sudo vi __init__.py`
  - Change client_secrets.json path to 
  ```
  /var/www/catalog/catalog/client_secrets.json
  ```
 - Also while there change create engine line in your `__init__.py` and `database_setup.py` and `database_init.py` to:
  ``` 
  engine = create_engine('postgresql://catalog:password@localhost/catalog')
  ```

11. Install Flask and other dependencies
  - Install pip with `sudo apt-get install python-pip`
  - Install Flask `pip install Flask`
  - Install other project dependencies `sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils`
  - Note: Installing dependencies sometimes require not using `sudo`

12. Install virtual environment
  - Install the virtual environment `sudo pip install virtualenv`
  - Create a new virtual environment with `sudo virtualenv venv`
  - Activate the virutal environment `source venv/bin/activate`
  - Change permissions `sudo chmod -R 777 venv`

13. Create the .wsgi File
  - Create a catalog.wsgi file, then add this inside under /var/www/catalog::
  - `sudo vi catalog.wsgi`
  - Paste this code: 
  ```
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, "/var/www/catalog/")
  
  from catalog import app as application
  application.secret_key = 'super_secret_key'
  ```
14. Configure and enable a new virtual host
  - Run this: `sudo nano /etc/apache2/sites-available/catalog.conf`
  - Paste this code: 
  ```
  <VirtualHost *:80>
      ServerName 34.222.140.232
      ServerAlias ec2-34-222-140-232.us-west-2.compute.amazonaws.com
      ServerAdmin admin@34.222.140.232
      WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
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
  - Enable the virtual host `sudo a2ensite catalog`
  - Disable the default virtual host: `sudo a2dissite 000-default.conf`

15. Install and configure PostgreSQL
  - `sudo apt-get install libpq-dev python-dev`
  - `sudo apt-get install postgresql postgresql-contrib`
  - `sudo su - postgres`
  - `psql`
  - `CREATE USER catalog WITH PASSWORD 'password';` 
  - Note: The user and password has to match what was used in engine
  - `ALTER USER catalog CREATEDB;`
  - `CREATE DATABASE catalog WITH OWNER catalog;`
  - `\c catalog`
  - `REVOKE ALL ON SCHEMA public FROM public;`
  - `GRANT ALL ON SCHEMA public TO catalog;`
  - `\q`
  - `exit`
  - `python /var/www/catalog/catalog/database_setup.py`
  - Make sure no remote connections to the database are allowed. Check if the contents of this file `sudo nano /etc/postgresql/9.3/main/pg_hba.conf` looks like this:
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```
  
16. Restart Apache 
  - `sudo service apache2 restart`
  
17. Visit site at [http://34.222.140.232.xip.io](http://34.222.140.232.xip.io)

### Sources

[How To Add and Delete Users on an Ubuntu 14.04 VPS](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps)

[How to deploy a flask application on an ubuntu vps](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

[Setup Virtualenv](http://flask.pocoo.org/docs/0.12/installation/)

[How to Run Django with mod_wsgi and Apache with a virtualenv Python environment on a Debian VPS](https://www.digitalocean.com/community/tutorials/how-to-run-django-with-mod_wsgi-and-apache-with-a-virtualenv-python-environment-on-a-debian-vps)

