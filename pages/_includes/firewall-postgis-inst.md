## Configure the firewall

If you are preparing a remote virtual machine, configure the firewall to allow remote access to the local port {{ include.port }}.

If you run a cloud based VM, also the VM itself shall be set to open this port.

## Install PostgreSQL and PostGIS

[PostgreSQL](https://www.postgresql.org/) is a relational database, and [PostGIS](http://postgis.net/) is its spatial extender, which allow you to store geographic objects like map data in it. PostgreSQL + PostGIS are used for a wide variety of uses such as rendering maps, geocoding, and analysis. It serves a similar function to ESRI’s SDE or Oracle’s Spatial extension.

Currently the tested versions are PostgreSQL 9.5 and PostGIS 2.2:

Also older PostgreSQL version should be suitable.

On Ubuntu there are pre-packaged versions of both postgis and postgresql, so these can simply be installed via the Ubuntu package manager.

```
sudo apt-get update
sudo apt-get install -y postgresql postgis pgadmin3 postgresql-contrib
```

Note: used PostgeSQL port is 5432 (default).

A user named {{ pg_user }} will be created during the installation process.

## Set the password for the *{{ pg_user }}* user

```
sudo -u {{ pg_user }} psql postgres
\password {{ pg_user }}
```

Enter the following password twice: {{ pg_password }}

This is just an example of password, you can use the one you prefer.

Exit with Control-D.

## Create the PostGIS instance

Now you need to create a postgis database. The defaults of various programs including openstreetmap-carto (ref. project.yaml) assume the database is called *gis*. You need to set up PostGIS on the PostgreSQL database.

```
export PGPASSWORD={{ pg_password }}
HOSTNAME=localhost # set it to the actual ip address or host name
psql -U {{ pg_user }} -h $HOSTNAME -c "create database gis" # alternative command: createdb -E UTF8 -O {{ pg_user }} gis
psql -U {{ pg_user }} -h $HOSTNAME -c "\connect gis"
psql -U {{ pg_user }} -h $HOSTNAME -d gis -c "CREATE EXTENSION postgis"
psql -U {{ pg_user }} -h $HOSTNAME -d gis -c "CREATE EXTENSION hstore"
```

The character encoding scheme to be used in the database is UTF8.

{% include_relative _includes/download-osm-data.md %}

## Install Osm2pgsql

Osm2pgsql is an OpenStreetMap specific software used to load the OSM data into the PostGIS database.

To install [osm2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql):

    sudo apt-get install -y osm2pgsql

### Alternative installation procedure

This alternative installation procedure generates the most updated executable by compiling the sources.

```
# Needed dependencies
sudo apt-get install -y make cmake g++ libboost-dev libboost-system-dev \
  libboost-filesystem-dev libexpat1-dev zlib1g-dev \
  libbz2-dev libpq-dev libgeos-dev libgeos++-dev libproj-dev lua5.2 \
  liblua5.2-dev

# Download osm2pgsql
cd /tmp
git clone git://github.com/openstreetmap/osm2pgsql.git 

# Prepare for compiling
cd osm2pgsql
mkdir build && cd build
cmake ..

# Compile
make

# Install
sudo make install

# Clean-out temporary files
cd ../..
rm -rf osm2pgsql
```

## Load data to PostGIS

```
cd {{ include.cdprogram }}
cd openstreetmap-carto
HOSTNAME=localhost # set it to the actual ip address or host name
osm2pgsql -s -C 300 -c -G -d gis --style openstreetmap-carto.style -H $HOSTNAME -U {{ pg_user }} [.osm or .pbf file]
```

Note: if you get the following error:

```
node_changed_mark failed: ERROR:  prepared statement "node_changed_mark" does not exist
```

do the following command on your *original.osm*:

```
sed "s/action='modify' //" < original.osm | > fixedfile.osm
```

Then process *fixedfile.osm*.

## Tuning

Information taken from [switch2osm](https://switch2osm.org/loading-osm-data).

The default PostgreSQL settings aren’t great for very large databases like OSM databases. Proper tuning can just about double the performance you’re getting. The most important PostgreSQL settings to change are `maintenance_work_mem` and `work_mem`, both which should be increased for faster data loading and faster queries while rendering respectively. Conservative settings for a 2GB VM are `work_mem=16MB` and `maintenance_work_mem=128MB`. On a machine with enough memory you could set them as high as `work_mem=128MB` and `maintenance_work_mem=1GB`. An overview to tuning PostgreSQL can be found on the [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server), but adjusting `maintenance_work_mem` and `work_mem` are probably enough on a development or testing machine.

To edit the PostgreSQL configuration file with *vi* editor:

    sudo vi /etc/postgresql/9.5/main/postgresql.conf

Suggested settings:

    shared_buffers = 128MB
    checkpoint_segments = 20
    maintenance_work_mem = 256MB
    autovacuum = off

To stop and start the database:

    sudo /etc/init.d/postgresql stop

    sudo /etc/init.d/postgresql start

## Create indexes

openstreet-carto shall be installed first.

```
HOSTNAME=localhost # set it to the actual ip address or host name
cd {{ include.cdprogram }}
cd openstreetmap-carto
scripts/indexes.py | psql -U {{ pg_user }} -h $HOSTNAME -d gis
```

Read [custom indexes](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md#custom-indexes) for further information.