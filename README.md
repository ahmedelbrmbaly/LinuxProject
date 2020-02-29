# Linux Server Configuration
## General information
**Public IP :** `18.196.108.191`
**ssh port :** 2200
**Host Name** http://ec2-18-196-108-191.eu-central-1.compute.amazonaws.com



## General Configurations
**1. Static Ip**
* Amazon Lightsail
* Networking
* Create static IP
* Create
* Static IP is `3.126.114.252`


**2. Open ports (123, UDP) and (2200, TCP)**
   * Amazon Lightsail
   * click on instance
   * Networking
   * Firewall
   * Add another
   * (123, UDP) and (2200, TCP)
   * Save

**3. Connect via ssh using private key**
* Amazon Lightsail
* click on instance
* Account page
* ssh keys
* Download
* Save it on my local computer as `aws.pem` in path `~/.ssh/aws.pem`
* secure the key `chmod 600 ~/.ssh/aws.pem`
* From my local terminal `ssh -i ~/.ssh/aws.pem ubuntu@3.126.114.252`

**4. Update machine & Install `finger`**
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install finger
```
**5. Grader User**
* Add grader user `sudo adduser grader`
* Give it sudo permission
  * `sudo nano /etc/sudoers.d/grader`
  * write in the file `grader ALL=(ALL:ALL) ALL`
  * Save

**6. Setup key login**
* From local terminal ` ssh-keygen -f ~/.ssh/udacity_key.rsa`
* Read the public key   `cat ~/.ssh/udacity_key.rsa.pub`
* Copy it
* Using shell that is connected to server
```
cd /home/grader
sudo mkdir .ssh
sudo touch .ssh/authorized_keys
sudo nano .ssh/authorized_keys
```
* Put the public key in the file and save it
* Change permissions
```
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
```
* Change .ssh owner to grader user
 `sudo chown -R grader:grader /home/grader/.ssh`
* Restart ssh service so changes take place
`sudo service ssh restart`

**7. Connect as grader**
* open local terminal
* `ssh -i ~/.ssh/udacity_key.rsa -p22 grader@3.126.114.252`


**8.Enforce key-based SSH authentication**
* `sudo nano /etc/ssh/sshd_config`
* channge `PasswordAuthentication` to `no`
* `sudo service ssh restart`

**9. Change ssh from port 22 to 2200**
* `sudo nano /etc/ssh/sshd_config`
*  chang `port 22` to `port2200`
* `sudo service ssh restart`
* Make sure that is was changed `sudo netstat -tulnp | grep ssh`
* connect through port 2200
`ssh -i ~/.ssh/udacity_key.rsa -p2200 grader@18.196.108.191`

**10. Connfigure ufw Firewall**
* check status `sudo ufw status`
* block all in-coming `sudo ufw default deny incoming`
* allow all out-coming `sudo ufw default allow outgoing`
* allow our ssh port `sudo ufw allow 2200/tcp`
* allow HTTP port `sudo ufw allow 80/tcp`
* allow NTP port `sudo ufw allow 123/udp`
* enable firewall `sudo ufw enable`
* check status `sudo ufw status`
```
Status: active
To                         Action      From
                         ------      ----
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123/udp                    ALLOW       Anywhere                  
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123/udp (v6)               ALLOW       Anywhere (v6)     
```

**11. Disable Root remote login**
* `sudo nano /etc/ssh/sshd_config`
* change `PermitRootLogin` to `no`
* `sudo service ssh restart`

## Web Server Configurations

**1. Required Installs**
* Install `apache2` Server
`sudo apt-get install apache2`
* Install `mod_wsgi` module that provides a WSGI compliant interface for hosting Python based web applications under Apache.
`sudo apt-get install libapache2-mod-wsgi`
* Install `python-dev` package
`sudo apt-get install python-dev`

**2. Enable mod_wsgi**
* `sudo a2enmod wsgi`
* edit `etc/apache2/sites-enabled/000-default.conf`
  * ` sudo nano /etc/apache2/sites-enabled/000-default.conf `
  * before `</VirtualHost>` line
  * add `WSGIScriptAlias / /var/www/html/myapp.wsgi`
  * save file
* Restart server
`sudo service apache2 restart`



**3. Get Item Catalog project**
* Install git `sudo apt-get install git`
* `cd /var/www`
* Make new directory `sudo mkdir catalog`
* Make `grader` user the owner
`sudo chown -R grader:grader catalog`
* `cd catalog`
* Clone `Item Catalog` project
`sudo git clone https://github.com/ahmedelbrmbaly/catalog`


**4. `.wsgi` file**
* `cd /var/www/catalog/`
* Make new file `sudo nano catalog.wsgi`
* add this
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")
from catalog import app as application
application.secret_key = 'your_secret_key'
```

**5. Install  virtual environment**
```
sudo apt-get install python-pip
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo chmod -R 777 venv
```


**6. Install `flask` and other packages**
* `sudo apt-get install python-pip`
* `sudo pip install Flask`
* `sudo pip install sqlalchemy`
* `sudo pip install httplib2`
* `sudo pip install requests`
* `sudo pip install  oauth2client`
* `sudo pip install psycopg2 `
* `sudo pip install SQLAlchemy-Utils `

**7. changes in `__init__.py`**
* `CLIENT_ID = json.loads( open('client_secrets.json', 'r').read())['web']['client_id']`
to
`CLIENT_ID = json.loads( open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']`
* `app.debug = True app.run(0.0.0.0, port=5000)`
to
`app.run()`

**8. Configure   virtual host**
* `sudo nano /etc/apache2/sites-available/catalog.conf`
* find my host name in this link: http://www.hcidata.info/host2ip.cgi
* Host name is
`ec2-18-196-108-191.eu-central-1.compute.amazonaws.com`
* Paste the following
```
<VirtualHost *:80>
    ServerName 18.196.108.191
    ServerAlias ec2-18-196-108-191.eu-central-1.compute.amazonaws.com
    ServerAdmin admin@18.196.108.191
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-$
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


**8. Install and configure PostgreSQL**

 * Install
 `sudo apt-get install libpq-dev python-dev`
 `sudo apt-get install postgresql postgresql-contrib`
 * `sudo -u postgres -i`
 * `psql`
 *
 ```
CREATE USER catalog WITH PASSWORD 'ahmed';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
 \c catalog
 REVOKE ALL ON SCHEMA public FROM public;
 GRANT ALL ON SCHEMA public TO catalog;
 \c
 exit
 ```
 * using `sudo nano` command  change all engines -in three files (`__init__.py`, `database_setup.py`, `items.py` )- to
` engine = create_engine('postgresql://catalog:ahmed@localhost/catalog)`

* Initiate the database `python database_setup.py`
* Add items `python items.py`
* `sudo service apache2 restart`

**9. Change default `apache2` page**
* `sudo a2dissite 000-default.conf`
* `sudo a2ensite catalog`
* `sudo service apache2 restart`

**10. Google sign in edits**
* add toAuthorized JavaScript origins and  Authorized redirect URIs in Credentials Setting   
  * the ip `http://18.196.108.191/.xio.io`
  * the hosy url `http://ec2-18-196-108-191.eu-central-1.compute.amazonaws.com`
* Download the new `client_secrets.json`



## Summary of installed software
**1. finger** `sudo apt-get install finger`
**2. apache2** `sudo apt-get install apache2`
**3. mod_wsgi** `sudo apt-get install libapache2-mod-wsgi`
**4. python-dev** `sudo apt-get install python-dev`
**5. git** `sudo apt-get install git`
**6. python-pip** `sudo apt-get install python-pip`
**7. Flask** `sudo pip install Flask`
**8. sqlalchemy** `sudo pip install sqlalchemy`
**9. httplib2** `sudo pip install httplib2`
**10. requests** `sudo pip install requests`
**11. oauth2client** `sudo pip install  oauth2client`
**12. psycopg2** `sudo pip install psycopg2 `
**13. SQLAlchemy-Utils** `sudo pip install SQLAlchemy-Utils`
**14. httplib2** `sudo pip install httplib2`








## Resources
* change port   [link 1](https://forums.aws.amazon.com/thread.jspa?messageID=921929) and [link 2](https://www.ubuntu18.com/ubuntu-change-ssh-port/)
* https://github.com/jungleBadger/-nanodegree-linux-server-troubleshoot/tree/master/OAuth_login
* https://stackoverflow.com/questions/36020374/google-permission-denied-to-generate-login-hint-for-target-domain-not-on-localh
* https://stackoverflow.com/questions/14238665/can-a-public-ip-address-be-used-as-google-oauth-redirect-uri
* https://github.com/jungleBadger/-nanodegree-linux-server-troubleshoot/tree/master/python3%2Bvenv%2Bwsgi
* https://github.com/juvers/Linux-Configuration
* https://github.com/chuanqin3/udacity-linux-configuration
* https://github.com/twhetzel/ud299-nd-linux-server-configuration
* https://github.com/boisalai/udacity-linux-server-configuration
* https://github.com/chuanqin3/udacity-linux-configuration
* https://github.com/twhetzel/ud299-nd-linux-server-configuration
* https://github.com/boisalai/udacity-linux-server-configuration
