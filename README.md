# ws-itag semantic enhancement for Earth Observation data

**ws-itag** is a web service for the semantic enhancement of Earth Observation products, i.e. the tagging of products with additional information about the covered area, regarding for example geology, water bodies, land use, population, countries, administrative units or names of major settlements. It is based on **iTag** whose original repository can be found [on GitHub](https://github.com/jjrom/itag).

The description below should be sufficient for an installation on **CentOS 7** and the original instructions are only needed as reference.

# Installation on CentOS 7

It is assumed that the instructions below are followed by an administrator logged in as the **_root_** user.

## Preparation

On a CentOS 7 host, make sure the following packages are installed (the commands below install them):

* git
* wget
* unzip
* httpd
* php
* postgresql96-server <sup>1</sup>
* postgresql96-contrib<sup>1</sup>
* php-pgsql
* gdal<sup>1</sup>
* gdal-python<sup>1</sup>
* postgis23_96<sup>1</sup>
* postgis23_96-client<sup>1</sup>

<sup>1</sup>) from PostgresSQL 9.6 repository (_pgdg96_)
```
yum install -y git
yum install -y wget
yum install -y unzip
yum install -y httpd
yum install -y php
yum install -y https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
yum install -y postgresql96-server
yum install -y postgresql96-contrib
yum install -y php-pgsql
yum install -y epel-release
yum install -y gdal
yum install -y gdal-python
yum install -y postgis23_96
yum install -y postgis23_96-client
```

## Obtaining the iTag software

Create a home directory for the installation files and clone the ws-itag GIT repository:

```
export ITAG_HOME=/usr/local/src/itag
cd $ITAG_HOME
git clone https://github.com/Terradue/ws-itag.git
# or: git clone git@github.com:Terradue/ws-itag.git
```

## Database installation

Create the database cluster. If the file system of the default location (_/var/lib/pgsql/data_) does not have sufficient space, specify a different location using the _-D_ option (in that case the directory must exist and be empty and owned by the _postgres_ user).
```
/usr/pgsql-9.6/bin/postgresql96-setup initd
```
Edit the _pg_hba.conf_ file in the (default location is _/var/lib/pgsql/9.6/data/pg_hba.conf_) inserting these two authentication rules before any other authentication line (temporary setting only):
```
local   all             postgres                                trust
host    all             postgres        localhost               trust
```
Start the PostgreSQL server:

```
systemctl start postgresql-9.6
```

# Data ingestion

There are two options to populate the database.

**A:** Create the database from scratch and ingest the data from the downloaded data files. Since the ingestion involves a great amount of calculations, this can be very time-consuming.

**B:** Use a database dump from a previously installed application instance.

### Option A: Recompute from data sources

Create a directory for data sources and make it the current directory:
```
export ITAG_DATA=$ITAG_HOME/data
mkdir $ITAG_DATA
cd $ITAG_DATA
```
Execute the following commands to download and uncompress the data sources:
```
wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_coastline.zip
unzip ne_10m_coastline.zip
[ $? -eq 0 ] && rm ne_10m_coastline.zip

wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_0_countries.zip
unzip ne_10m_admin_0_countries.zip
[ $? -eq 0 ] && rm ne_10m_admin_0_countries.zip

wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_1_states_provinces.zip
unzip ne_10m_admin_1_states_provinces.zip
[ $? -eq 0 ] && rm ne_10m_admin_1_states_provinces.zip

wget http://download.geonames.org/export/dump/allCountries.zip
wget http://download.geonames.org/export/dump/alternateNames.zip
unzip allCountries.zip
unzip alternateNames.zip
[ $? -eq 0 ] && rm allCountries.zip
[ $? -eq 0 ] && rm alternateNames.zip

wget http://www.colorado.edu/geography/foote/maps/assign/hotspots/download/hotspots.zip
unzip hotspots.zip
[ $? -eq 0 ] && rm hotspots.zip

wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_glaciated_areas.zip
unzip ne_10m_glaciated_areas.zip
[ $? -eq 0 ] && rm ne_10m_glaciated_areas.zip

wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_rivers_lake_centerlines.zip
unzip ne_10m_rivers_lake_centerlines.zip
[ $? -eq 0 ] && rm ne_10m_rivers_lake_centerlines.zip

wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_geography_marine_polys.zip
unzip ne_10m_geography_marine_polys.zip
[ $? -eq 0 ] && rm ne_10m_geography_marine_polys.zip
```

Install the database schema by executing the following commands (gazetteer creation could take several hours):
```
$ITAG_HOME/_install/installDB.sh -F -H localhost -p itag
$ITAG_HOME/_install/installDatasources.sh -F -H localhost -D $ITAG_DATA
$ITAG_HOME/_install/installGazetteerDB.sh -F -D $ITAG_DATA
```

Download the ESA GlobCover 2009 data and install it (this could take several hours or even days, depending on the host performance):
```
wget http://due.esrin.esa.int/files/GLOBCOVER_L4_200901_200912_V2.3.color.tif
$ITAG_HOME/_install/computeGlobCover2009.php -f GLOBCOVER_L4_200901_200912_V2.3.color.tif
```

### Option B: Import existing database

This is most likely faster than option A, even considering the transfer of the database dump file (5.2 GB).

Copy the database dump file _itag-pgsql96-postgis23.dump_ and the corrective script _correct-itag-db.sh_ to the host. These files can be provided if requested.

From the directory containing the transferred files, execute the following command:
```
pg_restore -U postgres -d itag itag-pgsql96-postgis23.dump
```

Upon completion, run the corrective script to create the indexes whose creation will have failed during the installation:

```
sh correct-itag-db.sh
```

## Deployment of the web application

Since the web application uses _.htaccess_ files, there is no need for a specific configuration in the Apache configuration directory. It is enough to execute the deployment script:
```
ITAG_TARGET=/var/www/html/itag
$ITAG_HOME/_install/deploy.sh -s $ITAG_HOME -t $ITAG_TARGET
```

## Verification of successful installation

In a web browser, open the following URL (replace _<itag-host>_ with the real hostname):
 [http://<itag-host>/itag/?taggers=Physical,Geology,Hydrology,LandCover2009,Political,Population,Toponyms&footprint=POLYGON((12.2%2042.0,%2012.8%2042.0,%2012.8%2041.7,%2012.2%2041.7,%2012.2%2042.0))&_pretty=true](http://10.16.10.71/itag/?taggers=Physical,Geology,Hydrology,LandCover2009,Political,Population,Toponyms&footprint=POLYGON((12.2%2042.0,%2012.8%2042.0,%2012.8%2041.7,%2012.2%2041.7,%2012.2%2042.0))&_pretty=true)

The result should be a JSON document with coverage information for the Rome area in the following nodes: **_physical_**, **_hydrology_**, **_geology_**, **_landcover_**, **_political_**, **_toponyms_**.

**Note:** _population_ information is not (yet) included, due to data unattainability.

## Cleanup

Remove the previously added authentication rules in _pg_hba.conf_ (see above) and restart the PostgreSQL server.

Remove the content of _\$ITAG_HOME/data_ (optionally also the entire _\$ITAG_HOME_ directory) and/or the database dump file.
