# I am working with
- Raspberry Pi 5
- Ubuntu 24.04

# Contents
- [Problems I Ran Into](#problems-i-ran-into)
- [Step-by-Step Guide to Install Apache Guacamole on Ubuntu](#step-by-step-guide-to-install-apache-guacamole-on-ubuntu)
    - [1. Install Dependencies](#1-install-dependencies)
    - [2. Obtain the Server Source Code](#2-obtain-the-server-source-code)
    - [3. Build Process](#3-build-process)
    - [4. Install Apache Tomcat](#4-install-apache-tomcat)
        - [If tomcat9 Can Be Downloaded from Your Package Manager](#if-tomcat9-can-be-downloaded-from-your-package-manager-you-can-simply-install-it-as)
        - [If tomcat9 Cannot Be Downloaded from Your Package Manager](#if-tomcat9-cannot-be-downloaded-from-your-package-manager)
    - [5. Obtain the Guacamole Client (Web Interface)](#5-obtain-the-guacamole-client-web-interface)
    - [6. Deploy Guacamole](#6-deploy-guacamole)
    - [7. Configure Guacamole](#7-configure-guacamole)
    - [8. Access Guacamole](#8-access-guacamole)
    - [9. Set Up Authentication Extension](#9-set-up-authentication-extension)
        - [Set Up MySQL Database Using Docker](#set-up-mysql-database-using-docker)
        - [Set Up the Auth Extension](#set-up-the-auth-extension)
    - [10. Open Guacamole](#10-open-guacamole)

---

Guacamole for ARM64 processors needs to be compiled manually

#### Guacamole docs
https://guacamole.apache.org/doc/gug/installing-guacamole.html

#### Releases can be found here
https://guacamole.apache.org/releases/ 

# Problems I Ran Into
- There is no arm64 version of the official Guacamole Docker image
- Tomcat10 doesn't work with Guacamole 1.5.5
- RDP to Ubuntu doesn't work with Guacamole 1.5.5, but works with the official Github repo


# Step-by-Step Guide to Install Apache Guacamole on Ubuntu

## 1. Install Dependencies
```
$ sudo apt update
$ sudo apt upgrade

$ sudo apt install build-essential libcairo2-dev libjpeg-turbo8-dev libpng-dev libtool-bin uuid-dev libavcodec-dev libavformat-dev libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev libssh2-1-dev libvncserver-dev libwebsockets-dev libpulse-dev libssl-dev libvorbis-dev libwebp-dev
```

## 2. Obtain the Server Source Code
```
$ git clone https://github.com/apache/guacamole-server
$ cd guacamole-server
```

## 3. Build Process
Create the configure script:
```
$ autoreconf -fi
```

Configure the build:
```
$ ./configure --with-init-dir=/etc/init.d
```

Compile guacamole-server:
```
$ make
```

Install components:
```
$ sudo make install
```

Update the system's cache of installed libraries:
```
$ sudo ldconfig
```

## 4. Install Apache Tomcat
Since tomcat10 isn't compatible with the latest guacamole version (1.5.5 as of writing) and tomcat9 isn't available in Ubuntu 24.04 package manager, you need to compile it yourself.

### If tomcat9 Can Be Downloaded from Your Package Manager, You Can Simply Install It as
```
$ sudo apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user
```

[**Continue to step 5.**](#5-obtain-the-guacamole-client-web-interface)

### If tomcat9 Cannot Be Downloaded from Your Package Manager
```
$ sudo apt update
```

Install the default JDK:
```
$ sudo apt install default-jdk
```

Create user and group for tomcat:
```
$ sudo groupadd tomcat
$ sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat
```

Download Tomcat to the /tmp directory:
```
$ cd /tmp
$ curl -O https://downloads.apache.org/tomcat/tomcat-9/v9.0.89/bin/apache-tomcat-9.0.89.tar.gz
```

Unpack the downloaded archive:
```
$ sudo mkdir /opt/tomcat
$ sudo tar xzvf apache-tomcat-*tar.gz -C /opt/tomcat --strip-components=1
```

Change access permissions:
```
$ cd /opt/tomcat
$ sudo chgrp -R tomcat /opt/tomcat
$ sudo chmod -R g+r conf
$ sudo chmod g+x conf
$ sudo chown -R tomcat webapps/ work/ temp/ logs/
```

Copy the path to your Java JDK:
(by running the following command, you can get the path  
**Path example:** /usr/lib/jvm/java-1.21.0-openjdk-arm64)
```
$ sudo update-java-alternatives -l
```

Create a systemd service file:
```
$ sudo nano /etc/systemd/system/tomcat.service
```

And paste the following into it:  
```
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-1.21.0-openjdk-arm64
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

Change **JAVA_HOME** to the copied Java JDK location
```
    ...
Environment=JAVA_HOME=/usr/lib/jvm/java-1.21.0-openjdk-arm64
    ...
```

Reload the systemd daemon:
```
$ sudo systemctl daemon-reload
```

Start the Tomcat service:
```
$ sudo systemctl start tomcat
```

Try to open http://\<server_ip_address\>:8080
- If it works, great, [continue to step 5](#5-obtain-the-guacamole-client-web-interface)
- If it doesn't work, run the following command

```
$ sudo ufw allow 8080
```

## 5. Obtain the Guacamole Client (Web Interface)
The client is already compiled and can be downloaded as follows:
```
$ wget https://downloads.apache.org/guacamole/1.5.5/binary/guacamole-1.5.5.war
```

## 6. Deploy Guacamole
Copy the application to the Tomcat folder:
```
$ sudo mv guacamole-1.5.5.war /opt/tomcat/webapps/guacamole.war
```

Restart Tomcat and guacd:
```
$ sudo systemctl restart tomcat
$ sudo systemctl restart guacd
```

## 7. Configure Guacamole
Create the required configuration directory and files:
```
$ sudo mkdir /etc/guacamole
$ sudo nano /etc/guacamole/guacamole.properties
```

Add the following configuration to the *guacamole.properties* file:
```
guacd-hostname: localhost
guacd-port: 4822
```

Link the configuration file to Tomcat’s directory:
```
$ sudo ln -s /etc/guacamole /opt/tomcat/.guacamole
```

Start guacd:
```
$ sudo systemctl start guacd
$ sudo systemctl enable guacd
```

## 8. Access Guacamole
http://<server_ip_address>:8080/guacamole/

**Guacamole login page should appear, but there is no account to log in to.**

## 9. Set Up Authentication Extension
This way, you will be able to log in to the Guacamole web UI with all privileges.

### Set Up MySQL Database Using Docker
It doesn't have to be setup with docker, but this is the easiest way.  
Change the **MYSQL_ROOT_PASSWORD** value to your desired password:
```
$ docker run --name guacamole-mysql -e MYSQL_ROOT_PASSWORD=password -d mysql:latest
```

### Set Up the Auth Extension
Download the auth extension:
```
$ wget https://downloads.apache.org/guacamole/1.5.5/binary/guacamole-auth-jdbc-1.5.5.tar.gz
$ tar -xzf guacamole-auth-jdbc-1.5.5.tar.gz
```

Open the mysql directory:
```
$ cd guacamole-auth-jdbc-1.5.5/mysql
```

Find the MySQL container ID:
```
$ docker ps
```

Create a database for Guacamole:
```
$ docker exec -it <container_id> bash
$ mysql -u root -p
# CREATE DATABASE guacamole_db;
# quit;
```

Insert data into the database:
```
$ mysql -u root -p guacamole_db;
```

Copy and paste contents of **001-create-schema.sql** into the console.  
Copy and paste contents of **002-create-admin-user.sql** into the console.

Leave the container:
```
# quit;
# exit;
```

Create folders for the extension:
```
$ sudo mkdir /etc/guacamole/extensions
$ sudo mkdir /etc/guacamole/lib
```

Download MySQL connector:
```
$ wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j_8.4.0-1ubuntu24.04_all.deb
$ dpkg --extract mysql-connector-j_8.4.0-1ubuntu24.04_all.deb ./mysql-connector-j_8.4.0-1ubuntu24.04_all
```

Move the extension and the connector to the Guacamole directory:
```
$ sudo mv guacamole-auth-jdbc-mysql-1.5.5.jar /etc/guacamole/extensions/guacamole-auth-jdbc-mysql-1.5.5.jar
$ sudo mv mysql-connector-j_8.4.0-1ubuntu24.04_all/usr/share/java/mysql-connector-j-8.4.0.jar /etc/guacamole/lib/mysql-connector-j-8.4.0.jar
```

Edit guacamole.properties:
```
$ nano /etc/guacamole/guacamole.properties
```

Replace it with the following content:
```
guacd-hostname: localhost
guacd-port: 4822
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: root
mysql-password: <password_you_chose_when_setting_up_mysql_docker_container>
```

Restart Guacamole and Tomcat:
```
$ sudo systemctl restart guacd
$ sudo systemctl restart tomcat
```

------------------------------------------------
GUACAMOHLE_HOME = /etc/guacamole/

## 10. Open Guacamole
http://<server_ip_address>:8080/guacamole/

### And log in with default credentials:  
**Username**: guacadmin  
**Password**: guacadmin
