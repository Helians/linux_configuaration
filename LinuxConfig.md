# Securing the server

## 1. Updating and Upgrading Packages og linux
- `sudo apt-get update && sudo apt-get upgrade`

## 2. Change the SSH port
- Edit the /etc/ssh/sshd_config file: `sudo vim /etc/ssh/sshd_config`
- Change the port number on line #port 22 from 22 to 2200 and uncomment it
- Save and exit
- Restart SSH: `sudo service ssh restart`

## 3. Configure the Uncomplicated Firewall (UFW)
- Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

 1. `sudo ufw status`                  # The UFW should be inactive
 2. `sudo ufw default deny incoming`   # Deny any incoming traffic
 3. `sudo ufw default allow outgoing`  # Enable outgoing traffic
 4. `sudo ufw allow 2200/tcp`          # Allow incoming tcp packets on port 2200
 5. `sudo ufw allow www`               # Allow HTTP traffic in
 6. `sudo ufw allow 123/udp`           # Allow incoming udp packets on port 123
 7. `sudo ufw deny 22`                 # Deny tcp and udp packets

- Turn UFW on by running this command: `sudo ufw enable`
- check the status by runnuning: `sudo ufw status`
- Exit form the SSH connection by using the command `exit`
- In Amazon lightsail instance go to **manage** --> **Networking** and hence change the firewall configuration to match the internal firewall setting
- Allow ports 80(TCP), 123(UDP), and 2200(TCP), and deny the default port 22
- From your local terminal, run: `ssh -i ~/.ssh/lightsail_key.pem -p 2200 ubuntu@13.235.19.244`

# Giving grader access

## 4. Creating the User
- Create a user using the command: `sudo adduser grader`
- Enter the password as **grader** or anything of your choice
- Install a package called **Finger** using: `sudo apt-get install finger` . Thins command will help you to check   whether the user has been created or not
- Now confirm that the new user is created using : `finger grader`

## 5. Give grader the permission to sudo
- Check the file sudoers : `sudo cat /etc/sudoers`
- At the bottom of the file you will see that there is a folder name listed : `/etc/sudoers.d`
- In this folder create a file grader : `sudo vim /etc/sudoers.d/grader`
- In this file add the following line : `grader ALL=(ALL) NOPASSWD:ALL`
- save and exit
- To check whether the grader has permission of sudo, first switch to grader : `su - grader`
- Type some command with sudo to verify : `sudo ls -a`

## 6. Create an SSH key pair for grader using the ssh-keygen tool
- On the local machine:
- Run `ssh-keygen`
- Enter file in which to save the key (I gave the name grader_key) in the local directory ~/.ssh
- Enter in a passphrase :  I have kept **grader** . Two files will be generated `( ~/.ssh/grader_key and ~/.ssh/grader_key.pub)`
- Run `cat ~/.ssh/grader_key.pub` and copy the contents of the file

- On the grader's virtual machine:
- Create a new directory called **~/.ssh** by using: `mkdir .ssh`
- Run `sudo vim ~/.ssh/authorized_keys` and paste the content into this file, save and exit
- Give the permissions to the files: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
- Check in `cat /etc/ssh/sshd_config` file if PasswordAuthentication is set to no
- Restart SSH: `sudo service ssh restart`
- On the local machine, run: `ssh -i ~/.ssh/grader_key -p 2200 grader@13.235.19.244`

# Prepare to deploy your project

## 7. Configure the local timezone to UTC
- While logged in as grader, configure the time zone: `sudo dpkg-reconfigure tzdata`
- It is already set to UTC

## 8. Install and configure Apache to serve a Python mod_wsgi application
- Install: `sudo apt-get install python3`
- Install: `sudo apt-get install python3-setuptools`
- While logged in as grader, install Apache: `sudo apt-get install apache2`
- Once the apache is installed, restart the apache using: `sudo service apache2 restart` and put the public ip in   the browser. You should see the apache default page
- I am using python 3 so i will be installing python 3 mod_wsgi : `sudo apt-get install libapache2-mod-wsgi-py3`
- Enable mod_wsgi using: `sudo a2enmod wsgi`

## 9. install and configure postgreSQL
- Install PostgreSQL: `sudo apt-get install postgresql`
- Check if no remote connections are allowed `sudo vim /etc/postgresql/9.3/main/pg_hba.conf`
- Login as user "postgres": `sudo su - postgres`
- Get into postgreSQL shell: `psql`
- Create a new database named catalog and create a new user named catalog in postgreSQL shell: `CREATE DATABASE catalog;` then `CREATE USER catalog;`
- Set a password for user catalog: `ALTER ROLE catalog WITH PASSWORD 'grader';`
- Give user "catalog" permission to "catalog" application database: `GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`
- Quit postgreSQL: `\q`
- Exit from user "postgres": `exit`

## 10. Install git and dependencies
- Install Git using : `sudo apt-get install git`
- Use `cd /var/www` to move to the /var/www directory
- Create the application directory `sudo mkdir FlaskApp`
- Move inside this directory using `cd FlaskApp`
- Clone the Catalog App to the virtual machine `sudo git clone https://github.com/Helians/udacity_flask_app.git FlaskApp`
- Move to the inner FlaskApp directory using `cd FlaskApp`
- Rename `main.py` to `__init__.py` using `sudo mv main.py __init__.py`
- Edit `item_catalog.py`, `item_catalog_insert.py` and `main.py` and change `engine = create_engine('sqlite:///       item_catalog_app.db')` to `engine = create_engine('postgresql://catalog:grader@localhost/catalog')`
- Install pip3 : `sudo apt-get install python3-pip`
- Use pip3 to install dependencies: `sudo pip3 install flask` , `sudo pip3 install --upgrade oauth2client ` , `sudo   pip3 install sqlalchemy` , `sudo pip3 install requests`

## 11. Configure and Enable a New Virtual Host
- Create FlaskApp.conf to edit: `sudo nano /etc/apache2/sites-available/FlaskApp.conf`
- Add the following lines of code to the file to configure the virtual host

 ```
  <VirtualHost *:80>
	ServerName 13.235.19.244
	WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
	<Directory /var/www/FlaskApp/FlaskApp/>
		Order allow,deny
		Allow from all
	</Directory>
  </VirtualHost>
```

- `sudo a2ensite FlaskApp`

- Create the .wsgi File under /var/www/FlaskApp:
  `cd /var/www/FlaskApp` then 
  `sudo vim flaskapp.wsgi`

- Add the following lines of code to the flaskapp.wsgi file:
  ```
	  import sys
	  import logging
	  logging.basicConfig(stream=sys.stderr)
	  sys.path.insert(0,"/var/www/FlaskApp/")

	  from FlaskApp import app as application
	  application.secret_key = 'Add your secret key' 
  ```
 - `sudo service apache2 restart`

## 12. Hosting instance
- Once your instance has started up, you can log into it with SSH from your browser.

- The public IP address of the instance is displayed along with its name. For me it is **13.235.19.244**.

**Note:** When you set up OAuth for your application, you will need a DNS name that refers to your instance's IP address. You can use the xip.io service to get one; this is a public service offered for free by Basecamp. For instance, the DNS name 13.235.19.244.xip.io refers to the server above
