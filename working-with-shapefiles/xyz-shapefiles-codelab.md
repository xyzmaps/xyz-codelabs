---
id: working-with-shapefiles
summary: importing shapefiles into XYZ
categories: XYZ CLI
tags: tutorial, XYZ, CLI
difficulty: 3
status: draft
feedback_url: https://here.com
published: 2019-03-15
author: XYZ Team

---

# Importing shapefiles into XYZ

[Shapefiles](https://en.wikipedia.org/wiki/Shapefile) are a proprietary but common geospatial file format developed by ESRI. It is frequently used by governments to store geospatial data. 

Many shapefiles can be easily uploaded into an XYZ space. Some require extra steps before you can bring it into XYZ.

In this tutorial, we'll cover what you need to do to successfully import shapefiles, along with special steps using other open source tools for those trickier ones.

This codelab assumes:

- you have already have a free [HERE developer account](https://developer.here.com/)
- you have installed the [HERE XYZ CLI](https://codelabs.here.xyz/tutorial/01-installing-the-here-cli#0)
- you have reviewed the [Using the CLI](https://codelabs.here.xyz/tutorial/02-using-the-xyz-cli) codelab
 
You should also install
- [mapshaper](https://github.com/mbloch/mapshaper)
- [QGIS](https://www.qgis.org/) and the [HERE XYZ QGIS plugin](https://plugins.qgis.org/plugins/XYZHubConnector/)

## Standard shapefile upload via the HERE XYZ CLI
Duration: 5:00

Unlike a GeoJSON file, a shapefile is made up of a number of separate files. Shapefiles on the internet are usually zipped, but once uncompressed you will see a number of files with the same name but different extensions. Some of the more important ones are:

- `.shp` - contains the geometries of the features (points, lines, polygons)
- `.dbf` - contains the attributes for the features 
- `.prj` - contains information aboute the projection and coordinate reference system (CRS)

If the shapefile uses lat/lon coordinates and the WGS84 projection, and is under 200MB, you should be able to upload it using the HERE XYZ CLI. 

In the terminal, `cd` to the shapefile directory, and type

	here xyz upload space_id -f my_shapefile.shp
	
The CLI will look for `my_shapefile.dbf` and other files in the specified directory. (If it is missing, no attributes of the geometries will be imported.)

Note that you can use `-a` to select attributes of features to convert into tags, which will let you filter features server-side when you access the XYZ Hub API.

## Advanced shapefile upload

Shapefiles are an infinitely variable format, and there will be cases where you may need to manipulate or modify the data in order to import it into your XYZ space. You can do this with other open-source geospatial tools, specifically `mapshaper` and QGIS.

### mapshaper

`mapshaper` is a powerful command-line tool for editing and manipulating geospatial data in a variety of common formats.

	https://github.com/mbloch/mapshaper
	https://github.com/mbloch/mapshaper/wiki/Command-Reference
	
You can install it using `npm`:
	
	npm install -g mapshaper
	
Note that `mapshaper` can modify shapefiles directly, or convert shapefiles into GeoJSON. Converting to GeoJSON will give you more options and faster uploads when bringing the data into XYZ. The [mapshaper documentation](https://github.com/mbloch/mapshaper/wiki/Command-Reference) provides a wide variety of options, but a simple conversion command is:

	mapshaper my_geodata.shp -o my_geodata.geojson
	
(Note that you can also specify `-o format=geojson` but `mapshaper` will also attempt use the extension of the output filename to determine the format.)
		

### HERE XYZ QGIS plugin

QGIS is an open source desktop GIS tool that lets you edit, visualize, manage, analyse and convert geospatial data. You can upload and download data from your XYZ spaces using the [HERE XYZ QGIS plugin](https://plugins.qgis.org/plugins/XYZHubConnector/). (The plugin is also available [on Github](https://github.com/heremaps/xyz-qgis-plugin).)

You can install the HERE XYZ QGIS plugin from within QGIS Plugin search tool if you have the "show experimental plugins" option checked in the plugin console settings.

![experimental](qgis_plugin_experimental.png)

You can easily open almost any shapefile in QGIS, at which point you can save it to your XYZ spaces using the HERE XYZ QGIS plugin, or export it as GeoJSON to the desktop to use the HERE XYZ CLI streaming upload options.


## Large individual features

Some shapefiles may contain very large and extremely detailed individual lines or polygons. If a single feature is greater than 10-20MB, you may see `400` or `413` http errors when you try to upload the shapefile. In many cases, this level of detail is unnecessary for web mapping. If so, you can try to simplify the feature using `mapshaper` or QGIS. You may also want to adjust HERE XYZ CLI upload parameters so less data is sent in each API request.

### Adjusting 'chunk' parameters

In order to optimize upload speed, the CLI "chunks" features together and then sends the chunk to the CLI. There are typically 200-400 features per chunk. While a large feature may be small enough to be uploaded, when combined with other features, it may be too large for the API.

You can adjust the chunk size using `-c` -- in this example, the CLI will upload 100 features per API request:
	
	here xyz upload spaceID -f large_features.shapefile -c 100

Depending on the size of the feature, you may want to try `c -10` (ten per request) or `c -1` (one at a time).

### mapshaper

You can simplify lines and polygons in shapefiles using `-simplify`.

	mapshaper very_large_features.shp -simplify dp 20% -o simplified_features.geojson
	
Depending on the zoom level and extent your web map (think the border of France at zoom 10 vs zoom 3), you can also try `10%`, `5%`, and `1%`.
	
More information on simplification is available here: https://github.com/mbloch/mapshaper/wiki/Command-Reference#-simplify

Note that for smaller shapefiles you can pipe output from `mapshaper` directly to the HERE XYZ CLI.

	mapshaper big_shapefile.shp -o format=geojson - | here xyz upload spaceID -p property_name -t specific_tag -s
	
In this case, you must specify the output format as `format=geojson` as there is no filename extension for `mapshaper` to reference. The `-` enables `stout`.
	
### QGIS

- open the shapefile in QGIS
- choose Vector -> Geometry Tools -> Simplify
- save the simplified data to a new XYZ space using the HERE XYZ plugin

Note that the Simplify tool works in decimal degrees, and the default is 1 degree, which is probably not what you want. Useful values depend on the extent and zoom levels of your map, but `0.01`, `0.001` and `0.0001` are interesting values.


## Very large shapefiles (> 200MB)

The HERE XYZ CLI will attempt to load the entire shapefile into memory before uploading it to the API. This will generally work for shapefiles up to 200-300MB, but you will start to see Node.js memory errors for shapefiles larger than that.

While GeoJSON and CSVs can be streamed via the `upload -s` option, this option is not yet available for shapefiles. You will have the most success converting the shapefile to GeoJSON and then uploading to XYZ.

	mapshaper big_data.shp -o format=geojson big_data.geojson
	here xyz upload spaceID -f big_data.geojson -s
	
_Note that `-a` is not available when `-s` is used, but you can still specify properties to convert into tags using `-p`._

You can also open the very large shapefile in QGIS and save directly to an XYZ space using the XYZ QGIS plugin, though this will be slower than using the CLI streaming feature.
	
## Projections and CRS (Coordinate Reference Systems)

Just like standards, the beauty of projections is there are so many to choose from. GeoJSON expects points to be projected in Web Mercator (WGS84/EPSG:4326). Many shapefiles are in different projections, or use local projections without lat/lon coordinates (i.e. state plane). Fortunately, it is easy to get `mapshaper` to convert into GeoJSON-friendly coordinates.

	mapshaper different_projection.shp -proj wgs84 -o format=geojson - | here xyz upload spaceID -p property_name -t specific_tag -s
	
If you see any node.js memory errors, you can break it up into two steps:

	mapshaper different_projection.shp -proj wgs84 -o format=geojson different_projection.geojson
	here xyz upload spaceID -f different_projection.geojson
	


