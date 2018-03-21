# Criteria

## User Management
* Can you log into the server as the user grader using the submitted key?
* Is remote login of the root user disabled?
* Is the grader user given sudo access?

## Security
* Is the firewall configured to only allow for SSH, HTTP, and NTP?
* Are users required to authenticate using RSA keys?
* Are the applications up-to-date?
* Is SSH hosted on non-default port?

## Application Functionality
* Is there a web server running on port 80?
* Has the database server been configured to properly serve data?
* Has the web server been configured to serve the Item Catalog application?

# Information
* Public IP Address: 18.188.90.162
* SSH port: 2200
* Web URL: 18.188.90.162

# Summary

## Creating an instance on Lightsail
The first step was to create an instance on Amazon Lightsail, selecting the following options:

* Linux/Unix
* (OS Only) Ubuntu

![Linux](/images/1_instance.png)
![OS](/images/2_instance.png)

## SSH Keys

![SSH_keys](/images/3_ssh.png)

Once I got the instance running, I downloaded my SSH keys and changed their permissions: 

```chmod 400 {{keyname}}```

After that I put the key into my working directory and was able to login via my terminal: 

```ssh {{username}}@18.188.90.162 -i {{keyname}}```

## Updating the Software

Once I logged into the server, I updated all of the software - first running:

```sudo apt-get update```

Then

```sudo apt-get upgrade```

## Configuring Time to UTC

```sudo dpkg-reconfigure tzdata```

[Setting Timezone From Termina](https://askubuntu.com/questions/323131/setting-timezone-from-terminal)

I then chose 'None Of The Above' and 'UTC'

## Checking Python

I first check to see if Python3 was installed (it was). Otherwise, I would have had to download it using:

```sudo apt-get install python3```

I also created a symbolic link to Python3 by executing:

```sudo ln -s /usr/bin/python3 /usr/bin/python```

## Installing Pip

In order to get the necessary python packages, Pip was installed.

```sudo apt-get install python3-pip```

## Installing Apache

```sudo apt-get install apache2```

I also installed WSGI PY3 as my project uses Python3

```sudo apt-get install libapache2-mod-wsgi-py3```

Next:

```sudo a2enmod wsgi```

To enable WSGI, though it was already enabled.

## Installing Git

Git was already installed on my server, so there was no additional steps here. If I had to install it, I would have ran:

```sudo apt-get install git```

## Cloning My Project

The next step was to clone my project. I changed my directory to /var/www/ and cloned my project into that directory:

```sudo git clone https://github.com/drownedout/udacity_project_4```

I then renamed the directory to 'ItemCatalog'

```sudo mv udacity_project_4 ItemCatalog```

## Downloading Project Libraries

```sudo pip3 install flask flask_assets psycopg2 oauth2client requests httplib2```

This caused me a few headaches as I later had to use the ```-H``` flag with pip to ensure that my server was looking for the libraries in the correct place.

There were also some issues with permissions when running the server - specifically with the static assets. It was fixed by letting www-data be the owner of the static folder. However, this isn't the best practice and can lead to security vulnerabilities.

## Installing Postgres

```sudo apt-get install postgresql```

Once Postgres was installed, I then created the database for my project.

```sudo su - postgres```
```sql```
```CREATE USER admin WITH PASSWORD 'password'```
```ALTER USER admin CREATEDB```
```CREATE DATABASE categoryitem with OWNER admin```

After that, I connected to my database and altered the permissions:

```REVOKE ALL ON SCHEMA public FROM public;```
```GRANT ALL ON SCHEMA public TO admin;```

The next step was to alter all of my python files used to connect to the database to the following:

```engine = create_engine(postgresql://admin:password@localhost/categoryitem)```

[Postgres Setup on Ubuntu](https://wixelhq.com/blog/how-to-install-postgresql-on-ubuntu-remote-access)

## Configuring the WSGI file

I then created the .wsgi file in my project's root directory with the following:

```import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/ItemCatalog")
from __init__ import app as application```

## Apache Virtual Host Configuration

Once the WSGI file was created, I made a virtual host .conf file.

```cd /etc/apache2/sites-available/```

```sudo touch itemCatalog.conf```

My configuration for the .conf file:

```<VirtualHost *:80>
     ServerName 18.188.90.162
     ServerAdmin admin@18.188.90.162
     #Location of the items-catalog WSGI file
     WSGIScriptAlias /  /var/www/ItemCatalog/itemCatalog.wsgi process-group=ItemCatalog
     WSGIDaemonProcess ItemCatalog processes=5 user=www-data group=www-data python-path=/home/ubuntu/.local/lib/python3.5/site-packages threads=1
     WSGIProcessGroup ItemCatalog
     #Allow Apache to serve the WSGI app from our catalog directory
     <Directory /var/www/ItemCatalog>
          WSGIApplicationGroup %{GLOBAL}
          WSGIScriptReloading On
          Order allow,deny
          Allow from all
     </Directory>
     #Allow Apache to deploy static content
     <Directory /var/www/ItemCatalog/static>
        Order allow,deny
        Allow from all
     </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>```

Restarted the server:

```sudo service apache2 restart```

[MOD WSGI](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/)
[MOD WSGI Configuration](https://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html)

## Adding a new user

Next, I added a new user 'grader' by running ```sudo adduser grader``` and set the password to (wait for it), `password`.

The user 'grader' was then given sudo privileges by running ```sudo visudo``` and then changing the file by adding ```grader ALL=(ALL:ALL) ALL``` under the root user under ```#User privilege specification```

[How to add, delete sudo privileges](https://www.digitalocean.com/community/tutorials/how-to-add-delete-and-grant-sudo-privileges-to-users-on-a-debian-vps)

## Changing the default SSH

After that, I changed the default SSH port by first going into the AWS console and adding a custom port 2200.

![SSH_port](/images/4_portchange.png)

Then, I went into my sshd config file using ```sudo nano /etc/ssh/sshd_config``` and adding port 2200 under ```What ports, IPs and protocols we list for``` while commenting out port 22. I also disabled root login by changing ```PermitRootLogin``` to "no"

Finally, I restarted the SSH service and logged back in via my terminal:

```ssh {{user}}@18.188.90.162 -i {{keyname}} -p 2200```

[How to set up SSH](https://lightsail.aws.amazon.com/ls/docs/how-to/article/lightsail-how-to-set-up-ssh)

## Configuring the Firewall

To configure the firewall, I first checked the status of the firewall.

```sudo ufw status```

The firewall was disabled, as expected. The following was done to update the firewall permissions:

```sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
```

[Firewalls](https://docs.bitnami.com/aws/faq/operating-servers-instances/open_firewall/)

## Generating SSH Keys

Finally I generated the SSH keys for the user, 'grader'.

On my local machine, I generated a pair of keys:
```ssh-keygen -f ~/.ssh/graderKeys```

Once I generated the keys, I created new directory inside of the grader's profile:

```cd /home/grader/```
```sudo mkdir .ssh```
```cd .ssh```
```sudo nano authorized_keys```

I then copied the public key's contents into the file and saved.

[Set up SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
[Configuring SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)