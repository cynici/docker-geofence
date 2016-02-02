# GeoServer + GeoFence

Serve Java web application GeoServer (frontend) and GeoFence (authentication and authorization) using [Tomcat](https://hub.docker.com/r/cheewai/tomcat/) in docker container.

## Notes

- *geofence-ovr* runtime directive does not work
- *hibernate-spatial-postgis-1.1.1.jar* required for using PostGreSQL database not included in *geofence.war*
- Non-admin users cannot log in to GeoFence web UI


## Prerequisites

- docker-engine 1.9.1+
- docker-compose 1.4+


## Set up PostgreSQL

Skip this section if you intend to use Hibernate file-based database instead of PostgreSQL

```
createuser --createdb --login --no-superuser --no-createrole --pwprompt geofence_test
CREATE DATABASE geofence OWNER geofence_test ENCODING 'UTF8';

# Connect to new database
\c geofence_test

CREATE EXTENSION postgis;
```

### GeoFence override

- Edit the following by replacing the placeholders {{}} with actual values and save it as *geofence-datasource-ovr.properties*. You may name it differently as long as it is also specified in the *docker-compose.yml*

- Comment out either the H2 stanza if you are using PostgreSQL database, or vice-versa. The parameter *geofenceDataSource.url* should use absolute pathname especially if you want to persist it outside the Docker container

- 

```
#geofenceVendorAdapter.databasePlatform=org.hibernatespatial.geodb.GeoDBDialect
#geofenceDataSource.driverClassName=org.h2.Driver
#geofenceDataSource.url=jdbc:h2:geofence_db/geofence
#geofenceDataSource.username=sa
#geofenceDataSource.password=sa
#geofenceEntityManagerFactory.jpaPropertyMap[hibernate.default_schema]=public

geofenceVendorAdapter.databasePlatform=org.hibernatespatial.postgis.PostgisDialect
geofenceDataSource.driverClassName=org.postgresql.Driver
geofenceDataSource.url=jdbc:postgresql://{{ dbserver }}:{{ dbport }}/{{ dbname }}
geofenceDataSource.username={{ dbuser }}
geofenceDataSource.password={{ dbpassword }}
geofenceEntityManagerFactory.jpaPropertyMap[hibernate.default_schema]={{ dbschema }}

################################################################################
## Other setup entries
################################################################################
## hbm2ddl.auto may assume one of these values:
## - validate: validates the DB schema at startup against the internal model. May fail on oracle spatial.
## - update: updates the schema, according to the internal model. Updating automatically the production DB is dangerous.
## - create-drop: drop the existing schema and recreates it according to the internal model. REALLY DANGEROUS, YOU WILL LOSE YOUR DATA.
## You may want not to redefine the property entirely, in order to leave the defult value (no action).

geofenceEntityManagerFactory.jpaPropertyMap[hibernate.hbm2ddl.auto]=update
geofenceEntityManagerFactory.jpaPropertyMap[javax.persistence.validation.mode]=none
geofenceEntityManagerFactory.jpaPropertyMap[hibernate.validator.apply_to_ddl]=false
geofenceEntityManagerFactory.jpaPropertyMap[hibernate.validator.autoregister_listeners]=false  

##
## ShowSQL is set to true in the configuration file; putting showsql=false in
## this file, you can easily check that this override file has been properly applied.

# geofenceVendorAdapter.generateDdl=false
# geofenceVendorAdapter.showSql=false

## Set to "true" in specific use cases
# workspaceConfigOpts.showDefaultGroups=false



################################################################################
## Disable second level cache.
## This is needed in a geofence-clustered environment.

#geofenceEntityManagerFactory.jpaPropertyMap[hibernate.cache.use_second_level_cache]=false

################################################################################
## Use external ehcache configuration file.
## Useful to change cache settings, for example diskStore path.
#geofenceEntityManagerFactory.jpaPropertyMap[hibernate.cache.provider_configuration_file_resource_path]=file:/path/to/geofence-ehcache-override.xml
```

## Create subdirectories

- downloads
- webapps

## Download files

Download the following files into subdirectory *downloads/*.

As at 2016-01-21, [GeoFence master branch](https://github.com/geoserver/geofence) supports GeoServer 2.8.x. It may be possible or necessary to download later version of files. Listed below are known to work at the time of writing.

```
wget http://download.oracle.com/otn-pub/java/jce/7/UnlimitedJCEPolicyJDK7.zip
# Unzip local_policy.jar US_export_policy.jar

wget http://sourceforge.net/projects/geoserver/files/GeoServer/2.8.1/geoserver-2.8.1-war.zip
# Unzip war file into webapps/

wget http://ares.boundlessgeo.com/geofence/master/geofence-master-latest-war.zip
# Unzip war file into webapps/

wget http://ares.boundlessgeo.com/geoserver/2.8.x/community-latest/geoserver-2.8-SNAPSHOT-geofence-plugin.zip

wget http://www.hibernatespatial.org/repository/org/hibernatespatial/hibernate-spatial-postgis/1.1.1/hibernate-spatial-postgis-1.1.1.jar
```


## Running the container for the first time

Before you begin, ensure that *webapps* directory contains *geoserver.war* and *geofence.war*

When Tomcat runs for the first time, on detecting the WAR files, it will uncompress them and start the application.

```
# Download the tomcat docker image
docker-compose -f docker-compose-firstrun.yml pull

# Download tomcat image
docker-compose -f docker-compose-firstrun.yml pull

# Start tomcat in foreground. When the application is ready to server, press control-C to stop it.
docker-compose -f docker-compose-firstrun.yml up
```

## docker-compose.yml

This example runs a single docker container serving both GeoServer and GeoFence on the same HTTP port. You may further lock down access to GeoFence web interface by running it in a separate instance and using [links](https://docs.docker.com/v1.8/compose/yml/#links) to allow only the GeoServer to reach it. But that is beyond the scope of this document.

Adjust the following YAML and save as *docker-compose.yml*.

```
tomcat7:
  container_name: geofence
  extends:
    file: docker-compose-firstrun.yml
    service: tomcat7
  ports:
    - "18080:8080"
  working_dir: /var/tmp
  #environment:
  #  GEOSERVER_DATA_DIR: /var/geoserver
  volumes:
    - ./webapps:/usr/tomcat/webapps
    - ./downloads/local_policy.jar:/usr/lib/jvm/default-jvm/jre/lib/security/local_policy.jar
    - ./downloads/US_export_policy.jar:/usr/lib/jvm/default-jvm/jre/lib/security/US_export_policy.jar
    - ./geofence-datasource-ovr.properties:/usr/tomcat/webapps/geofence/WEB-INF/classes/geofence-datasource-ovr.properties
    - ./downloads/hibernate-spatial-postgis-1.1.1.jar:/usr/tomcat/webapps/geofence/WEB-INF/lib/hibernate-spatial-postgis-1.1.1.jar
```

## Details docker-compose.yml

### Encryption strength

To install unlimited strength jurisdiction policy files for JVM, download the ZIP archive from http://docs.geoserver.org/latest/en/user/production/java.html containing and extract them to *downloads/* subdirectory

- local_policy.jar
- US_export_policy.jar

Mount these two files into the container:

```
volumes:
  - ./downloads/local_policy.jar:/usr/lib/jvm/default-jvm/jre/lib/security/local_policy.jar
  - ./downloads/US_export_policy.jar:/usr/lib/jvm/default-jvm/jre/lib/security/US_export_policy.jar
```

### GeoFence override

[Instructions](https://github.com/geoserver/geofence/wiki/GeoFence-configuration#providing-a-configuration-file) to apply an override file does not seem to be honored. Resort to mounting the override file instead with:

```
volumes:
  - ./geofence-datasource-ovr.properties:/usr/tomcat/webapps/geofence/WEB-INF/classes/geofence-datasource-ovr.properties
```

### Hibernate Spatial for PostGIS

Download from http://www.hibernatespatial.org/documentation/01-download/01-releases/ that is compatible with the Java classes for PostGreSQL+PostGIS [bundled in *geofence.jar*](https://github.com/geoserver/geofence/wiki/GeoFence-configuration#jdbc-drivers):

- postgis-jdbc-1.3.3.jar
- postgresql-8.4-702.jdbc3.jar

```
volumes:
  - ./downloads/hibernate-spatial-postgis-1.1.1.jar:/usr/tomcat/webapps/geofence/WEB-INF/lib/hibernate-spatial-postgis-1.1.1.jar
```

### JVM tuning

Before Tomcat starts the Java application, it runs the */usr/tomcat/bin/setenv.sh* shown below. 

```
# Tomcat Custom configuration

CATALINA_OPTS="-server -d64 -XX:+AggressiveOpts -Djava.awt.headless=true -XX:MaxGCPauseMillis=500 -XX:MaxPermSize=${MAXPERM} -XX:PermSize=${PERM} -Xmx${MAXMEM} -Xms${MINMEM} -Xincgc"

if $JMX ; then
    CATALINA_OPTS="${CATALINA_OPTS} -Xdebug -Xrunjdwp:transport=dt_socket,address=${DEBUG_PORT},server=y,suspend=n -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false  -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.port=${JMX_PORT} -Djava.rmi.server.hostname=${JMX_HOSTNAME} -Dcom.sun.management.jmxremote.rmi.port=${JMX_PORT}"
fi

export JAVA_OPTS="-Dgeofence-ovr=file:/gf-ovr.properties $JAVA_OPTS"
```

You may override the [default values](https://github.com/cynici/tomcat/blob/master/Dockerfile) for any of the variables in your *docker-compose.yml* file. GeoServer requires MINMEM greater or equal to 64 MB.

```
environment:
  MAXPERM: 256m
  PERM: 128m
  MAXMEM: 512m
  MINMEM: 128m
```
