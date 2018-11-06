# linux-server

Website: [18.234.248.46](http://18.234.248.46/)

## Server Stand-up

1. Start a new Ubuntu Linux server instance on Amazon Lightsail

2. SSH to the server with the following
```ssh ubuntu@18.234.248.46```

## Secure the Server

1. Disable root login by adding the following in ```/etc/ssh/sshd_config```
```
PermitRootLogin no
```

2. Ensure key-based ssh authentication is enforced by modifying ```/etc/ssh/sshd_config```
```
RSAAuthentication yes
PasswordAuthentication no
```

3. Update all currently installed packages
```
sudo apt-get update
sudo apt-get upgrade
```

4. Change the SSH port from 22 to 2200 in ```/etc/ssh/sshd_config```
Change `Port 22` to `Port 2200`
```
sudo service ssh restart
```
Test with SSH key using the new port
```
ssh ubuntu@18.234.248.46 -i udacity_rsa -p 2200
```

5. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

### Create User

1. Create a new user account named grader
```
sudo adduser grader 
(password: grader)
```

2. Give grader the permission to sudo
Create file ```/etc/sudoers.d/grader``` with following content
```
grader ALL=(ALL) NOPASSWD:ALL
```

3. Create an SSH key pair for grader using the ssh-keygen tool
Run ```ssh-keygen``` on your local machine and provide in the information

4. Create ```authorized_keys``` file under grader home directory
```
su grader
cd
sudo vim authorized_keys
```
Copy and paste public key in the file

### Prepare to deploy your project

1. Configure the local timezone to UTC
```sudo dpkg-reconfigure tzdata```, select UTC

2. Install and configure Apache to serve a Python mod_wsgi application
```
sudo apt install apache2
sudo apt install libapache2-mod-wsgi
```

3. Install and configure PostgreSQL
```
sudo apt install postgresql
```

Do not allow remote connections by checking
```
sudo vim /etc/postgresql/9.5/main/pg_hba.conf
```


4. Create a new database user named catalog
```
sudo su - postgres
psql
create database catalog;
create user catalog with password 'password';
grant all privileges on database catalog to catalog;
\q or Ctrl + D to quit postgresql
exit to logout of postgres user
```

5. Install git
```
sudo apt install git
```

## Deploy the Item Catalog project

1. Clone Item Catalog project from the Github repository and rename the directory
```
cd /var/www
sudo mkdir catalog

git clone https://github.com/wyuen524/item-catalog.git
mv item-catalog catalog
```

2. Modify application.py, database_setup.py, and load_db.py to use postgres db
```
From:
engine = create_engine('sqlite:///fortniteitems.db')

To:
engine = create_engine('postgresql://catalog:password@localhost/catalog')
```

3. Install prereqs
```
sudo apt-get install python-pip
sudo pip install Flask
sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests
```

### Setup Flask App

1. Create the .wsgi file with the following contents
```
sudo vim catalog.wsgi

import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

2. Configure a new virtual host by creating the file with the following contents
```
sudo vim /etc/apache2/sites-available/catalog.conf

<VirtualHost *:80>
    ServerName 18.234.248.46
    ServerAdmin ubuntu@18.234.248.46
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

3. Update filepath of ```client_secrets.json``` in ```application.py```
```
/var/www/catalog/item-catalog/client_secrets.json
```

4. Rename application.py

```
mv application.py __init__.py
```

5. Load database with items

```
sudo python load_db.py
```

6. Enable the virtual host
```
sudo a2ensite catalog
```

7. Restart Apache
```
sudo service apache2 restart
```

The file structure for catalog should look like this:
```
catalog
   ├── catalog
   │   ├── client_secrets.json
   │   ├── database_setup.py
   │   ├── __init__.py
   │   ├── load_db.py
   │   ├── README.md
   │   ├── static
   │   │   ├── background2.jpg
   │   │   ├── background.jpg
   │   │   ├── Burbank Big Condensed Black.ttf
   │   │   └── styles.css
   │   └── templates
   │       ├── deleteCategory.html
   │       ├── deleteItem.html
   │       ├── editCategory.html
   │       ├── editItemInfo.html
   │       ├── error.html
   │       ├── index.html
   │       ├── login.html
   │       ├── newCategory.html
   │       ├── newItem.html
   │       └── weapon_templ.html
   └── catalog.wsgi

```

Thanks to [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps) for the guide