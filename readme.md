# Qwant Maps

## Architecture

Qwant Maps can be seen as 4 separated components:

* a tile server
* a search engine (geocoder)
* a front end
* an API to detail Pois

### Tile server

A tile server is a service whose job is to give all that is needed to display a fraction of a map.

Qwant Maps provides only [vector tiles](https://en.wikipedia.org/wiki/Vector_tiles) so the tile server does not serve images (as it is done with [raster tiles](https://switch2osm.org/the-basics/)) but raw data. It is the front end, [tileview](#tileview) that takes the data and renders it into a user browsable map.

The tile server is a combination of 2 great opensource projects:

* [kartotherian](https://github.com/kartotherian/kartotherian), the wikimedia stack to have a highly available tile server. It is itself based on a number of [mapbox](https://www.mapbox.com/) components.
* [OpenMapTiles](https://github.com/openmaptiles/openmaptiles), for their great and flexible tile schema.

OpenMapTiles makes it possible for Qwant Maps to have a [easy to define/extend](https://github.com/QwantResearch/openmaptiles) vector tile schema.

All the data (see [below](#tilesdata)) are imported in a [Postgresql](https://www.postgresql.org/) database using mainly [imposm](https://imposm.org/docs/imposm3/latest/). All the world's vector tiles are then generated using kartotherian's [tilerator](https://github.com/kartotherian/tilerator) and stored in [cassandra](http://cassandra.apache.org/). This enable Qwant Maps to have a fast and scalable way to serve tiles.

### Geocoder

The geographical search engine (also called geocoder) used for Qwant Maps is [Mimirsbrunn](https://github.com/CanalTP/mimirsbrunn).

Mimirsbrunn is a geocoder based on [Elastic Search](https://www.elastic.co) and rust components developed by [Kisio Digital](http://www.kisiodigital.com/).

### Front end <a name="tileview"></a>

Qwant Maps's front, [tileview](:construction: TODO link to tileview) is a javascript single page app.

:construction: TODO more on this

### Poi API

To get more detail on Poi Of Interest (PoI) Qwant Maps uses an additional API, [Idunn](https://github.com/QwantResearch/idunn) that combines the information in the geocoder with external APIs to format detail PoIs data to display in the front end.

## Data

### Tiles data <a name="tilesdata"></a>

The main source of data for the tile server are the awsome [OpenStreetMap](https://www.openstreetmap.org) (OSM) data :heart: but other data like [natural earth](http://www.naturalearthdata.com/), better water polygon, ... are also used.

All the tile's data import process is defined in an [python script](https://github.com/QwantResearch/kartotherian_config/blob/master/import_data/tasks.py).

This script imports all the data in the Postgresql database and run lots of postprocess SQL functions (those functions are defined as postgresql triggers, so they will also be able to run on the [osm updates](#osm_updates).

When the data is loaded in postgresql, we use [tilerator](https://github.com/kartotherian/tilerator) to generate all the tiles from the zoom level 0 to 14 and store them in cassandra.
This way, when a vector tile is requested to the tiles API [kartotherian](https://github.com/kartotherian/kartotherian), no computation needs to be done, only a simple query by id on the highly available cassandra to get the raw protobuf tiles stored in it.

#### Daily OSM updates <a name="osm_updates"></a>

The import of the world's data is quite a long process so after the initial import, we have a [cron job](https://github.com/QwantResearch/kartotherian_config/blob/master/update/osm_update.sh) to read the osm differential updates and apply the diff in the postgresql database.
The tool used to import in postgresql ([imposm3](https://imposm.org/docs/imposm3/latest/)) can output the list of tiles impacted by the changes, so we can ask tilerator to generate them again.

### Geocoder data

The data import process in [Mimirsbrunn](https://github.com/CanalTP/mimirsbrunn) is defined in a [python script](https://github.com/QwantResearch/docker_mimir/blob/master/task.py).

First, the OSM data is given to [Cosmogony](https://github.com/osm-without-borders/cosmogony) which output a big json file with all the world's administrative regions.

This file is then imported in Mimir by [cosmogony2mimir](https://github.com/CanalTP/mimirsbrunn#cosmogony2mimir).

We then import some addresses from [OpenAddresses](http://openaddresses.io/) using [openaddresses2mimir](https://github.com/CanalTP/mimirsbrunn#openaddresses2mimir).

The streets are imported afterward from the osm pbf file with [osm2mimir](https://github.com/CanalTP/mimirsbrunn#osm2mimir).

Finally the PoIs are extracted from the postgresql database (thus we have a unified poi handling) using [Fafnir](https://github.com/QwantResearch/fafnir).

### Idunn

:construction: TODO

## Global picture

![global architecture](images/global_archi.svg)

## How to use

### Tiles

To run the tileserver, the easiest way is to use docker.

The project [kartotherian_docker](https://github.com/QwantResearch/kartotherian_docker) provides a docker-compose with all that is needed.

This repository heavily depends on [kartotherian_config](https://github.com/QwantResearch/kartotherian_config) that contains the real qwant maps business logic:

* imposm mappings / openmaptiles configurations (generated from [qwant's openmaptiles's fork](https://github.com/QwantResearch/openmaptiles) with the [openmaptiles-tools](https://github.com/openmaptiles/openmaptiles-tools), cf the [readme](https://github.com/QwantResearch/openmaptiles#qwant-openmaptiles-fork))
* data import pipeline scripts
* update scripts
* kartotherian/tilerator generic configuration

### Geocoder

The repository [docker_mimir](https://github.com/QwantResearch/docker_mimir) contains a docker-compose to easily spawn the needed docker containers and to import the needed data in them.

### Idunn

:construction: TODO

### Front end

:construction: TODO
