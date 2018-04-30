OpenClinica
===========

This article documents the installation of OpenClinica. OpenClinica is an open source software to create electronic CRF (Case Report Form). In RadPlanBio, it is one of the main components.

To begin with, login as root (``sudo su``). Perform ``yum update``. Install some support packages such as yum-plugin-remove-with-leaves, links, bash-completion, net-tools, unzip, wget, vim, mlocate, epel-release, lsof, bzip2 and gcc.

Prior to installation, create an install folder to store the required .rpm and .war files.

.. code-block:: bash

  mkdir /usr/local/oc
  mkdir /usr/local/oc/install

Configurations for OpenClinica
------------------------------

================== =========== =============== ==============
OS                 Init        Container, JDK  Database
================== =========== =============== ==============
CentOS 7           ``systemd`` Tomcat 7, JDK 7 PostgreSQL 8.4
================== =========== =============== ==============

List of required files
----------------------

============== ==================================================
Tomcat 7       apache-tomcat-7.0.84.tar.gz

JDK 7          jdk-7u80-linux-x64.rpm

PostgreSQL 8.4 postgresql84-8.4.22-1PGDG.rhel6.x86_64.rpm

               postgresql84-libs-8.4.22-1PGDG.rhel6.x86_64.rpm

               postgresql84-server-8.4.22-1PGDG.rhel6.x86_64.rpm

               postgresql84-contrib-8.4.22-1PGDG.rhel6.x86_64.rpm

               postgresql84-docs-8.4.22-1PGDG.rhel6.x86_64.rpm
============== ==================================================

JDK
---

Remove OpenJDK if originally instaled on the system. Install Oracle jdk

.. code-block:: bash

  cd /usr/local/oc/install
  yum install jdk-7u80-linux-x64.rpm
  ln -s /usr/java/default /usr/local/java

Container
---------

Install container Tomcat 7

.. code-block:: bash

  cd /usr/local/oc/install
  mkdir /usr/tomcat
  tar xvzf apache-tomcat-7.0.84.tar.gz
  mv apache-tomcat-7.0.84 /usr/tomcat/.
  ln -s /usr/tomcat/apache-tomcat-7.0.84 /usr/tomcat/latest
  ln -s /usr/tomcat/latest /usr/tomcat/default
  ln -s /usr/tomcat/default /usr/local/tomcat

Create user for tomcat service

.. code-block:: bash

  groupadd tomcat1
  useradd -g tomcat1 tomcat1
  chown -Rf tomcat1.tomcat1 /usr/tomcat/apache-tomcat-7.0.84
  chown -Rf tomcat1.tomcat1 /usr/tomcat/latest/
  chown -Rf tomcat1.tomcat1 /usr/tomcat/default/
  chown -Rf tomcat1.tomcat1 /usr/local/tomcat/

Create .service file for init

.. code-block:: bash

  vim /lib/systemd/system/tomcat.service

.. code-block:: bash

  [Unit]
  Description=Tomcat version 7.0.85
  Documentation=https://tomcat.apache.org/download-70.cgi
  After=syslog.target
  After=network.target

  [Service]
  Type=forking
  Restart=always

  User=tomcat1
  Group=tomcat1

  Environment=JAVA_HOME=/usr/local/java
  Environment=CATALINA_HOME=/usr/local/tomcat
  Environment='JAVA_OPTS=-Xms128m -Xmx512m -XX:PermSize=128m'

  ExecStart=/usr/local/tomcat/bin/startup.sh
  ExecStop=/usr/local/tomcat/bin/shutdown.sh
  SuccessExitStatus=143

  TimeoutSec=0

  [Install]
  WantedBy=multi-user.target

Enable and start the service

.. code-block:: bash

  systemctl enable tomcat.service
  systemctl start tomcat.service
  systemctl status tomcat.service

Database
--------

To install PostgreSQL 8.4, the rpms are needed to be downloaded first

.. code-block:: bash

  cd /usr/local/oc/install/
  wget https://yum.postgresql.org/8.4/redhat/rhel-6-x86_64/postgresql84-libs-8.4.22-1PGDG.rhel6.x86_64.rpm
  wget https://yum.postgresql.org/8.4/redhat/rhel-6-x86_64/postgresql84-8.4.22-1PGDG.rhel6.x86_64.rpm
  wget https://yum.postgresql.org/8.4/redhat/rhel-6-x86_64/postgresql84-server-8.4.22-1PGDG.rhel6.x86_64.rpm
  wget https://yum.postgresql.org/8.4/redhat/rhel-6-x86_64/postgresql84-contrib-8.4.22-1PGDG.rhel6.x86_64.rpm
  wget https://yum.postgresql.org/8.4/redhat/rhel-6-x86_64/postgresql84-docs-8.4.22-1PGDG.rhel6.x86_64.rpm

Install the rpms

.. code-block:: bash

  yum install postgresql84-libs-8.4.22-1PGDG.rhel6.x86_64.rpm
  yum install postgresql84-8.4.22-1PGDG.rhel6.x86_64.rpm
  yum install postgresql84-server-8.4.22-1PGDG.rhel6.x86_64.rpm
  yum install uuid
  yum install postgresql84-contrib-8.4.22-1PGDG.rhel6.x86_64.rpm
  yum install postgresql84-docs-8.4.22-1PGDG.rhel6.x86_64.rpm

Initialize database for postgres

.. code-block:: bash

  /etc/init.d/postgresql-8.4 initdb

Edit ``postgresql.conf``

.. code-block:: bash

  cp /var/lib/pgsql/8.4/data/postgresql.conf /var/lib/pgsql/8.4/data/postgresql.conf.BAK
  vim /var/lib/pgsql/8.4/data/postgresql.conf

.. code-block:: bash

  listen addresses = 'localhost'
  port = 5432

Create symlink to ``/opt/PostgreSQL``

.. code-block:: bash

  mkdir /opt/PostgreSQL
  ln -s /usr/pgsql-8.4 /opt/PostgreSQL/8.4
  ln -s /var/lib/pgsql/8.4/data /opt/PostgreSQL/8.4/data

Prior to running the database, create an init .service file (non-optimized)

.. code-block:: bash

  rm /etc/init.d/postgresql-8.4
  vim /lib/systemd/system/postgresql-8.4.service

.. code-block:: bash

  [Unit]
  Description=PostgreSQL 8.4 database server
  Documentation=https://www.postgresql.org/docs/8.4/static/index.html
  After=syslog.target
  After=network.target

  [Service]
  Type=forking
  Restart=always

  User=postgres
  Group=postgres

  OOMScoreAdjust=-1000
  Environment=PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj
  Environment=PG_OOM_ADJUST_VALUE=0

  Environment=PGDATA=/var/lib/pgsql/8.4/data/

  ExecStart=/usr/pgsql-8.4/bin/pg_ctl start -D ${PGDATA} -s -w -t 300
  ExecStop=/usr/pgsql-8.4/bin/pg_ctl stop -D ${PGDATA} -s -m fast
  ExecReload=/usr/pgsql-8.4/bin/pg_ctl reload -D ${PGDATA} -s

  TimeoutSec=0

  [Install]
  WantedBy=multi-user.target

Enable and start the service

.. code-block:: bash

  systemctl enable postgresql-8.4.service
  systemctl start postgresql-8.4.service
  systemctl status postgresql-8.4.service

Assign password to user postgres

.. code-block:: bash

  su postgres
  psql

  ALTER USER postgres PASSWORD 'xxxxxxxxx';
  \q

  exit

Edit authentication method to access database by editing ``pg_hba.conf``

.. code-block:: bash

  cp /var/lib/pgsql/8.4/data/pg_hba.conf /var/lib/pgsql/8.4/data/pg_hba.conf.BAK
  vim /var/lib/pgsql/8.4/data/pg_hba.conf

.. code-block:: bash

  # TYPE  DATABASE        USER            ADDRESS                 METHOD
  local   all             all                                     md5
  host    all             all             127.0.0.1/32            md5

Setup database for OpenClinica

.. code-block:: bash

  psql -U postgres -c "CREATE ROLE clinica LOGIN ENCRYPTED PASSWORD 'xxxxxxxxx' SUPERUSER NOINHERIT NOCREATEDB NOCREATEROLE"
  psql -U postgres -c "CREATE DATABASE openclinica WITH ENCODING='UTF8' OWNER=clinica"
  psql -U postgres

  ALTER USER clinica WITH PASSWORD 'xxxxxxxxx';
  \q

OpenClinica
-----------

We go to the install folder and unzip OpenClinica war file

.. code-block:: bash

  systemctl stop tomcat.service

.. code-block:: bash

  cd /usr/local/oc/install
  unzip OpenClinica.war -d OpenClinica
  mv OpenClinica /usr/local/tomcat/webapps

Edit ``datainfo.properties`` file

.. code-block:: bash

  vim /usr/local/tomcat/webapps/OpenClinica/WEB-INF/classes/datainfo.properties

.. code-block:: bash

  dbType=postgres
  dbUser=clinica
  dbPass=xxxxxxxxx
  db=openclinica
  dbPort=5432
  dbHost=localhost

  filePath=${catalina.home}/${WEBAPP.lower}.data/

  sysURL=http://localhost:8080/${WEBAPP}/MainMenu

  log.dir=${catalina.home}/logs/openclinica
  logLocation=local

  logLevel=info
  syslog.host=localhost
  syslog.port=514

Start tomcat and depends on the ``filePath`` parameter, a folder will be created that contains a new ``datainfo.properties``. Subsequent changes on the settings must be performed on this file.

.. code-block:: bash

  systemctl start tomcat.service

Test connection to OpenClinica

.. code-block:: bash

  links 127.0.0.1:8080/OpenClinica/MainMenu

OpenClinica Web Service
-----------------------

The procedure is the same like installing OpenClinica

We go to the install folder and unzip OpenClinica war file

.. code-block:: bash

  systemctl stop tomcat.service

.. code-block:: bash

  cd /usr/local/oc/install
  unzip OpenClinica-ws.war -d OpenClinica-ws
  mv OpenClinica-ws /usr/local/tomcat/webapps

Edit ``datainfo.properties`` file

.. code-block:: bash

  vim /usr/local/tomcat/webapps/OpenClinica-ws/WEB-INF/classes/datainfo.properties

.. code-block:: bash

  dbType=postgres
  dbUser=clinica
  dbPass=xxxxxxxxx
  db=openclinica
  dbPort=5432
  dbHost=localhost

  filePath=${catalina.home}/${WEBAPP.lower}.data/

  sysURL=http://localhost:8080/${WEBAPP}/MainMenu

  log.dir=${catalina.home}/logs/openclinica
  logLocation=local

  logLevel=info
  syslog.host=localhost
  syslog.port=514

Start Tomcat and another folder will be created too with a new ``datainfo.properties`` file.

.. code-block:: bash

  systemctl start tomcat.service

Test connection to OpenClinica Web Service

.. code-block:: bash

  links 127.0.0.1:8080/OpenClinica-ws/MainMenu
