
# Linux Server Configuration

This is the final project for the Full-Stack Web Development Nanodegree from Udacity. 
The goal is to deploy the third project in the nanodegree [(Back-End Catalog)](https://github.com/jfbeyond/Instruments-Catalog) in a linux server.

In this page, the detailed step by step process as required by the process instructions indicates how to deploy said website. 
Important features:

- Ubuntu 16.04 LTS
- Amazon LightSail is used as private server
- The database server is changed from SQLite to PostgreSQL

Please visit to see the catalog website deployed.

# Get your Server.

### Create an instance in Amazon Lightsail (AWS)
- Follow the instructions on getting started on Lightsail [here](https://classroom.udacity.com/nanodegrees/nd004/parts/ab002e9a-b26c-43a4-8460-dc4c4b11c379/modules/357367901175462/lessons/3573679011239847/concepts/c4cbd3f2-9adb-45d4-8eaf-b5fc89cc606e)
- Keep your private and public ip addresses for use later
 
# Secure the Server.
- Generate key from your AWS account and move it to your home directory: ~/.ssh/yourNewKey.pem
- Connect to your server (new instance) from your machine:
  ```sh
  $ ssh -i ~.ssh/yourNewKey.pem ubuntu@publicIP
  ```
- Once in your server session, update and upgrade installed packages:
  ```sh
  $ sudo apt-get update
  $ sudo apt-get upgrade
  ```
  NOTE: If there are packages pending after using the two previous commands, update and upgrade with:
  ```sh
  $ sudo apt-get update && sudo apt-get dist-upgrade
  ```
  For reference check this [thread](https://serverfault.com/questions/265410/ubuntu-server-message-says-packages-can-be-updated-but-apt-get-does-not-update)
- Change SSH port from 22 to 2200:
You need to manually edit sshd_config file and change port number 22 to 2200. Please make sure that the external AWS firewall is configured accordingly too. To edit sshd_config:
  ```sh
  $ sudo nano /etc/ssh/sshd_config
  ```
- Configure UFW (Uncomplicated FireWall):
You must configure the firewall so it allows only incoming connections from ports 2200 (SSH), 80 (HTTP) and 123 (NTP). This is a critical step, since a bad configuration could lock you out of your server.
  ```sh
  $ sudo ufw status                  # The UFW should be inactive.
  $ sudo ufw default deny incoming   # Deny any incoming traffic.
  $ sudo ufw default allow outgoing  # Enable outgoing traffic.
  $ sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  $ sudo ufw allow www               # Allow HTTP traffic in.
  $ sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  $ sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
  ```
- Turn on the UFW:
After applying the preceeding rules you can activate the UFW:
  ```sh
  $ sudo ufw enable
  ```
- Check the UFW status. It should display something like this:
  ```
  Status: active
  
  To                         Action      From
  --                         ------      ----
  2200/tcp                   ALLOW       Anywhere                  
  80/tcp                     ALLOW       Anywhere                  
  123/udp                    ALLOW       Anywhere                  
  22                         DENY        Anywhere                  
  2200/tcp (v6)              ALLOW       Anywhere (v6)             
  80/tcp (v6)                ALLOW       Anywhere (v6)             
  123/udp (v6)               ALLOW       Anywhere (v6)             
  22 (v6)                    DENY        Anywhere (v6)
  ```
- Exit the ssh session: `$ exit`

### Give `grader` user access.

- Re-connect to your session from your local machine: 
  `$ ssh -i ~/.ssh/yourKey.pem -p 2200 ubuntu@publicIP`
- Create a new user 'grader':
  `$ sudo adduser grader`
  Fill out the information and keep the password as it will be used later.
- Give user 'grader' sudo capabilities:
  To do this, a new user file 'grader' will be added to the sudoers.d folder:
  `$ sudo touch /etc/sudoers.d/grader`
  Then, it will be edited:
  `$ sudo nano /etc/sudoers.d/grader`
  with the following lines to grant sudo permissions:
  ```
  grader  ALL=(ALL:ALL) ALL
  ```
### Create an SSH key pair for `grader` using `ssh-keygen` tool:
- Open another terminal window and in your home directory (not connected to your server):
 `$ ssh-keygen`
  You will be asked to create the file where the key pair will reside. It is recommended to keep it in your home directory: `~/.ssh/yourKeyPairFile`
- Enter the content of the new created `.pub` file and copy it:
  `$ cat ~/.ssh/yourKeyPairFile.pub`
- Go to your `grader` session  and create `.ssh` folder:
  `$ mkdir .ssh`
- Create and open to edit `authorized_keys` file within the `.ssh` folder:
  `$ touch .ssh/authorized_keys`
  `$ sudo nano .ssh/authorized_keys`
- Paste the contents of `yourKeyPairFile.pub` into `authorized_keys` and save and exit:
  `Ctrl + X` and type `Yes`
- Update permissions for `.ssh` folder and `authorized_keys` file:
  `$ chmod 700 .ssh`
  `$ chmod 644 .ssh/authorized_keys`
- Make sure that password authentication is inactive in `sshd_config`
  `$ sudo nano /etc/ssh/sshd_config`
  and in its contents scroll towards the line:
  ```
  pw authentication no
  ```
- Prohibit remote login as `root` in `ssh_config`:
  `PermitRootLogin no`
  
- Restart `ssh` service:
  `$ sudo service ssh restart`

- You can confirm the `grader` key authentication is working by logging in in a new session in another terminal:
  `$ ssh -i ~/.ssh/yourKeyPairKey -p 2200 grader@publicIP`

# Prepare to deploy the `catalog` project

### Configure the local time zone to UTC
- In the `grader` session:
 `$ sudo dpkg-reconfigure tzdata`
 A menu will open where you can pick the desired time zone.
 
 ### Install Apache
 
 - Download Apache in `grader`:
   `$ sudo apt-get install apache2`
  NOTE: If you type the public ip address of your instance in your web browser you will see the default APACHE2 Ubuntu page.
- Install `mod-wsgi`:
  `$ sudo apt-get install python-setuptools`
  `$ sudo apt-get install libapache2-mod-wsgi`
- Re-start Apache:
  `$ sudo service apache2 restart`

### Install and configure PostgreSQL
- Install PostgreSQL:
  `$ sudo apt-get install postgresql`
- Verify that no remote connections are allowed. This is done by reviewing the contents of `pg-hba.conf`:
  `$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
- Log in as `postgres` user:
  `$ sudo su -postgres`
- Get into the PostgreSQL shell:
  `$ psql`
- Create new Database and User, both named 'catalog':
  ```
  postgres = # CREATE DATABASE catalog;
  postgres = # CREATE USER catalog;
  ```
- Set a password for `catalog` user:
  ```
  postgres = # ALTER ROLE catalog WITH PASSWORD 'yourPassword';
  ```
- Grant `catalog` user full permissions to `catalog` database:
  ```
  postgres = # GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
  ```
- Check that a new database has been created by typying `/l`.
- Connect to `catalog` database `/c catalog`
- Revoke all rights from the database:
  ```
  # REVOKE ALL ON SCHEMA public FROM public;
  ```
- Give permission on database only to `catalog` user:
  ```
  # GRANT ALL ON SCHEMA public TO catalog;
  ```
- Exit `psql` with `\q`.
- Go back to `grader` session: `$ exit`

Basic PSQL commands link [here](http://www.postgresqltutorial.com/psql-commands/).

### Install `Git` to clone your project in the server
- Download `Git` in the grader session:
  `$ sudo apt-get install git`

# Deploy the Music Instrument Catalog project
This is the final step in the configuration, yet, it is very elaborated task. I followed these two websites to help understand how a `flask` application can be deployed in a Linux server: 
1. [Digital Ocean Flask App configuration](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
2. [Configuring mod_wsgi on Ubuntu](https://devops.profitbricks.com/tutorials/install-and-configure-mod_wsgi-on-ubuntu-1604-1/)

### Clone and set up the Music Instrument Catalog project
- Create folder `catalogApp` in `/var/www`:
  `$ cd /var/www/`
  `$ sudo mkdir CatalogApp`
- Clone project repository in `/var/www/CatalogApp/`:
  `$ sudo git clone https://github.com/jfbeyond/Instruments-Catalog.git`
- Rename repository folder and go into it:
  `$ sudo mv ./Instruments-Catalog ./CatalogApp`
  `$ cd CatalogApp`
- Change the owner of the directory `CatalogApp`:
  `$ sudo chown -R grader:grader /var/www/CatalogApp/CatalogApp`
- Rename `application.py` file to `__init__.py`:
  `$ sudo mv application.py __init__.py`
- As PostgreSQL is recommended in this project and the original catalog project used SQLite, a change in the create engine lines in `__init__.py`, `cat_setup.py` and `someinstruments.py` needs to be made:
 Find line: 
    ``` 
    engine = create_engine('sqlite://instrumentswithusers') 
    ```
  and replace it with:
   ``` 
   engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog') 
   ```
   where PASSWORD is the password used for the database user.
 - Install pip to create a virtual environment and required dependencies:
   `$ sudo apt-get install python-pip`
 - Install `virtualenv`, give it the name `venv` and activate it:
   `$ sudo pip install virtualenv`
   `$ sudo virtualenv venv`
   `$ source venv/bin/activate`
 - Install `Flask` and other required dependencies for the catalog project:
   `$ sudo pip install Flask httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests bleach`
 - Move one level up in the current directory structure and create `catalog.wsgi`. 
   `$ cd ..`
   `$ sudo nano catalog.wsgi`
    paste following and save:
   ```
   #!/usr/bin/python
   import sys
   import logging
   logging.basicConfig(stream=sys.stderr)
   sys.path.insert(0,"/var/www/CatalogApp/CatalogApp")

   from CatalogApp import app as application
   application.secret_key = 'super_secret_key'
   ```
 - Configure and enable a new virtual host:
   `sudo nano /etc/apache2/sites-available/catalog.conf`
    paste the following:
   ```
   <VirtualHost *:80>
                ServerName 18.212.151.249
                ServerAdmin ubuntu@18.212.151.249
                ServerAlias 18.212.151.249.xip.io
                WSGIScriptAlias / /var/www/CatalogApp/CatalogApp/catalog.wsgi
                <Directory /var/www/CatalogApp/CatalogApp>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/CatalogApp/CatalogApp/CatalogApp/static
                <Directory /var/www/CatalogApp/CatalogApp/CatalogApp/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
   ```
   
### Update OAuth credentials for the server

In this project Google log in was used for authentication. However, as Google does not accept single `ip` addresses for redirects a hostname is required to update the credentials settings.
- Obtain a hostname for the server instance public `ip` address:
  The free DNS service from `xip.io` is used in this project. By simply appending `.xip.io` to the instance's public IP we can get an usable url for redirects in the Google dev console.
  `publicIP.xip.io --> 18.212.151.249.xip.io`
NOTE: This has to be specified in the Virtual Host configuration file as shown previously in this document.
- You can either update your current google client for the catalog project or create a new one (I recommend creating a new one):
  1. Go to Google API Credentials [page](https://console.cloud.google.com/apis/credentials) and create new OAuth Client ID for your project.
  2. You will be given a new client ID and client secret. Fill out the required fields (javascript origins and redirectd URI's for your project):
  ![Google OAuth Client Configuration](https://snag.gy/R6J1Mq.jpg)  
  3. Download the JSON File and use it to replace the current `client_secrets.json` in the project. The google client ID must also be updated in the template `login.html`.
  
### Run the catalog project
At this point, the foundations for the catalog have been layed down and we're ready to start the server and make it active in the web.
- As in the third project, where a dummy file was run to initialize the catalog database, in this project the `someinstruments.py` must be run to give a couple of entries to the catalog, each one with two items:
  `$ sudo python cat_setup.py`
  `$ sudo python someinstruments.py`
These commands will ensure the database configuration and initial entries are in place to run the catalog website.
- Disable the default configuration file `000_default.conf` and enable the configuration file created for this project `catalog.conf`:
  `$ sudo a2dissite 000_default.conf`
  `$ sudo a2ensite catalog.conf`
- Activate Apache2:
  `$ sudo serviceapache2 reload`
  NOTE: This command must be run everytime you make changes in the configuration files and you run any of the two commands in the previous line.

The catalog app [Instruments & Accessories](http://18.212.151.249.xip.io/) should be up and live when typing the `publicIP.xip.io` --> `18.212.151.249.xip.io` in the browser.

![Catalog Snapshot](https://snag.gy/T40dVZ.jpg)

### General Notes
- This is not a straightforward project for those who'd never dealt with Linux before, however, it becomes very enjoyable if you're set the explore the web with ample curiosity. This is a project that requires a lot investigation and asking. I recommend you start by following the project guidelines as they highlight the big tasks. 
- When errors show up in the way there are two features I utilized the most: 1. Apache server error log and 2. Google dev tools. They can point what the source of the problem is.
  To access the apache server log from your virtual environment:
  `$ sudo nano /var/log/apache2/error.log`
- My project has the capability of displaying items pictures if an image URL link is provided. I placed the following pictures in the `/static/images` folder, which can be accessed by appending their names to the url `http://18.212.151.249.xip.io/static/images/PicName.ext`. Just for the sake of testing adding a new catalog item, you can use an URL link like this to include an image in the picture field, otherwise, no picture will be displayed.
  List of pictures:
    - drums.jpeg
    - electrical-drum.jpeg
    - guitar-chords.jpeg
    - piano-scoresheet.jpeg
    - wooden-tambor.jpeg
    
This is one of the features I plan to improve. To allow saving URL's from anywhere in the web.

### Acknowledments
- Udacity FSND Linux Server Configuration lessons and support in the Student Hub (Thanks to Sarmad G)
- Google, StackOverFlow.
