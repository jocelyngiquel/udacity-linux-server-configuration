# LINUX SERVER CONFIGURATION
Configuring a Linux server to host a web app securely.

# Server details
Based on Amazon Lightsail service, the server is linux based and runs Ubuntu 16.04 LTS - Xenial (HVM)
Application URL: --- (the service has been stopped)

# Configuration steps
### Launching the server on AWS Lightsail
Connect to https://lightsail.aws.amazon.com/ and follow the instruction to launch a linux based, OS only, Ubuntu instance.
Download the SSH key (.pem) provided under your account page and store it locally under ```/user/.ssh```

### Connect to your instance using your local terminal
Open a terminal on your machine and connect with the following command
```ssh -i ~/.ssh/LightsailDefaultPrivateKey-eu-central-1.pem ubuntu@18.184.157.67```
### Update and upgrade packages
To update and upgrade the various packages on the instance run simultaneously the 2 following commands
```sudo apt-get update```
```sudo apt-get upgrade```
Then run ```sudo apt-get dist-upgrade``` to install packages kept back

### User creation and SSH connection
Create a new user. Provide a password and details when prompted
```sudo adduser grader```
Grant sudo rights to the user grader
```sudo nano /etc/sudoers.d/grader```
And add in the file
```grader ALL=(ALL:ALL) ALL```

Now create a new key pair using ```ssh-keygen``` in another terminal, on *your own machine*.
Save the key pair under ```/user/.ssh```
Open the ```.pub``` file and copy its content

Now back on the terminal that is connected to your instance
```mkdir /home/grader/.ssh```
```sudo nano /home/grader/.ssh/authorized_keys```
Paste the key, save and exit the file
Now change the owner and permissions for following folder and file.
```chown grader:grader /home/grader/.ssh```
```chmod 700 /home/grader/.ssh```
```chown grader:grader /home/grader/.ssh/authorized_keys```
```chmod 644 /home/grader/.ssh/authorized_keys```

You should be now able to connect to your instance using
```ssh -i ~/.ssh/THE_NAME_OF_YOUR_KEY grader@18.196.194.53 -p 22```

### Configure Uncomplicated Firewall (UFW)
Allow incoming connection for SSH on port 2200:
```sudo ufw allow 2200/tcp```
Allow incoming connections for HTTP on port 80:
```sudo ufw allow www```
Allow incoming connection for NTP on port 123:
```sudo ufw allow ntp```

To check the rules that have been added before enabling the firewall use:
```sudo ufw show added```
To enable the firewall, use:
```sudo ufw enable```
To check the status of the firewall, use:
```sudo ufw status```

If the port 22/tcp is still allowed, run
```sudo ufw deny 22/tcp```
Then rerun ```sudo ufw status``` for a last check

### Switch connection port from 22 to 2200, and remove root access
Change the port 22 to 2200 by amending the ```sshd_config``` file
```sudo nano /etc/ssh/sshd_config``
In addition, change the ```PasswordAuthentication``` to ```no```.
Change ```PermitRootLogin``` to ```no```

Then restart the SSH service ```sudo service ssh restart```

Now you will to log using ```ssh -i ~/.ssh/THE_NAME_OF_YOUR_KEY grader@18.196.194.53 -p 2200```

### Complete the UFW configuration
We need to complete the firewall configuration. By default, block all incoming connections on all ports:
```sudo ufw default deny incoming```
And allow outgoing connection on all ports:
```sudo ufw default allow outgoing```

### Change the time zone to UTC
use the following command
```sudo timedatectl set-timezone UTC```

### Install Apache, mod_wsgi and git
Install Apache
```sudo apt-get install apache2```
Install mod_wsgi
```sudo apt-get install libapache2-mod-wsgi python-dev```
Enable mod_wsgi and start apache
```sudo a2enmod wsgi```
```sudo service apache2 start```
Then, install git
```sudo apt-get install git```

### Create virtual environment and install required Python packages
We will start by creating the hosting folder
```cd /var/www/```
```sudo mkdir catalog```
```chown -R grader:grader catalog```
Then clone the app catalog git repository into the newly created folder
```git clone https://github.com/jocelyngiquel/udacity-item-catalog.git```
Now the trick is that this repository has been saved to run on a local machine with a Vagrant VM. The target folder (catalog) is insid the vagrant one.
use the mv and rm commands to arrive to the following structure:
```
-var
--www
---catalog
----catalog
-----project.py
-----...
```

Install pip and virtualenv
```sudo apt-get install python-pip```
```sudo pip install virtualenv```
Set up the virtual environment
```sudo virtualenv venv```
```source venv/bin/activate```
```sudo chmod -R 777 venv```

Install the following python dependencies using the ```sudo pip install``` command
- flask
- oauth2client
- sqlalchemy
- requests
- httplib2
- psycopg2

### Configure Apache and mod_wsgi
Configure and enable a new vitural host
```sudo nano /etc/apache2/sites-available/catalog.conf```
and add the following content
```
<VirtualHost *:80>
    ServerAlias --- the server URL
    ServerAdmin grader@18.196.194.53
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

Create and configure the ```.wsgi``` file
```cd /var/www/catalog```
```sudo nano catalog.wsgi```

And add the following content
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

### Install and configure PostgreSQL
First of all, lets install some packages to work with PostgreSQL via python
```sudo apt-get install libpq-dev python-dev```
Then, let's install PostgreSQL
```sudo apt-get install postgresql postgresql-contrib```

Now we need to configure a new database in PostgreSQL to serve the catalog application
Let's access PostgreSQL using the default postgres user
```sudo -u postgres psql```
We are now in PostgreSQL, notice the # sign. For the following steps, do not forget the ```;``` at the end of the lines.
Create a new user called 'catalog' with his password
```CREATE USER catalog WITH PASSWORD 'catalog';```
Give catalog user the CREATEDB permission
```ALTER USER catalog CREATEDB;```
Create the 'catalog' database owned by catalog user
```CREATE DATABASE catalog WITH OWNER catalog;```
Connect to the database ```\c catalog```
Revoke all the rights
```REVOKE ALL ON SCHEMA public FROM public;```
Lock down the permissions to only let catalog role create tables
```GRANT ALL ON SCHEMA public TO catalog;```
Log out from PostgreSQL ```\q```

### Update application files
As mentioned above, the git repository you cloned was not designed to run on a Ubuntu server. In order to run there is a couple of change to do on the code.
1. No client_secrets.json is provided on github. In order to run, the user need to set up the google Oauth with http://console.developers.google.com/. From the credentials manager, after having set up the URL, save the client_secrets.json file under /var/www/catalog/catalog. The name and the destination path are important. 
2. Change ```project.py``` into ```__init__.py```
3. update the absolute path for client_secrets.json in ```__init__.py```
4. Paste the client_id from the client_secrets.json into /templates/login.html, at the place of PLACE_YOUR_GOOGLE_CLIENT_ID_HERE
5. Change the port from 8000 to 80 in ```__init__.py```
6. In ```models.py```, ```legodataloader.py``` and ```__init__.py```, change ```engine = create_engine('sqlite:///category.db')``` to ```engine = create_engine('postgresql://catalog:catalog@localhost/catalog')```
7. in ```__init__.py```, remove the SQLite specific arguments ```connect_args``` and ```poolclass```

### Update Oauth
Into the Google Developers Console -> API Manager -> Credentials, in the web client under "Authorized JavaScript origins", update the URL

### One last step
If the virtual environment is deactivated, activate it again
```cd /var/www/catalog/catalog```
```source venv/bin/activate```
and run ```models.py``` and ```legodataloader.py``` to set the tables and add some starting data
```python models.py```
```python legodataloader.py```

Disable the default virtual host with:
```sudo a2dissite 000-default.conf```
Then enable the catalog app virtual host:
```sudo a2ensite catalog.conf```
To make these Apache2 configuration changes live, reload Apache:
```sudo service apache reload```

# Resources and help
https://libraries.io/github/golgtwins/Udacity-P7-Linux-Server-Configuration
https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
https://www.dabapps.com/blog/introduction-to-pip-and-virtualenv-python/
https://serverfault.com/questions/265410/ubuntu-server-message-says-packages-can-be-updated-but-apt-get-does-not-update
https://knowledge.udacity.com/questions/7685
https://knowledge.udacity.com/questions/858
https://knowledge.udacity.com/questions/1009










