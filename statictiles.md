# Rendering Static Tiles

So you want to provide an offline slippy-map for your OpenLayers map. 
You want a rough map for most of the world, and a somewhat detailed map in one particular 
country (or area). 
You want to serve it with *just* a regular web-server (i.e. you don't want to deploy a tile server 
into the offline environment).  
What you'd like is to pre-render a low-resolution tile-set for the whole world and then a 
high-resolution tile-set for the particular country (area) in which customers are interested. 
You can then deploy just the pre-rendered tiles to the local http server.

First get yourself an OSM tile rendering Docker instance. You don't actually need the 
whole apache setup, but it's easy to set up, as it's already there:
```bash
$ sudo aptitude install docker.io git
$ sudo docker.io build .
$ git clone git@github.com:mcfletch/openstreetmap-tiles-docker.git
$ cd openstreetmap-tiles-docker
```
Now, you'll basically wait a forever-or-so while that builds, so let's get some information 
to render into our maps while we're waiting.
```bash
$ mkdir -p data
$ wget http://download.geofabrik.de/asia/vietnam-latest.osm.pbf
$ mv vietnam-latest.osm.pbf data/import.pbf
```
Okay, hopefully your build finished. Read the docker ID from the end of the build process,
you'll see a line like this (hopefully):
```
Successfully built 81b7b441efcc
```
That's the image that has our built software. Let's get that image set up and import our downloaded data:
```
$ sudo mkdir /var/virtualmachines/openstreetmapdb
$ sudo docker.io run -v /var/virtualmachines/openstreetmapdb:/var/lib/postgresql -v ~/openstreetmap-tiles-docker/data:/data 81b7b441efcc initdb startdb createuser createdb import
$ sudo docker.io run -v /var/virtualmachines/openstreetmapdb:/var/lib/postgresql -v ~/openstreetmap-tiles-docker/scripts:/root/scripts 81b7b441efcc cli
```
Now, inside the image's CLI, we're going to run our generate_tiles.py command (in the /root/scripts directory):
```bash
$ cd /root/scripts
$ python generate_tiles.py --help
$ mkdir output
```
You have to run the script as a user that has access to the gis database. The Dockerfile has 
set the www-data user as the owner of the `gis` database:
```bash
$ chown www-data output
$ sudo -u www-data python generate_tiles.py -z 0 -Z 7 --output=./output
```
Which will generate the first 8 zoom levels for the entire world. 
Now, you want to *also* create high-zoom maps for your area-of-interest, such as Vietnam.
Note that the coordinates are minx,miny,maxx,maxy where x goes from -180 to 180 degrees
and y goes from -90 to 90 degrees.
```bash
$ sudo -u www-data python generate_tiles.py -z 7 -Z 13 --bbox 101,8.4,110,24 --output=./output
```
You can find the bounding box for the area you want to render [online](http://www.latlong.net)
