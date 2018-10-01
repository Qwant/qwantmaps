# Qwant Maps

## Architecture

Qwant Maps can be seen as 4 separated components:

* a tile server
* a search engine (geocoder)
* an API to detail POIs
* a front end

### Tile server

A tile server is a service whose job is to give all that is needed to display a fraction of a map.

Qwant Maps provides only [vector tiles](https://en.wikipedia.org/wiki/Vector_tiles) so the tile server does not serve images (as it is done with [raster tiles](https://switch2osm.org/the-basics/)) but raw data. It is the front end, [Erdapfel](#erdapfel), that takes the data and renders it into a user browsable map.

The tile server is a combination of 2 great opensource projects:

* [kartotherian](https://github.com/kartotherian/kartotherian), the Wikimedia stack to have a highly available tile server. It is itself based on a number of [mapbox](https://www.mapbox.com/) components.
* [OpenMapTiles](https://github.com/openmaptiles/openmaptiles), for their great and flexible tile schema.

OpenMapTiles makes it possible for Qwant Maps to have an [easy to define/extend](https://github.com/QwantResearch/openmaptiles) vector tile schema.

All the data (see [below](#tilesdata)) are imported in a [PostgreSQL](https://www.postgresql.org/) database using mainly [imposm](https://imposm.org/docs/imposm3/latest/). All the world's vector tiles are then generated using kartotherian's [tilerator](https://github.com/kartotherian/tilerator) and stored in [Cassandra](http://cassandra.apache.org/). This enables Qwant Maps to have a fast and scalable way to serve tiles.

### Geocoder

The geographical search engine (also called geocoder) used for Qwant Maps is [Mimirsbrunn](https://github.com/CanalTP/mimirsbrunn).

Mimirsbrunn is a geocoder based on [Elasticsearch](https://www.elastic.co) and rust components developed by [Kisio Digital](http://www.kisiodigital.com/).

### POI API

To get more details on Poi Of Interests (POIs), Qwant Maps uses an additional API: [Idunn](https://github.com/QwantResearch/idunn), that combines the information in the geocoder with external APIs to format detailed POIs data to display in the front end.

### Front end <a name="erdapfel"></a>

Qwant Maps's front, [Erdapfel](https://github.com/QwantResearch/erdapfel), is a Javascript single page app that allows to browse the map, search for places, see your position on the map, etc. You can also save your favorite places, using [Masq](https://github.com/QwantResearch/masq).

The front end uses [Mapbox GL](https://www.mapbox.com/mapbox-gl-js/api/) to render the map, using both the tiles from the tile server, and Qwant Maps [custom map style](https://github.com/QwantResearch/qwant-basic-gl-style).

## Data

### Tiles data <a name="tilesdata"></a>

The main source of data for the tile server is the awsome [OpenStreetMap](https://www.openstreetmap.org) (OSM) data :heart: but other data like [natural earth](http://www.naturalearthdata.com/), better water polygon... are also used.

All the tile's data import process is defined in a [python script](https://github.com/QwantResearch/kartotherian_config/blob/master/import_data/tasks.py).

This script imports all the data in the PostgreSQL database and runs lots of post-process SQL functions (those functions are defined as postgresql triggers, so they will also be able to run on the [OSM updates](#osm_updates).

When the data is loaded in PostgreSQL, we use [tilerator](https://github.com/kartotherian/tilerator) to generate all the tiles from the zoom level 0 to 14 and store them in Cassandra.
This way, when a vector tile is requested to the tiles API [kartotherian](https://github.com/kartotherian/kartotherian), no computation needs to be done, only a simple query by id on the highly available Cassandra to get the raw protobuf tiles stored in it.

#### Daily OSM updates <a name="osm_updates"></a>

The import of the world's data is quite a long process, so after the initial import, we have a [cron job](https://github.com/QwantResearch/kartotherian_config/blob/master/update/osm_update.sh) to read the OSM differential updates and apply the diff in the PostgreSQL database.
The tool used to import in PostgreSQL ([imposm3](https://imposm.org/docs/imposm3/latest/)) can output the list of tiles impacted by the changes, so we can ask tilerator to generate them again.

### Geocoder data <a name="geocoder_data"><a/>

The data import process in [Mimirsbrunn](https://github.com/CanalTP/mimirsbrunn) is defined in a [python script](https://github.com/QwantResearch/docker_mimir/blob/master/task.py).

First, the OSM data is given to [Cosmogony](https://github.com/osm-without-borders/cosmogony) which outputs a big JSON file with all the world's administrative regions.

This file is then imported in Mimir by [cosmogony2mimir](https://github.com/CanalTP/mimirsbrunn#cosmogony2mimir).

We then import some addresses from [OpenAddresses](http://openaddresses.io/) using [openaddresses2mimir](https://github.com/CanalTP/mimirsbrunn#openaddresses2mimir).

The streets are imported afterwards from the OSM pbf file with [osm2mimir](https://github.com/CanalTP/mimirsbrunn#osm2mimir).

Finally the POIs are extracted from the PostgreSQL database (thus we have a unified POI handling) using [Fafnir](https://github.com/QwantResearch/fafnir).

### POI API data

Idunn does not need its own data import process, but the API depends on:

* the data of the [geocoder](#geocoder_data)
* Wikipedia data, fetched either on the Wikipedia APIs or in a custom Elasticsearch database.

## Global picture

![global architecture](images/global_archi.svg)

NB: To provide the Qwant Maps service, Qwant hosts instances of the tile server, the geocoder and the Idunn API on its own servers.

Please note that Qwant instances are internal components and are not meant (yet) to be used directly other than as part of Qwant Maps.

## How to run

### Tiles

To run the tileserver, the easiest way is to use Docker.

The project [kartotherian_docker](https://github.com/QwantResearch/kartotherian_docker) provides a docker-compose with all that is needed.

This repository heavily depends on [kartotherian_config](https://github.com/QwantResearch/kartotherian_config) that contains the real Qwant Maps business logic:

* imposm mappings/openmaptiles configurations (generated from [Qwant's openmaptiles's fork](https://github.com/QwantResearch/openmaptiles) with the [openmaptiles-tools](https://github.com/openmaptiles/openmaptiles-tools), cf the [readme](https://github.com/QwantResearch/openmaptiles#qwant-openmaptiles-fork))
* data import pipeline scripts
* update scripts
* kartotherian/tilerator generic configurations

### Geocoder

The repository [docker_mimir](https://github.com/QwantResearch/docker_mimir) contains a docker-compose to easily spawn the needed Docker containers and to import the needed data in them.

### POI API

[Idunn](https://github.com/QwantResearch/idunn) is quite straightforward to use as it is a simple python API (using [apistar](https://github.com/encode/apistar/)).

The [readme](https://github.com/QwantResearch/idunn#running) details the way to run it.

### Front end

Erdapfel is a simple node application but you can also use Docker to run it.

You will need to set a few environment variables to link it to the other Qwant Maps components. Check out the [readme](https://github.com/QwantResearch/erdapfel) to know more about the config.
