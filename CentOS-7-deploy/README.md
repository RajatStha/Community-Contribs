# DefectDojo installation on CentOS 7 using Postgresql

## Introduction

Documentation how to install DefectDojo on CentOS 7 using docker-compose and having Postgresql on a separate host.  Due to the nature of CentOS and RHEL (RedHat Enterprise Linux), this guide will work equally well on RHEL 7.  New versions of CentOS/RHEL should also work with minor adjustments for the packages provided by the newer distro versions.

The guide is divided into two major sections:

1. Installation of Postgresql database on CentOS 7
2. Installation of the DefectDojo application on CentOS 7

High level description of the deployment requirements

* Single VM/server to run DefectDojo and any related services using Docker Compose
* Single VM/server to run Postgresql database server.

Deployment overview

* Both hosts have a local firewall restricting access to only needed ports
* Postgresql will allow DB connections only from the DefectDojo host at both the firewall and database server level
* Postgresql 11 will be used since CentOS 7 ships with Postgesql 9.2.4 and Django requires Postgresql 9.4 or newer
* This guide was written using the DefectDojo 1.5.4.1 release. Newer releases should also work if you update the docker-compose.yml file as needed.

Installation conventions

* Before starting the install, both VMs/servers are assumed to be up and running with CentOS installed and known IP addresses and/or hostnames.
* Any command-line starting wtih "#" is to be run by root or using sudo - your choice
* Install commands below use the most recent versions at the time of writing this guide. Newer versions may be available.  Check for them.  You have been warned. ;-)

## Installation of Postgresql on CentOS 7

**Update OS Packages to the latest**

```
# yum update -y
```

**Setup local firewall**

The following setup assumes you are going to run Postgresql on the defaul port (5432) and have port 22 for remote administration. Firewall will allow inbound connections to 22 (SSH) and allow the DefectDojo VM only to connect to Postgresql. Feel free to add other inbound traffic as your deployment requires.  And, yes, I'm lazy and use ufw to setup the firewall rules because it makes it super easy.  If you're l33t, feel free to use your preferred tool.

Replace the IP 8.8.8.8 with the IP address of the VM/server for the DefectDojo application.

```
# yum install -y epel-release
# yum install -y --enablerepo="epel" ufw
# ufw enable
# ufw allow 22
# ufw allow from 8.8.8.8 to any port 5432 proto tcp
# ufw status
```

The "ufw status" command should show port 22 open for any IP inbound and 5432 (Postgresql) only for the DefectDojo VM/server's IP. If you're new to ufw, Google is your friend.

**Setup Postgresql** (Skipped this step)

You will need to edit /etc/yum.repos.d/CentOS-Base.repo using the $EDITOR of your choice (vi, nano, ...) and add the following line to the [base] and [update] stanzas:

```
  exclude=postgresql*
```

So those stanzas look like:

```
    [base]
    name=CentOS-$releasever - Base
    mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
    #baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
    gpgcheck=1
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

    exclude=postgresql*

    #released updates
    [updates]
    name=CentOS-$releasever - Updates
    mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
    #baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
    gpgcheck=1
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

    exclude=postgresql*
```

To check that the exclusion of Postgresql worked, you can run the following:

```
# yum install postgresql
```

If the edits above worked, you'll get a message that postgresql is not available.

**Install Postgresql 11** (Skipped some of these instructions - Followed the install steps from here https://www.postgresql.org/download/linux/redhat/)

START - RJ
Followed the install steps from here https://www.postgresql.org/download/linux/redhat/
VERSION: psql (PostgreSQL) 16.3
```
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable postgresql-16 (Do this later after couple steps below, you will see this executed below)
sudo systemctl start postgresql-16 (Same as above do this later after couple steps below, you will see this executed below)
```
END - RJ


Start - SKIPPED
```
# yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
# /usr/pgsql-11/bin/postgresql-11-setup initdb
```
END - SKIPPED

Add the database server's IP address to postgresql.conf so remote hosts can connect.

```
# vi /var/lib/pgsql/16/data/postgresql.conf
```

Search for and find the line which reads:

```
listen_addresses = 'localhost'
```

And update it to read - making sure to replace 8.8.8.9 with the IP address of the Postgresql VM/server:

```
listen_addresses = 'localhost,8.8.8.9'
```

We also need to edit pg_hba.conf to allow the DefectDojo application to connect to Postgresql. Open that file in an editor:

```
# vi /var/lib/pgsql/16/data/pg_hba.conf
```

And find the line that matches the line below:

```
host    all             all             127.0.0.1/32            ident
```

And add the line below right after the line we found above ^ replacing 8.8.8.8 with the IP address of the DefectDojo application VM/server:

```
host    all             all             8.8.8.8/32              md5
```

Enable and startup Postgresql

```
# systemctl enable postgresql-16
# systemctl start postgresql-16
```

Check Postgresql's status

```
# systemctl status postgresql-16
```

**Setup Postgresql for DefectDojo**

Set the postgres (aka 'root' user for the database) replacing [your password here] with a long, random password

```
# sudo -i -u postgres psql template1
psql (10.9 (Ubuntu 10.9-0ubuntu0.18.04.1))
Type "help" for help.

template1=# ALTER USER postgres with encrypted password '[your password here]';
ALTER ROLE
template1=# \q

```

Restart Postgresql so the change takes effect

```
# systemctl restart postgresql-16
```

Switch to the postgres user and setup the database for DefectDojo. Change [DefectDojo DB password] to an appropriate long, random password.

```
# sudo -i -u postgres
$ createdb dojodb
$ createuser dojodbusr
$ psql -c "alter user dojodbusr with encrypted password '[DefectDojo DB password]';"
$ psql -c "grant all privileges on database dojodb to dojodbusr;"

# Connect to database "dojodb" as user "postgres".
# With PostgreSQL 15, this has changed ([source](https://www.postgresql.org/about/news/postgresql-15-released-2526/)):
#      PostgreSQL 15 also revokes the CREATE permission from all users except a database owner from the public (or default) schema.
$ psql
postgres=# \c dojodb postgres
$ GRANT ALL ON SCHEMA public TO dojodbusr;
```

The above:

* Creates a database called "dojodb" for DefectDojo to use
* Creates a DB user named "dojodbusr" for DefectDojo to log in with using the value you supplied for [DefectDojo DB password]
* Grants all privileges to the "dojodb" database to the "dojodbusr" DB user
* Grants all schema public permission to "dojodbusr" DB user 

Postgresql is now ready for DefectDojo.  Make sure and note down the hostname, port, database name, db username and password to use while setting up DefectDojo.

## Installation of DefectDojo using Docker-Compose on CentOS 7

**Update OS Packages to the latest**

```
# yum update -y
```
 
**Setup local firewall** (SKIPPED this)

The following setup assumes you are going to run DefectDojo on port 443 and have port 80 redirect to 443 (TLS).  Firewall will allow inbound connections to 22 (SSH), 80 (HTTP) and 443 (HTTPS). Feel free to add other inbound traffic as your deployment requires.  And, yes, I'm lazy and use ufw to setup the firewall rules because it makes it super easy.  If you're l33t, feel free to use your preferred tool.

```
# yum install -y epel-release
# yum install -y --enablerepo="epel" ufw
# ufw enable
# ufw allow 22
# ufw allow 80
# ufw allow 443
# ufw status
```

The "ufw status" command should show ports 22, 80 and 443 open for inbound connections from any IP. Adjust the rules as needed. If you're new to ufw, Google is your friend.

**Install Docker and Docker Compose** (SKIPPED this on Test Server - might want to install docker and all required docker packages for PRD - use the install steps from confluence) 

```
# yum remove docker docker-client docker-client-latest docker-common  docker-latest docker-latest-logrotate docker-logrotate docker-engine
# yum install -y yum-utils
# yum-config-manager  --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum install docker-ce docker-ce-cli containerd.io
# curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# chmod +x /usr/local/bin/docker-compose
```

**Setup DefectDojo installation directory** (Skipped the steps here and followed the steps from release/2.35.3 branch)

RJ - START HERE (This step was sufficient for Dojo using the release 2.35.3 branch)
```
# mkdir -p /opt/dojo
git clone https://github.com/DefectDojo/django-DefectDojo
# copy the local_settings.py file to django-DefectDojo/dojo/settings
# Replace the original the Jira template with custom ones
# Update the docker-compose.yml file
# Update the docker/environments/postgres-redis.env file
./dc-build.sh
./dc-up-d.sh postgres-redis
check the initializer container logs and validate
```
RJ - END HERE



```
# mkdir -p /opt/dojo
# cd /opt
  [copy over dojo subdirectory from this repo]
```

There's many ways to do "[copy over dojo subdirectory from this repo]" above. You could:

* git clone the DefectDojo Community repo (https://github.com/DefectDojo/Community-Contribs) and delete all but the dojo directory in the same location as this README.md.
* rsync over SSH the dojo directory and it's subdirectories from another host
* Add the dojo directory contents in a tarball/compressed archive into your favorite configuration mgmt system like Puppet, Chef, Ansible, Salt, ...

However you choose to do it, you should end up with a set of directories under /opt that looks like:

```
dojo
├── certs
├── docker-compose.yml
├── env.defectdojo
├── media
│   ├── CACHE
│   │   └── images
│   │       └── finding_images
│   └── finding_images
├── nginx
│   └── nginx.conf
├── setEnv.defectdojo
└── systemd
    └── defectdojo-compose.service

8 directories, 5 files
```

Note: The above doesn't show the hidden files named ".placeholder" which allow 'empty' directories to be stored in git.

**Test Connectivity with Postgresql**

Replace the example values in the psql command below with ones that fit your situation for 8.8.8.8 (IP address or hostname), dojodb (database for DefectDojo), dojodbusr (database user for DefectDojo to connect to the DB).  These will vary depending on what you did above in the "Installation of Postgresql on CentOS 7" section.

```
# yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
# yum install postgresql11
# psql -h 8.8.8.8 -p 5432 -d dojodb -U dojodbusr
Password for user dojodbuyum install epel-releasesr:
psql (9.2.24)
Type "help" for help.

dojodb=> \q
```

If connecting to Postgresql fails, you'll need to fix that before moving forward with the DefectDojo install.

**Setup OS user for DefectDojo Application**

```
# useradd -u 1001 dojosrv
# chown -R dojosrv.dojosrv /opt/dojo
# chmod g+s /opt/dojo/media /opt/dojo/media/CACHE /opt/dojo/media/finding_images
```

If there are additional users like a backup user account that need access to the DefectDojo files, then add them to the dojosrv OS group with the command below replacing "root" with the OS username.

```
# usermod -aG dojosrv root
```

**Make Docker start on boot/reboot**

```
# systemctl enable docker
# systemctl start docker
```

**Setup certificates for TLS**

You have loads of options here depending on how you get TLS certificates. However you get them, you'll need to put them in /opt/dojo/certs and make sure the certificate file is named "dojo.crt" and the key file is named "dojo.key".  Both files need to be in PEM format - not the binary DER format.

For example, here's an example of using a sym-link to have DefectDojo utilize Let's Encrypt:

```
# cp /etc/letsencrypt/live/second-cent.appsecpipeline.com/fullchain.pem /opt/dojo/certs/dojo.crt
# cp /etc/letsencrypt/live/second-cent.appsecpipeline.com/privkey.pem /opt/dojo/certs/dojo.key
```

At the end of this guide, I have a appendix on getting certificates from Let's Encrypt for those that want to use that service for TLS certificates.

**Set required Environmental variables**

The way you pass in deployment specific information into Docker Compose for DefectDojo is to set environmental variables.  The following ENV variables **must** be set and available to docker-compose for the application to start correctly:

* DD_DATABASE_URL - this will look something like postgres://dojodbusr:MyLongDBUserPassword@8.8.8.9:5432/dojodb where you use the value for [DefectDojo DB password] from the Postgresql setup section
* DD_SECRET_KEY - 128 random characters - used by Django for cryptographic functions.  Obviously, keep this secret.
* DD_CREDENTIAL_AES_256_KEY - 128 random characters - used by DefectDojo to encrypt any credentials stored in the DB.  Also keep this secret.

Only for the **first** time you start DefectDojo, you'll need to set this to initialize the installation:

* DD_INITIALIZE - set this to true ONLY for the first time you startup DefectDojo

There are other optional ENV variables. See the docker-compose.yml file for more information.

In the appendix below, you'll find suggested places to set these ENV variables.

**Copy over service file for DefectDojo and Enable the service**

```
# cp /opt/dojo/systemd/defectdojo-compose.service /etc/systemd/system/
# systemctl enable defectdojo-compose
```

**Start DefectDojo**

Before starting and initializing DefectDojo, check that the ENV variables work as expected. A quick check for warnings from docker-compose is to:

```
# source /opt/dojo/setEnv.defectdojo && docker-compose config | grep warning
```

To see how docker-compose will handle the variable substitutions in the yaml file (a more thorough check), do:

```
# source /opt/dojo/setEnv.defectdojo && docker-compose config
```

which will print the final version with substitutions docker-compose will execute.

For the **first time** you start DefectDojo, it's recommended to start it 'manually'.

```
# cd /opt/dojo
# source /opt/dojo/setEnv.defectdojo && DD_INITIALIZE="true" docker-compose up -d
```

You can also run with the verbose option to see the full output of docker-compose. I'd also suggest running this command in a screen session so you have access to a terminal prompt if needed.

```
# cd /opt/dojo
# source /opt/dojo/setEnv.defectdojo && DD_INITIALIZE="true" docker-compose --verbose up
```

**SUPER IMPORTANT**

Right after you initialize DefectDojo, run this command to get the Admin passwod so you can log into the Web UI:

```
# docker logs dojo_initialize_1 | grep Admin
```

Depending on how things went and how closely you followed these instructions, you can just startup DefectDojo and start enjoying the goodness:

```
# systemctl start defectdojo-compose
```

## Appendix

**Obtaining Let's Encrypt Certificates for DefectDojo**

Before running the commands below, make sure that:

* The hostname of the VM/server where the DefectDojo app is run resolves correctly in DNS
* Port 80 is open to the Internet for the Let's Encrypt ACME challenge to work correctly

```
# yum -y install yum-utils
# yum install epel-release
# yum install certbot
# certbot certonly --standalone
```

When this is done, you'll find:

* the cert at /etc/letsencrypt/live/[your hostname here]/fullchain.pem
* the key at /etc/letsencrypt/live/[your hostname here]/privkey.pem

**Customizing your install**

There's several ways to customize your installation depending on your needs, some options include:

* Setting environmental variables you want to persist by adding them to env.defectdojo
* Write some Bash in setEnv.defectdojo to pull in some run-time values such as pulling database credentials from Vault, CyberArk, AWS kms, Azure key vault, ...
* Tweak the settings for the nginx running in the container by modifying /opt/dojo/nginx/nginx.conf
* Use the default Nginx that ships with the defectdojo/defectdojo-nginx container by removing the volume line from docker-compose that reads:
  /opt/dojo/nginx/nginx.conf:/etc/nginx/nginx.conf

Best of luck

-- Matt Tesauro
   DefectDojo Maintainer
