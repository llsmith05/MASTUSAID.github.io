# Deploying the DMI
## Hosting
Before beginning MAST deployment, first consider what type of server hosting you will be using. MAST can either be run on a cloud server, like Amazon Web Services (AWS), or on a local, physical server. Either option is viable, and the decision rests on the purpose, environment, and longevity of the project.

A cloud server is appropriate for a MAST instance that is large in scale (regional or national) and that requires consistent server uptime and availability, but does not have a physical environment that would allow these needs to be fulfilled by a local server, i.e. lacks reliable datacenters, electricity, or connectivity. Cloud servers ensure infrastructure reliability for a reasonable monthly fee, but do require an internet connection to reach. Costs tend to be reasonable, but the ability to pay them does require a consistent cash flow. Although cloud servers don’t require infrastructure maintenance, they are still servers and require system administration knowledge to set up and maintain.

A local server is appropriate for two cases: when MAST is only going to be utilized in a single office or location, or if there are high quality data centers available locally. In the first case, MAST can be installed on a single computer (even a laptop) and accessed only via a local network. Advantages are the low cost and low complexity, at the expense of flexibility and reach. It also requires a qualified local technician, or a preconfigured machine sent to the location. Configuration will be somewhat different than the cloud server. In the second case, configuration will be identical to that of a cloud server, but will require a database and significant infrastructure support, in addition to overall systems support. This type of solution is only likely to be viable in instances where data sovereignty is paramount and funding is sufficient to pay for high levels of support. Cost may be high.

### Amazon Web Services
This guide largely discusses hosting in terms of AWS, though any other hosting service could be used. Simply find a similar server configuration to those suggested below.

MAST can run on relatively small servers, as small as 't2.large'. However, for large deployments, 'm4.large' and above are recommended. While day to day operations don’t require significant resources, serving large amounts of imagery and executing topology checks can be CPU-intensive. It is also recommended that the MAST database run on a separate Amazon Relational Database Service (RDS) server. This makes database management and backup very easy. Since storage is relatively cheap on AWS, elect for 100 GB minimum on each server. SSD is preferable for performance reasons. If you wish to use a smaller server size, such as a t2-tier server, regular application function will likely be sufficient, but topology check runs may be significantly hampered and cause performance issues, and memory management may become an issue.

For a local server, select hardware with a similar profile to the AWS server above. A minimum of 4 GB of RAM, 200 GB of storage, and a modern, multicore processor are recommended. If there will be a very small number of users with minimal concurrency, most consumer laptops will run MAST, though performance may not be optimal.

##Server Setup and Prerequisites
###AWS Configuration
This guide assumes that the server is running Ubuntu 16.04. This is a long-term support version that is supported with free images from Amazon. Any other Linux distribution will function just as well; simply use your package manager of choice and substitute directory paths with the correct ones.

When configuring your AWS server, make sure that these ports are open via security settings.
- 22 (SSH): limit this to your own IP 
- 5432 (PostgreSQL): for the database server if you wish to directly connect a database management client; limit this to your own IP
- 80 (HTTP): for the application server; open this to all IPs that should have access to MAST, or leave open to all

At the time of this writing, MAST does not support SSL out of the box.

### OS and Package Installation
Before proceeding, update your apt packages and lists:
```
$ sudo apt-get update
$ sudo apt-get upgrade
```

Install Java JDK 8 using the webupd8 installer:
```
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
```
 
Install PostgreSQL and PostGIS extensions, if you are using a single server for application and database. If you are using Amazon RDS or equivalent, you will not need to do this, but will need to configure your RDS database to work with PostGIS.
```
$ sudo apt install postgresql postgresql-contrib postgis
```
 
There are no apt packages for Tomcat, so first browse to the Apache Tomcat 8 download page at https://tomcat.apache.org/download-80.cgi and copy the link for the tar.gz archive link. At the time of this writing, it is https://www-eu.apache.org/dist/tomcat/tomcat-8/v8.5.38/bin/apache-tomcat-8.5.38.tar.gz. Then download the package and unzip it to the `/opt` folder:
 
Files will be unzipped into `apache-tomcat-8.5.38` folder. Rename it to `tomcat`.
```
$ sudo mv /opt/apache-tomcat-8.5.38/ /opt/tomcat
```

Download and install GeoServer for serving map layers. GeoServer will be deployed to the Tomcat server and therefore we need the WAR package. For this guide, the latest 2.15.0 is used.
```
$ wget 'http://sourceforge.net/projects/geoserver/files/GeoServer/2.15.0/geoserver-2.15.0-war.zip/download'
```

Rename downloaded file `geoserver.zip` and unzip it to `geoserver` folder.
```
$ mv download geoserver.zip
$ unzip geoserver.zip -d geoserver
```

Now the `geoserver` folder contains different files and folders. We need to copy geoserver.war into our Tomcat folder under the web applications subfolder - `/opt/tomcat/webapps`.
```
$ sudo cp geoserver/geoserver.war /opt/tomcat/webapps
```

Once the Tomcat server is started, geoserver.war will be expanded into the geoserver folder and deployed as a web application.
In order to connect to the MAST database, download appropriate JDBC drivers for PostgreSQL and PostGIS and put them into the `/opt/tomcat/lib` folder. The following lines use the latest versions as of this writing but you may check [here](https://jdbc.postgresql.org/download.html) for postgres and [here](here for postgis) for postgis driver updates. For this guide, since we installed Java 8 or higher, use version 4.2.
```
$ sudo wget https://jdbc.postgresql.org/download/postgresql-42.2.5.jar -P /opt/tomcat/lib/
$ sudo wget http://central.maven.org/maven2/net/postgis/postgis-jdbc/2.3.0/postgis-jdbc-2.3.0.jar -P /opt/tomcat/lib/
```

###Configuration
####Configure PostgreSQL Password
First, run the psql utility using postgres account and change and then change the password. If you used Amazon RDS, you’ve already done this during the setup process and will not need to repeat the process now.
 ```
 $ sudo -u postgres psql
postgres=# \password postgres
```
Remember this password, as it will be used to access the database in the future.

####Configure Tomcat
First, open your `.bashrc` file for editing using your favorite editor.
```
$ vim ~/.bashrc
```

And add the following lines (assuming Java 8 is installed):
```
JAVA_HOME=/usr/lib/jvm/java-8-oracle
CATALINA_HOME=/opt/tomcat
```

Create or edit `setenv.sh` file in the Tomcat folder under `/opt/tomcat/bin` with the editor of your choice and set the following content:
```
CATALINA_OPTS="$CATALINA_OPTS -Dfile.encoding=UTF8 -Duser.timezone=UTC -server -d64 -XX:NewSize=700m -XX:MaxNewSize=700m -Xms1024m -Xmx2048m -XX:MaxPermSize=1024m -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:SurvivorRatio=20 -XX:ParallelGCThreads=8"
```

This line sets different options for your Tomcat. The most important ones are `-Xms` (sets minimum allocated memory) and `-Xmx` (sets maximum allocated memory). Make sure that you have enough RAM for the configured values.

Make sure that the file is executable:
```
sudo chmod ug+x /opt/tomcat/bin/setenv.sh
```

Add the following resource reference to `/opt/tomcat/conf/context.xml` inside the Context tags, i.e. before `</Context>`:
```
<ResourceLink global="jdbc/mast" name="jdbc/mast" type="javax.sql.DataSource"/>
```

Add the following lines, configuring MAST database connection, to `/opt/tomcat/conf/server.xml` file inside the GlobalNamingResources tags, i.e. right before closing `</GlobalNamingResources>` tag, swapping out the host, database name, username and password strings for the correct ones. Please make sure that the database is not created yet and remember database name, configured in this setting. This name has to be used at the next steps when creating database.
```
<Resource type="javax.sql.DataSource"
name="jdbc/mast"
factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
driverClassName="org.postgresql.Driver"
url="jdbc:postgresql://host:5432/dbname"
username="username"
password="password"
testWhileIdle="true"
testOnBorrow="true"
testOnReturn="false"
validationQuery="SELECT 1"
validationInterval="30000"
timeBetweenEvictionRunsMillis="5000"
maxActive="100"
minIdle="10"
maxWait="10000"
initialSize="10"
removeAbandonedTimeout="60"
removeAbandoned="true"
logAbandoned="true"
minEvictableIdleTimeMillis="30000"
jmxEnabled="true"
jdbcInterceptors="org.apache.tomcat.jdbc.pool.interceptor.ConnectionState;
org.apache.tomcat.jdbc.pool.interceptor.StatementFinalizer;
org.apache.tomcat.jdbc.pool.interceptor.SlowQueryReportJmx(threshold=10000)"
/>
```

##Application and Database Deployment
###Initialize Database
Fetch a copy of the latest scripts from the GitHub repository:
 ```
 $ git clone https://github.com/MASTUSAID/DB-SCRIPTS.git
 ```
 
If you don’t have Git installed, you can download the database creation script directly:
```
$ wget https://raw.githubusercontent.com/MASTUSAID/DB-SCRIPTS/master/create_database.sql
```

Login to your postgres server and create a new database. Your connection string will vary depending on your server choice.
```
$ sudo -u postgres psql
postgres=# CREATE DATABASE mast;
```

If you are using RDS, you will have to enable PostGIS support by manually adding several extensions. Follow the instructions in [Amazon’s documentation for RDS here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html#Appendix.PostgreSQL.CommonDBATasks.PostGIS). You should end up with several new topology schemas.

Restore the database schema and data downloaded from the GitHub repository:
```
postgres=# \c mast
postgres=# \i create_database.sql
```

You may need to specify full path to the `create_database.sql` file if it’s not in the current folder. If you have no errors, check to ensure that the database has been populated.

###Install the DMI Application
You can either clone and compile MAST-DMI application from the [GitHub repository](https://github.com/MASTUSAID/MAST-DMI) or download a compiled version from [releases](https://github.com/MASTUSAID/MAST-DMI/releases/). Check for the latest release and download mast.war from the assets list. At the time of writing this guide, the latest release is 3.1.
```
$ wget https://github.com/MASTUSAID/MAST-DMI/releases/download/3.1/mast.war
```

If you are planning to do further development of MAST or explore the code, clone and open the project in your favorite Java IDE, supporting Maven.
Before building the application, you may wish to further configure some items, such as reports. MAST reports are configured using Jasper Reports, and all templates can be found under `/src/main/resources/reports` folder. Detailed use of the Jasper report editor is outside the scope of this document, but details and help can be found online at http://community.jaspersoft.com/documentation.

Once you have the WAR file available, copy it to the Tomcat `webapps` folder.
```
$ sudo cp mast.war /opt/tomcat/webapps/
```

###Start the Server
Finally, start your Tomcat server by using the following command:
```
$ sudo /opt/tomcat/bin/startup.sh
```

If you choose, you may also install Tomcat as a systemd service, though this is outside the scope of this guide.

Wait a few moments (e.g. 1-2 minutes) and check that the application has been deployed by going to `http://[your_server_IP]:8080/mast`. If you are testing from the local machine, use localhost instead of IP address. Please, note that if you didn’t configure the HTTP port for Tomcat, it will use 8080 by default. This can be changed in the `server.xml` configuration file.
If components were properly installed and configured, you should see the login screen.
