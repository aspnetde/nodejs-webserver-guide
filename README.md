# Node.js Web Server Guide

## Purpose

The main purpose of this guide is to summarize all the steps one has to take to get Node.js and MongoDB up and running on a Ubuntu web server. It is not 100% complete but it covers hopefully the most things you need to know to run different Node.js applications behind nginx, using MongoDB as a database.

If you find anything you can't agree with, please let me know. Send me an email or just a pull request and I will see if I am able to fix this ;-).

## Initial server installation

### Make Tools

The make tools are essential to build some npm packages and other stuff. So it’s generally a good idea to install them early.

	sudo apt-get install gcc make build-essential

(If the installation of build-essential fails, see Misc/Missing packages section).

### nginx

	sudo apt-get install nginx
	
Once the setup of nginx is complete, you should be able to call http://{server_ip} and see the default page with the “Welcome to nginx!” headline.

Also make sure the server starts automatically after booting the system (Should be enabled by default):

	sudo update-rc.d nginx defaults
	
### Node.js

	wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.14.0/install.sh | bash

	# Refresh Path
	source ~/.profile

	# Use latest Node.JS version
	nvm install v0.11.13
	
	# Make it default
	nvm use default v0.11.13

### Bower

	sudo npm install bower -g

### PM2

PM2 manages the different node applications running on this server.

	npm install pm2 -g

To start pm2 with the system:

	sudo env PATH=$PATH:/home/{username}/.nvm/v0.10.32/bin pm2 startup ubuntu -u {username}

(Replace v0.10.32 with the actual version.)

### Glances

Glances can be used to monitor the overall state of the server.

	sudo apt-get install python-pip build-essential python-dev
	sudo pip install Glances
	sudo pip install PySensors

### Git

	sudo apt-get install git

### Zip

	sudo apt-get install zip

### MongoDB

For a detailed explanation see [http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/).

#### Install the database service

	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
	echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list
	sudo apt-get update
	sudo apt-get install -y mongodb-org

#### Pin the current version

	echo "mongodb-org hold" | sudo dpkg --set-selections
	echo "mongodb-org-server hold" | sudo dpkg --set-selections
	echo "mongodb-org-shell hold" | sudo dpkg --set-selections
	echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
	echo "mongodb-org-tools hold" | sudo dpkg --set-selections

#### Set up security


##### Set up mongod.conf


	sudo vi /etc/mongod.conf 


Set:

	auth = true
	bind_ip = 127.0.0.1

	
##### Add admin user

Open the Mongo Console with `mongo`.

	use admin
	db.createUser({ user: "<username>",
          pwd: "<password>",
          roles: [ "userAdminAnyDatabase",
                   "dbAdminAnyDatabase",
                   "readWriteAnyDatabase"

	]})

##### Restart the service

	sudo service mongod restart

## Installation of a new website

### Create a MongoDB database

	mongo -u admin -p {adminpassword} --authenticationDatabase admin

Add a MongoDB user for the website

	use website-com
	db.createUser(
    {
      user: "website-com",
      pwd: "{newpassword}",
      roles: [
		{ role: "readWrite", db: "website-com" }
	]})

### Add the website’s directories

Websites are organized as follows:


The main identifier is the domain name of a website, while dots are replaced by dashes. 

Directory | Path
------------ | -------------
Single website’s root | /var/www/xyz-com
Single website’s Git repository	 | /var/www/xyz-com/repo
Single website’s web root | /var/www/xyz-com/www

Examples: 

- blog.xyz.de => blog-xyz-de
- website.com => website-com
	
### Add a dedicated website user

Use the domain name for the user’s name:

	sudo adduser website-com

### Make the user the owner of the website’s directories

In `/var/www` execute:

	sudo chown website-com website-com –R

### Create a Git repository

In `/var/www/website-com/repo` run

	git init --bare

### Add the Git deployment hook

This hook is used to deploy changes made to the master repository. It can be customized for each website depending on the specific needs.

Go to `/var/www/website-com/repo/hooks` and create a new file called “post-receive”:

	vi post-receive
	
Add the following commands to it:

	#!/bin/bash

	# Make this executable: chmod +x post-receive

	PREPARATION_DIR="/var/www/website-com/repo/$(uuidgen)"
	WEBSITE_ROOT="/var/www/website-com/www"
	PM2_APP_NAME="website-com"

	echo "Deployment started"

	read oldrev newrev branch

	if [[ $branch =~ .*/master$ ]];
	then
    	echo "Master received. Deploying to production..."

    	# Creates a temporary working directory
	    mkdir $PREPARATION_DIR

    	# Checks out the master from the repository
	    GIT_WORK_TREE="$PREPARATION_DIR" git checkout -f

    	# Installing all npm and bower modules/packages
	    cd $PREPARATION_DIR
	    npm install --allow-root
	    bower install --allow-root

	    # Removes all files in the Website's root
    	cd $WEBSITE_ROOT
		rm -rf *

		# Copies all files over
		cd $PREPARATION_DIR
		cp -r . $WEBSITE_ROOT

		# Restart the Website via PM2
		pm2 restart $PM2_APP_NAME

		# Removes the preparation directory
		rm -R $PREPARATION_DIR
	else
    	echo "$branch successfully received. Nothing to do: only the master branch may be deployed on this server."
	fi

	echo "Deployment finished"

Remember the value of **PM2_APP_NAME** and use it as an identifier for your pm2 application later.

After saving, make the script executable:

	chmod +x post-receive

### Push your application to your repository

By cloning your deployment repository on your local development machine and pushing the first version to the master branch, you should receive a fresh version at `/var/www/website-com/www` (Check with `ls –l`).

	git clone ssh://{username}@{ipaddress}/var/www/website-com/repo website-com

Make sure the user you’re connecting with has the necessary rights to run the Git repository. It’s recommended to connect with the user you have just created before to run the website, because he/she has the necessary access rights.

### Start the node application with PM2

With the `–u` parameter you can specify to run the application in the context of the given user, in our case the user you did just create before.

	cd /var/www/website-com/www/
	pm2 start app.js --name "website-com" -u website-com

If everything works PM2 reponds with `Process {nameofstarting.js}` launched. Wait a few seconds and use

	pm2 list

for a fresh status update. For more information see

	pm2 help

### Configure the website in nginx

We’re using a single configuration, which can be found at:

	sudo vi /etc/nginx/sites-available/default

nginx is running as a reverse proxy to handle all the public stuff on port 80 for us. It then passes all the traffic we want to to our node application.

	server {
    	listen 80;

	    server_name your-domain.com;

    	location / {
        	proxy_pass http://localhost:{YOUR_PORT};
	        proxy_http_version 1.1;
    	    proxy_set_header Upgrade $http_upgrade;
        	proxy_set_header Connection 'upgrade';
	        proxy_set_header Host $host;
    	    proxy_cache_bypass $http_upgrade;
	    }
	}

After saving the configuration, use

	sudo service nginx reload

to tell the server it should use it.

## Backup

As I am currently using a Hetzner server, I am also able to use 100 GB backup space which they offer free of charge. But anyway: If you're not hosting your server at Hetzner this may also apply to your actual hosting environment.

### Directory structure

Directory | Path
------------ | -------------
Backup Root | /var/backup
Website Backups | /var/backup/www
MongoDB Backups | /var/backup/mongo
nginx Backups | /var/backup/nginx

### MongoDB

#### Backup user

There needs to be a user with a backup role to create backups of all databases:

	use admin
	db.createUser(
    	{
	      user: "backup”,
    	  pwd: "{newpassword}",
	      roles: [{ role: "backup", db: "admin" }]
    	}
	)

#### Backup script

Save the following shell script as `/var/backup/create-backup-for-mongo` and make it executable:

	#!/bin/bash

	echo "Mongo Backup started"

	ADMIN_USERNAME="backup"
	ADMIN_PASSWORD="{password}"

	BACKUP_TARGET_ROOT="/var/backup/mongo"
	CURRENT_BACKUP_TARGET="$BACKUP_TARGET_ROOT/$(uuidgen)"

	# Remove all but the latest 30 backups
	cd $BACKUP_TARGET_ROOT
	rm -rf `ls -t | tail -n +30`

	# Back up all the databases to a new directory
	mongodump -u $ADMIN_USERNAME -p $ADMIN_PASSWORD -o $CURRENT_BACKUP_TARGET --authenticationDatabase admin

	zip -r "$(uuidgen).zip" $CURRENT_BACKUP_TARGET
	rm -rf  $CURRENT_BACKUP_TARGET

	echo "Mongo Backup finished"

#### Websites

Create the script `/var/backup/create-backup-for-www` and make it executable:

	#!/bin/bash

	echo "WWW Backup started"

	BACKUP_SOURCE="/var/www"

	BACKUP_TARGET_ROOT="/var/backup/www"
	CURRENT_BACKUP_TARGET="$BACKUP_TARGET_ROOT/$(uuidgen)"
	cd $BACKUP_TARGET_ROOT
	rm -rf `ls -t | tail -n +30`

	# Back up all the websites to a new directory
	rsync -a -E -c --stats $BACKUP_SOURCE $CURRENT_BACKUP_TARGET

	zip -r "$(uuidgen).zip" $CURRENT_BACKUP_TARGET
	rm -rf  $CURRENT_BACKUP_TARGET

	echo "WWW Backup finished"
	
#### nginx

Create the script `/var/backup/create-backup-for-nginx` and make it executable:

	#!/bin/bash

	echo "nginx Backup started"

	BACKUP_SOURCE="/etc/nginx"

	BACKUP_TARGET_ROOT="/var/backup/nginx"
	CURRENT_BACKUP_TARGET="$BACKUP_TARGET_ROOT/$(uuidgen)"

	# Remove all but the latest 30 backups
	cd $BACKUP_TARGET_ROOT
	rm -rf `ls -t | tail -n +30`

	rsync -a -E -c --stats $BACKUP_SOURCE $CURRENT_BACKUP_TARGET

	zip -r "$(uuidgen).zip" $CURRENT_BACKUP_TARGET
	rm -rf  $CURRENT_BACKUP_TARGET

	echo "nginx Backup finished"

### Mount the backup server 


#### Install cifs-utils

	sudo apt-get install cifs-utils

##### Mounting

This will mount the backup server as a local drive and it will reconnect on each reboot (it’s a single line added to fstab).

	sudo vi /etc/fstab 

 
	//u100445.your-backup.de/backup /mnt/backup-server       cifs    iocharset=utf8,rw,credentials=/var/backup/backup-credentials.txt,uid={localusername},gid={localusergroupid},file_mode=0660,dir_mode=0770 0       0

(If you are not aware of the group id, just type `id` and it will be displayed for the current user (in case you want to mount the drive in his/her context)).

#### Credentials

	vi /var/backup/backup-credentials.txt
	
Set:

	username={username}
	password={password}

#### Transfer script

Create a script that combines all backup actions and that finally transfers everything from the current backup folder to the backup server. Save that script as `/var/backup/create-and-transfer-backups` and make it executable.

	#!/bin/bash

	echo "Global Backup started"

	#Important: use absolute paths to be independent of the user context
	/var/backup/create-backup-for-mongo
	/var/backup/create-backup-for-www
	/var/backup/create-backup-for-nginx

	rsync -avzE --delete --stats /var/backup /mnt/backup-server

	echo "Global Backup finished"

#### Schedule backup

	sudo vi /etc/crontab 

Set:


	# m  h  dom mon dow user command
	10 14 *   *   *   root bash /var/backup/create-and-transfer-backups

(Runs the backup every day at 2:10 pm.)


## Miscellaneous


### Commands

Action | Command
------------ | -------------
Change User name |	passwd
Grant root priviliges | visudo, add: {username} ALL=(ALL:ALL) ALL
System Updates	| sudo apt-get update && sudo apt-get upgrade
Reboot | sudo reboot
Remove a directory and its sub-directories recursively | rm –R {directoryname} 
Remove everything, files and directories | sudo rm -rf *
Show current network settings | ifconfig
Make rudi the owner of a directory | chown -R rudi {directoryname}
Start a system service | sudo service {servicename} start
Restart a system service | sudo service {servicename} restart
Stop a system service | sudo service {servicename} stop
Reload a system service | sudo service {servicename} reload
See who’s the owner of a process | ps aux | grep {processname}
chmod Details |	[http://serverfault.com/a/357109](http://serverfault.com/a/357109) & [http://linuxcommand.org/lts0070.php](http://linuxcommand.org/lts0070.php)  
Grant a Role in MongoDB afterwards |	db.grantRolesToUser("username", [{ role: "readWrite", db: "dbname" }])
Make a shell script executable |chmod +x {scriptname}


### Add a SSH shortcut (on the development machine)

Create (or edit) a file named config in `~/.ssh/` (User's root directory)

	Host username
    	HostName xxx.xxx.xxx.xxx
    	User username

Now just type `ssh username` in terminal.

### Packages via apt-get not available?

If some packages are not available via apt-get, take a look at the sources configuration:

	sudo vi /etc/apt/sources.list

If the extras and the archive repositories are commented out, just remove the comments.
