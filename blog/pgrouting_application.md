title: Building a Routing Application using PostGIS, OpenStreetMap, PgRouting and MapLibre
date: 2026-01-11 12:21
tags: osm, pgrouting, gis
category: osm
slug: pgrouting_application
author: Philipp Wagner
summary: Shows how to build a Routing Application using PostGIS, OpenStreetMap and PgRouting

In this article I want to share a map and route planning application I have been working the 
past weekends. It is built on PostGIS, Open Street Map, PgRouting and the maplibre-gl library.

All code can be found in a Git repository at:

* [https://github.com/bytefish/PgRoutingExperiments](https://github.com/bytefish/PgRoutingExperiments)

## Table of contents ##

[TOC]

## Screenshots ##

The final application allows you to browse a map and calculcate routes for 3 transportation types:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/pgrouting_application/screenshot.jpg">
        <img src="/static/images/blog/pgrouting_application/screenshot.jpg" alt="Route Application">
    </a>
</div>

We can also do a Network Analysis for finding routing islands in our data:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/pgrouting_application/screenshot-network-analysis.jpg">
        <img src="/static/images/blog/pgrouting_application/screenshot-network-analysis.jpg" alt="Route Application Network Analysis">
    </a>
</div>


## Running the Application ##

### Certificates ###

Create and Trust the Developer Certificates using `dotnet`:

```
dotnet dev-certs https --clean
dotnet dev-certs https -ep ${HOME}/.aspnet/https/aspnetapp.pfx -p SuperStrongPassword --trust
```

### Data ###

I am using the Natural Earth Relief using the following raster tiles:

* [https://github.com/lukasmartinelli/naturalearthtiles/releases/download/v1.0/natural_earth_2_shaded_relief.raster.mbtiles](https://github.com/lukasmartinelli/naturalearthtiles/releases/download/v1.0/natural_earth_2_shaded_relief.raster.mbtiles)


For the OSM Data I am using geofabrik:

* [https://download.geofabrik.de/europe/germany/nordrhein-westfalen/muenster-regbez.html](https://download.geofabrik.de/europe/germany/nordrhein-westfalen/muenster-regbez.html)

And you can use these Vector Tiles of Münster (Germany):

* [https://data.maptiler.com/downloads/europe/germany/nordrhein-westfalen/muenster-regbez/](https://data.maptiler.com/downloads/europe/germany/nordrhein-westfalen/muenster-regbez/)

### Configuration ###

In the `.env` file adjust the PBF path: 

```ini
PBF_LOCAL_FOLDER=C:\Users\philipp\Downloads
PBF_FILENAME=muenster-regbez-260102.osm.pbf
```

In the `.config/api/appsettings.json` define the tilemaps to be used:

```json
{
  "Application": {
    "Tilesets": {
      "openmaptiles": {
        "Filename": "/pgrouting-tiles/osm-2020-02-10-v3.11_nordrhein-westfalen_muenster-regbez.mbtiles",
        "ContentType": "application/vnd.mapbox-vector-tile"
      },
      "natural_earth_2_shaded_relief.raster": {
        "Filename": "/pgrouting-tiles/natural_earth_2_shaded_relief.raster.mbtiles",
        "ContentType": "image/png"
      }
    }
  }
}
```

In the `.config/client/assets/appsettings.json` define the API Url and the Style to use:

```json
{
  "apiUrl": "https://localhost:5000",
  "mapOptions": {
    "mapStyleUrl": "https://localhost:5000/style/osm_liberty/osm_liberty.json",
    "mapInitialPoint": {
      "lng": 7.628202,
      "lat": 51.961563
    },
    "mapInitialZoom": 10
  }
}
```

In the `.config/client/assets/style/osm_liberty/osm_liberty.json` you'll need to setup the sources to match the API:

```json
{
  "sources": {
    "ne_2_hr_lc_sr": {
      "tiles": [
        "https://localhost:5000/tiles/natural_earth_2_shaded_relief.raster/{z}/{x}/{y}"
      ],
      "type": "raster",
      "tileSize": 256,
      "maxzoom": 6
    },
    "openmaptiles": {
      "type": "vector",
      "tiles": [
        "https://localhost:5000/tiles/openmaptiles/{z}/{x}/{y}"
      ],
      "minzoom": 0,
      "maxzoom": 14
    }
  }
}
```

### Running the Application ###

Then you can start the application with:

```
docker-compose --profile dev up
```

Go to `https://localhost:5001` and you can start to browse the map and calculate routes.


