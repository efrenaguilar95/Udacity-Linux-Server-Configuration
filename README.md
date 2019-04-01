# Linux Server Configuration Project

This project is a deployment of a Flask application using Apache on a Linux machine. The project that is being deployed can be found [here](https://github.com/efrenaguilar95/Udacity-Item-Catalog). For this project I used a virtual Linux machine (Ubuntu 16) hosed by Amazon Lightsail.

This project showcases my ability to configure a Linux machine to host an Apache web server from the ground up. It also shows my ability to configure said server to be protected and secure against many different threats that come along with having a public website.

## Getting Started
### Server Info
IP Address: 54.245.168.69

SSH Port: 2200

SSH Key: Given to project grader, is unique ror every user

Application Link: [54.245.168.69.xip.io](54.245.168.69.xip.io)

## Initial Security Configuration
I first edited the SSH port on my instance from 22 to 2200 by changing the sshd_config file
```
sudo vim /etc/ssh/sshd_config
```
I then configured my firewall, but first I checked it was offline with
```
sudo ufw status
```
Then I blocked all incoming connections and allowed all outgoing connections
```
sudo ufw defualt deny incoming
sudo ufw default allow outgoing
```
I then configured my firewall to allow HTTP on port 80, NTP on port 123, and SSH on port 2200
```
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
```
I then double checked my work, turned on the firewall and applied my changes to sshd_config
```
sudo ufw show added
sudo ufw enable
sudo ufw show status
sudo service sshd restart
```

## Giving 'grader' user ssh access
I first created a new user named 'grader'
```
sudo adduser grader
```
I then edit the sudoers file to grant the grader user sudo access. I use the visudo command to safely do this
```
sudo visudo -f /etc/sudoers.d/grader
```
Adding this line
```
grader ALL=(ALL:ALL) NOPASSWD:ALL
```
I then went on my local machine and generated an ssh key. I named this key 'grader'
```
ssh-keygen
```
I copied the contents of "grader.pub" and went back to my linux machine where I switched to the grader user
```
sudo su - grader
```
As this user, I created a subdirectory called ".ssh" and set its permissions to only allow the grader to read write or execute
```
mkdir .ssh
chown grader:grader /home/grader/.ssh
chmod 700 /home/grader/.ssh
```
In this new directory I created a file called 'authorized_keys' and pasted in the contents of 'grader.pub'. I then set the permissions of the file to 400
```
sudo vim .ssh/authorized_keys
sudo chmod 400 .ssh/authorized_keys
```
Lastly, I disabled root login and forced key authentication by changing the file '/etc/ssh/sshd_config'. Here I changed "PermitRootLogin without-password" to "PermitRootLogin no" and uncommented the line that reads "PasswordAuthentication no" and ran "sudo service sshd restart"

I then double checked that I could login in as grader with this command
```
ssh -i grader.pem grader@54.245.168.69 -p 2200
```

## Preparing for Deployment
There were a few steps I had to do before I could deploy the Flask app.
The first was to install any packages I needed.
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install apache2 libapache2-mod-wsgi-py3
sudo apt-get install build-essential python3-dev python3-setuptools
sudo apt-get install postgresql
sudo apt-get install python3-pip
sudo apt-get install git
```
I then went into the postgresql command line to create a new database for my Flask app. I also made a new user that would have permission to edit the database.
```
sudo su - postgres
postgres $ psql;
postgres=# CREATE DATABASE itemcatalog;
postgres=# CREATE USER efren;
postgres=# ALTER ROLE efren WITH PASSWORD 'xxxxxxxxxxxxxxx';
postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO efren;
```

## Deploying the Flask App
I first run sudo a2enmod wsgi to enable the mod_wsgi
```
sudo a2enmod wsgi
```
I then go into the /var/www directory and create a ItemCatalogApp folder. I clone my repository in here, and rename it ItemCatalogApp as well. I also delete the .git folder. And rename my 'application.py' to '\__init__.py'
```
cd /var/www
sudo mkdir ItemCatalogApp
cd ItemCatalogApp
git clone https://github.com/efrenaguilar95/Udacity-Item-Catalog.git
sudo mv Udacity-Item-Catalog ItemCatalogApp
cd ItemCatalogApp
sudo mv application.py __init__.py
sudo rm -rf .git
```
I then created a virtual python environment to run this app and installed all of the needed packages
```
sudo python3 -m virtualenv -p python3 venv
sudo chown -R username:username virtualenv
source vevn/bin/activate
pip3 install sqlalchemy
pip3 install psycopg2
pip3 install flask
pip3 install oauth2client
pip3 install requests
deactivate
```
Once this was done, I created the config file for the application
```
sudo vim /etc/apache2/sites-available/ItemCatalogApp.conf
```
and added the following lines:
```
<VirtualHost *:80>
                ServerName 54.245.168.69
                ServerAdmin xxxxxxxx@xxxxx
                ServerAlias 54.245.168.69*

                WSGIDaemonProcess ItemCatalogApp python-home=/var/www/ItemCatalogApp/venv
                WSGIProcessGroup ItemCatalogApp
                WSGIApplicationGroup %{GLOBAL}
                WSGIScriptAlias / /var/www/ItemCatalogApp/application.wsgi
                <Directory /var/www/ItemCatalogApp/ItemCatalogApp>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/ItemCatalogApp/ItemCatalogApp/static
                <Directory /var/www/ItemCatalogApp/ItemCatalogApp/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Lastly I created the file 'catalog.wsgi'
```
sudo vim /var/www/ItemCatalogApp/catalog.wsgi
```
and inserted the following lines
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/ItemCatalogApp/")

from ItemCatalogApp import app as application
application.secret_key = 'XXXXXXXXXXXXXXX'
```
Lastly, I restarted Apache, lauching my server
```
sudo service apache2 restart
```
## Acknowledgements
Both of these resources were an immense help in completing this.

* [This repo by Dakota Michael](https://github.com/kotamichael/amazon-lightsail-server-configuration/blob/master/README.md)

* [Digital Ocean's Guide to deploying a Flask App](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
