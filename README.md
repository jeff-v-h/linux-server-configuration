# Linux Server Configuration with Amazon Lightsail
The following Linux server configuration is completed on a windows machine with a bash terminal.


## IP address and SSH port of server

- Public Static IP address: 35.176.129.195
- SSH port: 2200


## URL to hosted web application
http://ec2-35-176-129-195.eu-west-2.compute.amazonaws.com or
http://35.176.129.195/



## Software installed and configuration changes made
### Software used

- Amazon Web Services Account
- Google Developers Account
- PuTTY 0.70
- Bash Unix Shell/Terminal
- Apache2
- mod_wsgi
- Python2
- PostgreSQL


### Secure your server
#### Updating the instance
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get autoremove
```

#### Configuring the firewall
```
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo allow 222/tcp
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw enable
sudo ufw status
```

In Amazon Lightsail, In the 'jeffh-ubuntu-instance' instance, under the 'Networking' tab, and under the 'Firewall' heading:
- Click 'Add another' to add additional ports
- Open the following 2 ports: TCP 2200 and UDP 123
- Remove SSH 22

#### Connect using your own SSH client (for Linux/Unix-based instances)
1. Ensure an instance is created
2. Create a static IP and attach it to an instance in Amazon Lightsail by following the instructions at https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-create-static-ip
3. Download, install and setup PuTTY following the instructions at https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-set-up-putty-to-connect-using-ssh


### Create `grader`
#### Create grader user and give access to sudo
```
sudo adduser grader  # password = grader, fullname give = udacity grader
sudo usermod -a -G grader  # gives grader access to sudo
sudo -l -U grader  # grader has access to sudo if (ALL: ALL) ALL is seen
```

#### Create SSH key pair for grader using ssh-keygen
On local machine bash terminal:
- `ssh-keygen`
- When asked which file to save the key, copy the listed destination in the brackets but replace the last file (after /.ssh) with your own chosen name (/linuxServerProject was used for this project)
- An empty passphrase was used.
- `cat [file destination the public key has been saved in] # Ends in .pub`
- Copy everything in this file (including ssh-rsa, long encryption and user@lcoation) to be used below

On PuTTY terminal:
```
su - grader  # to switch to grader. Enter password for grader to login
mkdir .ssh
touch .ssh/authorized_keys  # This is where all public keys that this user is allowed to use for auth wil be stored. One key per line in this file
nano .ssh/authorized_keys  # Now paste the copied code from above into this authorized_keys file
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

Back on local bash terminal:
- Login with grader: `ssh grader@[static/public ip of amazon instance] -p 2200 -i ~/.ssh/[file destination of public key]`
	eg. `ssh@grader@35.176.129.195 -p 2200 -i ~/.ssh/linuxServerProject`
- Additionally to force public key login instead of password: `sudo nano /etc/ssh/ssh_config`
- Scroll down and change the yes next to passwordAuthentication to 'no', then save.
- `sudo service ssh restart`


### Prepare to deploy project
#### Configure local timezone to UTC
- `sudo dpkg-reconfigure tzdata`
- Select `Etc` or `None of the above` and then select `UTC`

#### Install and configure Apache to serve a Python mod_wsgi application
- `sudo apt-get install apache2`
- Confirm apache is working by visiting the public/static IP address of the instance as the URL in a browser. The 'Apache2 Ubuntu Default Page' should appear
- `sudo apt-get install libapache2-mod-wsgi python-dev`  # The mod_wsgi package allows Apache to serve Flask applications
- `sudo nano /etc/apache2/sites-available/000-default.conf`

#### Install and configure PostgreSQL
- `sudo apt-get install postgresql`
- `sudo nano /etc/postgresql/9.5/main/pg_hba.conf` and do not allow remote connections by ensuring near bottom of the file it looks like the following:
```
local	all 	postgres 					peer
local	all 	all 						peer
host 	all 	all 		127.0.0.1/32	md5
host 	all 	all 		::1/128 		md5
```
- `sudo service postgresql restart`
- Configuration of project files to be compatible with psql can be found in the instructions further below.

#### Creating catalog user with limited permissions in PSQL
- `sudo su - postgres` to switch to postgres user
- `psql` to enter database
- `CREATE USER catalog;`
- To confirm the user is created, `\du` to look at the list of roles. Press q to stop looking at the table.
- `ALTER ROLE catalog WITH LOGIN;`
- `ALTER ROLE catalog CREATEDB;` allows catalog user to create databases
- `\password catalog` and then enter a password for the user. 'password' was used for the password. The same can be done for postgres. 'password' is also used.
- `CREATE DATABASE catalog WITH OWNER catalog;` creates a database named catalog. `\l` to confirm the table is there.
- Revoke all access rights except to `catalog` user.
```
postgres=# \c catalog;
postgres=# REVOKE ALL ON SCHEMA public FROM public;
postgres=# GRANT ALL ON SCHEMA public TO catalog;
```
- `\q` to exit the database. `exit` to logout and switch back to user ubuntu.

#### Install git
- `sudo apt-get install git`
- Make sure the .git directory is not publicly accessible via a browser
At the root of the web directory, add a .htaccess file and include this line:
`RedirectMatch 404 /\.git`


### Deploy Project
#### Clone project from Github repository
- `sudo mkdir /var/www/catalog` to create a directory called 'catalog' in /var/www destination
- `cd /var/www/catalog` to be inside the new catalog directory
- `sudo git clone https://github.com/jeffvhuang/catalog-app.git catalog`
The name of the folder will be changed from catalog-app to catalog to avoid potential problems with hyphens with Apache.
- `cd ..` to move back into /var/www
- `sudo chown -R ubuntu:ubuntu catalog` to change ownership of 'catalog' to ubuntu user

#### Setup a virtual environment and install dependencies
- `sudo apt-get install python-pip` # You may not want to upgrade pip to 10.X as it may change where its internals are situated. `python -m pip uninstall pip` can help to uninstall.
- `sudo apt-get install python-virtualenv`
- `cd /var/www/catalog/catalog`
- `virtualenv venviro` creates a virtual environment named 'venviro'
- `. venviro/bin/activate` activates the new 'venviro' environment. `deactivate` will exit the virtual environment.
- Install dependencies:
```
pip install httplib2
pip install requests
pip install --upgrade oauth2client
pip install sqlalchemy
pip install flask
pip install psycopg2-binary # you may need to use 'sudo apt-get install libpq-dev' instead
sudo apt-get install python-passlib
```
- `sudo python app.py` to run the app. It should display "Running on http://127.0.0.1:5000/" to indicate that the app is successfully configured.
- `deactivate` the environment.

#### Enable a virtual host
- `cd /etc/apache2/sites-available`
- `sudo cp 000-default.conf catalog.conf` to make of copy of the .conf file
- `sudo nano catalog.conf`
- Include the following lines within the `<VirtualHost>` blocks to configure Apache to handle requests using the WSGI module
```
	ServerName 35.176.129.195

	ServerAdmin jh0791@gmail.com
	DocumentRoot /var/www/catalog
	WSGIScriptAlias / /var/www/catalog/catalog.wsgi
 	<Directory /var/www/catalog/catalog/>
		WSGIProcessGroup catalog
		WSGIApplicationGroup %{GLOBAL}
		Order deny,allow
		Allow from all
	</Directory>
	Alias /static /var/www/catalog/catalog/static
	<Directory /var/www/catalog/catalog/static/>
		Order allow,deny
		Allow from all
	</Directory>
```
- `sudo a2ensite catalog` to enable the virtual host

#### Create WSGI file
- `nano /var/www/catalog/catalog.wsgi` to create a .wsgi file
- Add the following to the .wsgi file
```
activate_this = '/var/www/catalog/catalog/venviro/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'catalog_secret_key'
```
- `sudo service apache2 restart`

#### Google authentication
- Go to https://console.developers.google.com and go to the 'Catalog App' web application.
- Go into the Credentials tab
- Under 'Authorized Javascript origins' add http://35.176.129.195 and http://ec2-35-176-129-195.eu-west-2.compute.amazonaws.com
- Under 'Authorized redirect URIs' add http://ec2-35-176-129-195.eu-west-2.compute.amazonaws.com/login, http://ec2-35-176-129-195.eu-west-2.compute.amazonaws.com/gconnect and http://ec2-35-176-129-195.eu-west-2.compute.amazonaws.com/oauth2callback
- Download the json for these client secrets.
- Open the json file and copy all it's contents.
- Back in the terminal open the client_secrets.json file `sudo nano /var/www/catalog/catalog/client_secrets.json` and replace all current contents with the copied contents.


#### Modifications to ensure files are compatible with server (including database calls to use PostgreSQl rather than SQLite):
- `cd /var/www/catalog/catalog` to go into the project directory
- `mv app.py __init__.py` to change the file name
- `nano __init__.py` to edit the __init__.py file. In the file:
	1. On lines 20 and 50, change the path from 'client_secrets.json' to the full path '/var/www/catalog/catalog/client_secrets.json'
	2. On line 22, replace
	"engine = create_engine(‘sqlite:///itemCatalog.db’)"
	with
	"engine = create_engine('postgresql://catalog:password@localhost/catalog')"
	2. On/near lines 191 and 237, replace
	"session.query(Item).group_by(Item.category).all()"
	with
	"session.query(Item.category).distinct()"
	This is because the current way group_by is used is not compatible with psql
	3. Under "if __name__ == '__main__", on line 322, change "app.run(host='0.0.0.0'), port=8000)"" to "app.run()"
- In db_model.py and populate_db.py files, step 2 above also needs to be completed. The create_engine() function will be found on different lines of code.
- `sudo service apache2 restart` to refresh apache

#### Run the app!
Load the app at http://35.176.129.195/ and/or http://ec2-35-176-129-195.eu-west-2.compute.amazonaws.com/

#### Debugging
`cat /var/log/apache2/error.log`

#### Useful Commands
`sudo service apache2 restart` to restart apache whenever a file is changed.
`cd /var/www/catalog/catalog` to go to file with virtual environment and project files
`. venviro/bin/activate` to activate the 'venviro' virtual environment. Must be currently located in the directory where 'venviro' is located
`python __init__.py` to run the main app file (it has been changed from app.py to __init__.py)


## Third-Party resources used

- Amazon Lightsail Documentation https://lightsail.aws.amazon.com/ls/docs/en/getting-started
- GoDaddy: https://uk.godaddy.com/help/changing-the-ssh-port-for-your-linux-server-7306
- Network Time Procotol: http://support.ntp.org/bin/view/Support/TroubleshootingNTP
- Ask Ubuntu on Stack Exchange: https://askubuntu.com/questions/168280/how-do-i-grant-sudo-privileges-to-an-existing-user
https://askubuntu.com/questions/611584/how-could-i-list-all-super-users
https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt
- Digital Ocean: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
https://www.digitalocean.com/community/tutorials/how-to-use-roles-and-manage-grant-permissions-in-postgresql-on-a-vps--2
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- Medium: https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e
- Flask Documentation: http://flask.pocoo.org/docs/0.12/deploying/#deployment
http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
- Stack Overflow: https://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible
https://stackoverflow.com/questions/49836676/python-pip3-cannot-import-name-main-error-after-upgrading-pip
https://stackoverflow.com/questions/1769361/postgresql-group-by-different-from-mysql
https://stackoverflow.com/questions/22275412/sqlalchemy-return-all-distinct-column-values
- PostgreSQl for Beginners: http://www.postgresqlforbeginners.com/2010/11/sql-distinct-distinct-on-and-all.html


## SSH Key location for grader
