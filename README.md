# Multi-Tier Web Application Hosted Locally

## Project Overview
A fully automated, local development environment simulating a real-world social media application architecture. This project demonstrates infrastructure-as-code principles using Vagrant to orchestrate multiple VMs hosting Nginx, Tomcat, MySQL, Memcached, and RabbitMQ.

## Architecture


## Project Goal:

This repository provides a fully automated, local replication of a multi-tier web application infrastructure using Vagrant and VirtualBox. It serves as a Rapid Development and Research & Development (R&D) baseline for a Java-based social media application.

## Key Technologies Demonstrated:

Automation: Vagrant, Shell Scripting

Virtualization: Oracle VirtualBox

Web Server/Proxy: Nginx

Application Server: Apache Tomcat

Data/Caching: MySQL, Memcached

Messaging (Future Expansion): RabbitMQ

Application Language: Java


## Application Execution Flow
User Access: User's browser → Nginx (acting as a reverse proxy).

Request Routing: Nginx forwards the request → Apache Tomcat (hosting the Java application).

Application Logic (Login): The Java application receives the request.

Caching Check: It first queries Memcached for existing session/login details.

Database Query: If not found in cache (first login or session expired), the application queries the MySQL database.

Messaging Layer: RabbitMQ is installed and running to demonstrate a production-ready setup but is not actively used in the current login flow.

## Project Workflow
1) Install the following in the local system
      -Oracle VM Virtualbox
      -Vagrant
      -Git
      -sublime text or VS-code 

2) Clone the source code from the Repository
    $ git clone https://github.com/vee-kay8/MultiTier_WebApp_Locally.git

3) Execute below command in your computer to install hostmanager plugin
    $ vagrant plugin install vagrant-hostmanager

4) CD into repository and bring up VMs**

      $ cd MultiTier_WebApp_Locally

      $ vagrant up

      NOTE: Bringing up all the vm’s may take a long time based on various factors. If vm setup stops in the middle run “vagrant up” command again.
      INFO: All the vm’s hostname and /etc/hosts file entries will be automatically updated.

5) Setup Services
  Setup should be done in below mentioned order:
    -MySQL (Database Service)
    -Memcache (DB Caching Service)
    -RabbitMQ (Broker/Queue Service)
    -Tomcat (Application Service)
    -Nginx (Web (Load balancing) Service)

  ### MYSQL Setup 
    Login to the db vm
      $ vagrant ssh db01
    Switch to root user
      $ sudo -i
    Update packages to latest versions
      $ dnf update -y 
    Set Repository
      $ dnf install epel-release -y 
    Install Maria DB Package
      $ dnf install git mariadb-server -y 
    Starting & enabling mariadb-server
      $ systemctl start mariadb
      $ systemctl enable mariadb
      RUN mysql secure installation script.
    this command asks a series of question to secure the database
      $ mysql_secure_installation

    NOTE: Set db root password, I will be using admin123 as password just because this is a home labproject, in real world, you should use a more complicaed password

    Set root password? [Y/n] Y New password:
    Re-enter new password: Password updated successfully! Reloading privilege tables..
    ... Success!

    By default, a MariaDB installation has an anonymous user, allowing anyone to log into MariaDB without having to have a user account created for them.  This is intended only for testing, and to make the installation go a bit smoother. You should remove them before moving into a production environment.
    Remove anonymous users? [Y/n] Y ... Success!

    Normally, root should only be allowed to connect from 'localhost'.  This ensures that someone cannot guess at the root password from the network.
    Disallow root login remotely? [Y/n] n ... skipping.

    By default, MariaDB comes with a database named 'test' that anyone can access.  This is also intended only for testing, and should be removed before moving into a production environment.
    Remove test database and access to it? [Y/n] Y - Dropping test database...
    ... Success!
    Removing privileges on test database...
    ... Success!
    Reloading the privilege tables will ensure that all changes made so far will take effect immediately.
    Reload privilege tables now? [Y/n] Y ... Success!


    Log in with root user
      $ mysql -u root -p
        Enter password:
    Create database
      MariaDB> create database accounts;
    this next command grants privillege to an account created called admin created locally with a password of admin123
      MariaDB> grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123';
    The next command allows remote login
      MariaDB> grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
    This reloads the DB
      MariaDB> FLUSH PRIVILEGES;
      MariaDB> exit;


    Download Source code & Initialize Database.
     $ cd /tmp/
     $ git clone https://github.com/vee-kay8/MultiTier_WebApp_Local.git
     $ cd MultiTier_WebApp_Local
    Restore the contents of the db_backup.sql file into the accounts database.
     $ mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
    log in to the "account" database
     $ mysql -u root -padmin123 accounts
       MariaDB> show tables;
       MariaDB> show databases;
       MariaDB> exit;
    Restart mariadb-server
       $ systemctl restart mariadb

  ### MEMCACHE SETUP 
    Login to the Memcache VM
      $ vagrant ssh db01
    Switch to root user
      # sudo -i
    Update packages to latest versions
      # dnf update -y
    Install, start & enable memcache on port 11211
      # sudo dnf install epel-release -y
      # sudo dnf install memcached -y
      # sudo systemctl start memcached
      # sudo systemctl enable memcached
      # sudo systemctl status memcached
    The next command enables the memcached to be able to connect remotely to tomcat and not just locally
      # sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
    Restart the memcached
      # sudo systemctl restart memcached
    Enable memcached to listen on port 11211 and udp port 11111
      # sudo memcached -p 11211 -U 11111 -u memcached -d

 
  ### RABBITMQ SETUP
    Login to the RabbitMQ vm
      $ vagrant ssh rmq01
    Switch to root user
      # sudo -i
    Update packages to latest versions
      # dnf update -y
    Set EPEL Repository
      # dnf install epel-release -y 
    Install Dependencies
      # sudo dnf install wget -y
      # dnf -y install centos-release-rabbitmq-38
      # dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
    Setup access to user test and make it admin
      # sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
      # sudo rabbitmqctl add_user test test
      # sudo rabbitmqctl set_user_tags test administrator
      # rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
      # sudo systemctl restart rabbitmq-server
      # systemctl enable --now rabbitmq-server

  ### TOMCAT SETUP
    Login to the tomcat vm
      $ vagrant ssh app01
    Switch to root user
      # sudo -i
    Update packages to latest versions
      # dnf update -y
    Set Repository
      # dnf install epel-release -y 
    Install Dependencies
      # dnf -y install java-17-openjdk java-17-openjdk-devel 
      # dnf install git wget vim unzip -y
    Change dir to /tmp
      # cd /tmp/
    Download Tomcat Package
      # wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
    Extract
      # tar xzvf apache-tomcat-10.1.26.tar.gz
    Add tomcat user
      # useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
    Copy data to tomcat home dir
      # cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/ 
    Make tomcat user owner of tomcat home dir
      # chown -R tomcat.tomcat /usr/local/tomcat Setup systemctl command for tomcat
    Create tomcat service file
      # vi /etc/systemd/system/tomcat.service 
        Update the file with below content
        [Unit]
        Description=Tomcat
        After=network.target
        [Service]
        User=tomcat
        Group=tomcat
        WorkingDirectory=/usr/local/tomcat
        Environment=JAVA_HOME=/usr/lib/jvm/jre
        Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
        Environment=CATALINA_HOME=/usr/local/tomcat
        Environment=CATALINE_BASE=/usr/local/tomcat
        ExecStart=/usr/local/tomcat/bin/catalina.sh run
        ExecStop=/usr/local/tomcat/bin/shutdown.sh
        RestartSec=10
        Restart=always
        [Install]
        WantedBy=multi-user.target

    Reload systemd files
      # systemctl daemon-reload 
    Start & Enable service
      # systemctl start tomcat
      # systemctl enable tomcat

    Maven Setup
    
    CODE BUILD & DEPLOY (app01)
      # cd /tmp/
      # wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
      # unzip apache-maven-3.9.9-bin.zip
      # cp -r apache-maven-3.9.9 /usr/local/maven3.9
      # export MAVEN_OPTS="-Xmx512m"
    Download Source code
      # git clone https://github.com/vee-kay8/MultiTier_WebApp_Local.git
    Update configuration
      # cd MultiTier_WebApp_Local
    Update file with backend server details
      # vim src/main/resources/application.properties

    Build code

    Run below command inside the repository (MultiTier_WebApp_Local)
      # /usr/local/maven3.9/bin/mvn install
    Deploy artifact  
      # rm -rf /usr/local/tomcat/webapps/ROOT*
      # cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
      # chown tomcat.tomcat /usr/local/tomcat/webapps -R
      # systemctl restart tomcat


  ### NGINX SETUP
    Login to the Nginx vm
      $ vagrant ssh web01
    Switch to root user
      # sudo -i
    Update packages to latest versions
      # apt update
      # apt upgrade
    Install nginx
      # apt install nginx -y 
    Create Nginx conf file
      # vi /etc/nginx/sites-available/vproapp 
    Update with below content

      upstream vproapp {
       server app01:8080;
      }
      server {
       listen 80;
      location / {
        proxy_pass http://vproapp;
      }
      }
    Remove default nginx conf
      # rm -rf /etc/nginx/sites-enabled/default 
    Create link to activate website
      # ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
    Restart Nginx
      # systemctl restart nginx


## Verification 
  Go to browser and put in the IP address previosly assigned to the NGINX virtual machine\
  http://192.168.10.11/\
  Note: The webpage that shows proves that NGINX is working and it is connected to the APP (TOMCAT Apache), when signin is successfull this proves the database is connected to the App. This shows the project was successful 

## Troubleshooting & Maintenance
  Restarting Services: Provide commands for restarting individual services on their respective VMs (e.g., sudo systemctl restart tomcat).

  Reloading VMs: "vagrant reload" command is used to restart and reload the VMs, this can also be done on the individual VMs for example, vabrant reload app01 

  Destroying the Environment: Command to completely remove all VMs and free up resources. "vagrant halt" to stop the VMs and "vagrant destroy" to delete the VMs
