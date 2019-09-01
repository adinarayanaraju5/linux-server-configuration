# Project - Linux Server Configuration

## Overview

The aim of the project is to deploy the web application online to run live on a secure web server.

This project explains how to secure and set up a Linux distribution on a server, install and configure a web and database server to host a plant catalog web application.

- The Linux distribution is [Ubuntu](https://www.ubuntu.com/download/server) 18.04 LTS.
- The virtual private server is [Amazon Lighsail](https://lightsail.aws.amazon.com/).
- The web application is my [plant Catalog project](https://github.com/adinarayanaraju5/plant_catalog).
- The database server is [PostgreSQL](https://www.postgresql.org/).

You can visit http://13.127.100.15.xip.io/ for the deployed plant catalog web app.

## Get a amazon lightsail server

### Step-1: Start a new Ubuntu 18.04 LTS Linux server instance on Amazon Lightsail

- Login into [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) using account of Amazon Web Services.
- Once you are login into the site, click `Create instance`.
- Choose `Linux/Unix` platform, `OS Only` and `Ubuntu 18.04 LTS`.
- Choose a instance plan (I took, \$3.50/month plan).
- Keep the default name provided by AWS or rename your instance.
- Click the `Create` button to create the instance.
- Wait for the instance to start up.

### Step-2: Start SSH into the server

- From the `Account` menu on Amazon Lightsail, click on `SSH keys` tab and download the Default Private Key.
- Move this private key file named `LightsailDefaultPrivateKey-*.pem` into the local folder `~/.ssh` and rename it `lightsail_key.rsa`.
- In your terminal, for file permission type: `chmod 600 ~/.ssh/lightsail_key.rsa`.
- To connect to the instance via the terminal: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@13.127.100.15`.

## Securing the server

### Step-3: Update and upgrade installed linux packages

```
sudo apt-get update && sudo apt-get upgrade
```

### Step-4: Change the SSH port

- Edit the `/etc/ssh/sshd_config` file: `sudo nano /etc/ssh/sshd_config`.
- Change the port number on line #port 22 from `22` to `2200`.
- Save and exit using CTRL+X and confirm with Y.
- Restart SSH: `sudo service ssh restart`.

### Step-5: Configure the Uncomplicated Firewall (UFW)

- Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # Deny tcp and udp packets.
  ```

- Turn UFW on by running this command: `sudo ufw enable`. The output should be like this:

  ```
  Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
  Firewall is active and enabled on system startup
  ```

- status of UFW is seen by running: `sudo ufw status`. The output should be like this:

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

- Exit the SSH connection: `exit`.

- Click on the `Manage` option of the Amazon Lightsail Instance,
  then the `Networking` tab, and then change the firewall configuration to match the internal firewall settings above.
  <img src="image/1.png" width="600px">

- Allow ports 80(TCP), 123(UDP), and 2200(TCP), and deny the default port 22.
  <img src="image/2.png" width="600px">

- From your local terminal, run: `ssh -i ~/.ssh/lightsail_key.rsa -p 2200 ubuntu@13.127.100.15`.

## creating user named grader and giving access

### Step-6: Create a new user account named `grader`

- While logged in as `ubuntu`, add user: `sudo adduser grader`.
- Enter a password of your choice I have used grader as the password and fill out information for this new user.

### Step-7: Give `grader` the permission to sudo

- Edits the sudoers file: `sudo visudo`.
- Search for the line that looks like this:

  ```
  root    ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `grader` user.

  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Save and exit using CTRL+X and confirm with Y.
- Verify that `grader` has sudo permissions. Run `su - grader`, enter the password,
  run `sudo -l` and enter the password again. The output should be like this:

  ```
  Matching Defaults entries for grader on ip-172-26-7-153:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

  User grader may run the following commands on ip-172-26-7-153:
    (ALL : ALL) ALL
  ```

### Step-8: Create an SSH key pair for `grader` using the `ssh-keygen` tool

- On the local machine:
- Run `ssh-keygen`
- Enter file in which to save the key (I gave the name `grader_key`) in the local directory `~/.ssh`
- Enter in a passphrase twice if you wish I have put grader as my passphrase. Two files will be generated ( `~/.ssh/grader_key` and `~/.ssh/grader_key.pub`)
- Run `cat ~/.ssh/grader_key.pub` and copy the contents of the file
- Log in to the grader's virtual machine
- On the grader's virtual machine:
- Create a new directory called `~/.ssh` by using (`mkdir .ssh`)
- Run `sudo nano ~/.ssh/authorized_keys` and paste the content into this file, save and exit
- Give the permissions to the files: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
- Check in `/etc/ssh/sshd_config` file if `PasswordAuthentication` is set to `no`
- Restart SSH: `sudo service ssh restart`
- On the local machine, run: `ssh -i ~/.ssh/grader_key -p 2200 grader@13.127.100.15`.

## Preparing to deploy the plant catalog project

### Step-9: Configure the local timezone to UTC

- While logged in as `grader`, configure the time zone: `sudo dpkg-reconfigure tzdata`. You should see something like that:

```

  Current default time zone: 'Etc/UTC'
  Local time is now: Thu Aug 29 15:59:55 UTC 2019.
  Universal Time is now: Thu Aug 29 15:59:55 UTC 2019.

```

### Step-10: Install and configure Apache to serve a Python mod_wsgi application

- While logged in as `grader`, install Apache: `sudo apt-get install apache2`.
- Enter public IP of the Amazon Lightsail instance into browser. If Apache is working, you should see:
  <img src="image/3.png" width="600px">

- My project is built with Python 2.7. So, I need to install the Python 2.7 mod_wsgi package:
  `sudo apt-get install libapache2-mod-wsgi`.
- Enable `mod_wsgi` using: `sudo a2enmod wsgi`.

### Step-11: Installing and configuring PostgreSQL

- While logged in as `grader`, install PostgreSQL:
  `sudo apt-get install postgresql`.
- PostgreSQL should not allow remote connections. In the `/etc/postgresql/10/main/pg_hba.conf` file, you should see:

```

local all postgres peer
local all all peer
host all all 127.0.0.1/32 md5
host all all ::1/128 md5

```

- Switch to the `postgres` user: `sudo su - postgres`.
- Open PostgreSQL interactive terminal by running `psql`.
- Create the `catalog` user with a password and give them the ability to create databases:

```

postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
postgres=# ALTER ROLE catalog CREATEDB;

```

- List the existing roles: `\du`. The output should be like this:

```

                                     List of roles

Role name | Attributes | Member of
-----------+------------------------------------------------------------+-----------
catalog | Create DB | {}
postgres | Superuser, Create role, Create DB, Replication, Bypass RLS | {}

```

- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.
- Create a new Linux user called `catalog`: `sudo adduser catalog`. Enter password and fill out information.
- Give to `catalog` user the permission to sudo. Run: `sudo visudo`.
- Search for the lines that looks like this:

```

root ALL=(ALL:ALL) ALL
grader ALL=(ALL:ALL) ALL

```

- Below this line, add a new line to give sudo privileges to `catalog` user.

```

root ALL=(ALL:ALL) ALL
grader ALL=(ALL:ALL) ALL
catalog ALL=(ALL:ALL) ALL

```

- Save and exit using CTRL+X and confirm with Y.
- Verify that `catalog` has sudo permissions. Run `su - catalog`, enter the password, run `sudo -l` and enter the password again. The output should be like this:

```

Matching Defaults entries for catalog on ip-172-26-13-170.us-east-2.compute.internal:
env_reset, mail_badpass,
secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User catalog may run the following commands on ip-172-26-13-170.us-east-2.compute.internal:
(ALL : ALL) ALL

```

- While logged in as `catalog`, create a database: `createdb catalog`.
- Run `psql` and then run `\l` to see that the new database has been created. The output should be like this:

```

                                    List of databases
     Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges

-----------+----------+----------+-------------+-------------+-----------------------
catalog | catalog | UTF8 | en_US.UTF-8 | en_US.UTF-8 |
postgres | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 |
template0 | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 | =c/postgres +
| | | | | postgres=CTc/postgres
template1 | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 | =c/postgres +
| | | | | postgres=CTc/postgres
(4 rows)

```

- Exit psql: `\q`.
- Switch back to the `grader` user by running: `exit` command.

### Step-12: Installing git version control

- While logged in as `grader` user, install `git`: `sudo apt-get install git`.

## Deploying the plant Catalog project

### Step-13: Clone and setup the Item Catalog project from the GitHub repository

- While logged in as `grader`, create `/var/www/catalog/` directory.
- Change to that directory and clone the catalog project:<br>
  `sudo git clone https://github.com/adinarayanaraju5/plant_catalog.git catalog`.
- From the `/var/www` directory, change the ownership of the `catalog` directory to `grader` using: `sudo chown -R grader:grader catalog/`.
- Change to the `/var/www/catalog/catalog` directory.
- Rename the `application.py` file to `__init__.py` using: `mv application.py __init__.py`.

- In `__init__.py`, replace line 27:

```

# app.debug = True

# app.run(host="0.0.0.0", port=8000, threaded= False)

app.run()

```

- In `database_setup.py`, replace sqlite database to postgresql:

```

# engine = create_engine("sqlite:///catalog.db")

engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')

```

### Step-14: Authenticate Google login by changing the javascript origins

- Go to [Google Cloud Plateform](https://console.cloud.google.com/).
- Click `APIs & services` on left menu.
- Click `Credentials`.
- Add http://13.127.100.15.xip.io and as authorized JavaScript origins.
- Add http://13.127.100.15.xip.io/oauth2callback as authorized redirect URI.
- Download the corresponding JSON file and copy it.
- Open `/var/www/catalog/catalog/client_secrets.json` and paste into the this file.

### Step-15: Installing the virtual environment and dependencies

- While logged in as `grader`, install pip: `sudo apt-get install python-pip`.
- Install the virtual environment: `sudo apt-get install python-virtualenv`
- Change to the `/var/www/catalog/catalog/` directory.
- Create the virtual environment: `sudo virtualenv -p python venv`.
- Change the ownership to `grader` with: `sudo chown -R grader:grader venv/`.
- Activate the new environment: `. venv/bin/activate`.
- Install the following dependencies by using requirements file:

```

pip install -r requirements.txt

```

- Run `python __init__.py` and you should see:

```

- Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)

```

- Deactivate the virtual environment: `deactivate`.

### Step-16: Set up and enable the virtual host

- Add the following line in `/etc/apache2/mods-enabled/wsgi.conf` file
  to use Python.

```

#WSGIPythonPath directory|directory-1:directory-2:...
WSGIPythonPath /var/www/catalog/catalog/venv/lib/python2.7/site-packages

```

- Create `/etc/apache2/sites-available/catalog.conf` and add the
  following lines to configure the virtual host:

```

<VirtualHost \*:80>
ServerName 13.127.100.15
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

- Enable virtual host: `sudo a2ensite catalog`. The following prompt will be returned:

```

Enabling site catalog.
To activate the new configuration, you need to run:
service apache2 reload

```

- Reload Apache: `sudo service apache2 reload`.

### Step-17: Set up the Flask application

- Create `/var/www/catalog/catalog.wsgi` file add the following lines:

```

    activate_this = '/var/www/catalog/catalog/venv/bin/activate_this.py'
    with open(activate_this) as file_:
        exec(file_.read(), dict(__file__=activate_this))

    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/catalog/")
    sys.path.insert(1, "/var/www/catalog/")

    from catalog import app as application
    application.secret_key = "rajus_secret"

```

- Restart Apache: `sudo service apache2 restart`.

### Step-18: Set up the database schema and populate the database

- Edit `/var/www/catalog/catalog/plant_data.py`.
- Add the these two lines at the beginning of the file.

```

import sys
sys.path.insert(0, "/var/www/catalog/catalog/venv3/lib/python2.7/site-packages")

# Delete all rows.

session.query(Item).delete()
session.query(Category).delete()
session.query(User).delete()

```

- From the `/var/www/catalog/catalog/` directory,
  activate the virtual environment: `. venv/bin/activate`.
- Run: `python plant_data.py`.
- Deactivate the virtual environment: `deactivate`.

### Step-19: Disable the default Apache site

- Disable the default Apache site: `sudo a2dissite 000-default.conf`.
  The following prompt will be returned:

```

Site 000-default disabled.
To activate the new configuration, you need to run:
service apache2 reload

```

- Reload Apache: `sudo service apache2 reload`.

### Step-20: Launch the Web Application

- Change the ownership of the project directories: `sudo chown -R www-data:www-data catalog/`.
- Restart Apache again: `sudo service apache2 restart`.
- Open your browser to http://13.127.100.15.xip.io

## Useful commands

- To get log messages from Apache server run: `sudo tail /var/log/apache2/error.log`.
- To restart Apache: `sudo service apache2 restart`.

## Resources

- Full stack web development - udacity -[udacity](https://www.udacity.com/courses/full-stack-developer?keyword=full%20stack%20web%20development).
- ServerPilot, [How to Create a Server on Amazon Lightsail](https://serverpilot.io/community/articles/how-to-create-a-server-on-amazon-lightsail.html)
- Official Ubuntu Documentation, [UFW - Uncomplicated Firewall](https://help.ubuntu.com/community/UFW).
- TechRepublic, [How to install and use Uncomplicated Firewall in Ubuntu](https://www.techrepublic.com/article/how-to-install-and-use-uncomplicated-firewall-in-ubuntu/).
- DigitalOcean, [How To Add and Delete Users on an Ubuntu 14.04 VPS](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps)
- DigitalOcean, [How To Set Up SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2).
- Ubuntu Wiki, [SSH/OpenSSH/Keys](https://help.ubuntu.com/community/SSH/OpenSSH/Keys).
- Ubuntu Wiki, [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime)
- Ask Ubuntu, [How do I change my timezone to UTC/GMT?](https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt/138442)
- DigitalOcean, [How To Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).
- Flask documentation, [virtualenv](http://flask.pocoo.org/docs/0.12/installation/).
- [Create a Python 3 virtual environment](https://superuser.com/questions/1039369/create-a-python-3-virtual-environment).
- Flask documentation, [Working with Virtual Environments](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/#working-with-virtual-environments)
- DigitalOcean [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
