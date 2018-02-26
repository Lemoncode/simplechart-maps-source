# Maps Source & Processing

This application is built around Maps as its foundation. Map data has been obtained from official sources, processed, transformed and optimized to be consumed efficiently in a web environment. Let's explain below the processing flow followed to obtain the final result and its format.

## Data Source

As a first approach, we will focus our scope to Spain, and thus we will need map data for Spain at 3 different levels: 
- Regions.
- Provinces.
- Municipalities.

[CNIG](https://www.cnig.es/) is the Spanish official source for Geographic Information. Maps are easily accessible and available for free and provide enough detail for our purposes. You can get them in 2 ways:
- Please go to [CNIG Download Center](http://centrodedescargas.cnig.es/CentroDescargas/equipamiento.do?method=mostrarEquipamiento) and refer to the product called `Líneas Límite Municipales`.
- Or, you can also donwload the sources from the repository [simplechart-maps-source](https://github.com/Lemoncode/simplechart-maps-source), folder `[CNIG] Spain - Original Source`.

## Data Description

**Source**:That source contains either boundaries or regions for the 3 levels mentioned before:
- Folders that start with `recintos_` contain multipolygon SHP files representing areas of the regions.
- Folders that start with `ll_` contain multiline SHP files representing boundaries of the regions. 

**Territories**: Canary island and the iberian peninsula come in separated SHP files:
- Folder with the tag `canarias` refer to the Canary Islands SHP.
- Folder with the tag `peninbal` refer to the Peninsula SHP.

**SRS**: Both, Canary and Peninsula, come in geographical coordinates (GCS/unprojected), but with different datum:
- Canary Island maps come in REGCAN95, a WGS84 compatible datum.
- Peninsula maps come in ETRS89.

**Feature Attribute Table**: Find below the attributes for each feature.
- INSPIREID: Temporary INSPIRE ID.
- COUNTRY: country ID (`ES` constant).
- NATLEVEL: hierarchy level (URL).
- NATCODE: national code ID.
- NAMEUNIT: name of the feature (string UTF-8).
- CODNUT1: NUT1 Level this feature belongs to. NUT1 corresponds to large regions.
- CODNUT2: NUT2 Level this feature belongs to. NUT2 corresponds to Autonomous Communities.
- CODNUT3: NUT3 Level this feature belongs to. NUT3 corresponds to Provinces.

As we can see, the format of the data is SHP, splitted in 2 different files for Canary Islands and the Peninsula, features come with plenty of attributes that we won't use and add unnecessary extra weight. So, a processing is mandatory to optimize the maps to be consumed by our application use case.


## Processing Workflow

A minimun knowledge of GIS concepts and tools is required to follow these steps. For any doubt, please contact us. 

- Let's start from the original map data that we saw in the previous section as our baseline.
- Discard the SHPs that represents boundaries. We are only interested in areas (multipolygon SHP files).
- Now repeat these steps for the 3 levels available (regions, provinces and municipalities):
  - [QGIS] Merge both SHP layers (Canary and Peninsula) into a single vector file. You can use your favorite GIS edition tool, we will do it with [QGIS](https://www.qgis.org/es/site/). The simplest solution is to copy/paste the Canary Island features into the peninsula layer. This method will keep features attribute table intact. Remember to activate OTF-Reprojection so the features coordinates are transformed accordingly.
  - [QGIS] Select EPSG:4326 as your OTF target coordinate system. Let's reproject all the features into GCS WGS84 datum. This is a widely used and supported coordinate system.
  - [QGIS] Save this resulting layer, merged and reprojected, as SHP format. This will be our SHP of reference. Name suggested: `Spain-[01/02/03]-[Regions/Provinces/Municipalities]-EPSG-4326.shp`.
  - [QGIS] Now, Remove unnecessary attributes, this will be specially useful in municipalities for a considerable space saving. For instance, we can unambiguously identify a feature with its NATCODE, although CODNUT is also desirable as it is a more friendly ID. CODENUT1, however, won't be useful for us. So we can let only NATCODE, CODENUT2, CODENUT3 and NAMEUNIT as our feature attributes.
  - [QGIS] Save it in GeoJSON format. This will be our GeoJSON of reference. Name suggested: `Spain-[01/02/03]-[Regions/Provinces/Municipalities]-EPSG-4326.geojson`.
  - Let's transform the format from GeoJSON to TopoJSON. Install globally the `topojson` library with npm like this:
    ```cmd
    npm install -g topojson
    ```
  - [geo2topo] Run the transformation with the following command, using the proper index and name for each case:
    ```cmd
    geo2topo regions=Spain-01-Regions-EPSG-4326.geojson > Spain-01-Regions-EPSG-4326.topojson
    ```
    ```cmd
    geo2topo provinces=Spain-02-Provinces-EPSG-4326.geojson > Spain-02-Provinces-EPSG-4326.topojson
    ```
    ```cmd
    geo2topo municipalities=Spain-03-Municipalities-EPSG-4326.geojson > Spain-03-Municipalities-EPSG-4326.topojson
    ```
  - [toposimplify] Now that we have created our maps in TopoJSON format, let's do an important optimization by simplifying the polygon shapes (reducing the number of points that conform them by a given threshold). For our application, where the maps are gonna be viewed as a whole, precision in the rendering is not that important. This optimization will bring huge space saving and will allow faster load times.

    ```cmd
    toposimplify -f -p 0.0001 Spain-01-Regions-EPSG-4326.topojson > Spain-01-Regions-EPSG-4326.MIN.topojson
    ```
    ```cmd
    toposimplify -f -p 0.0001 Spain-02-Provinces-EPSG-4326.topojson > Spain-02-Provinces-EPSG-4326.MIN.topojson
    ```
    ```cmd
    toposimplify -f -p 0.0001 Spain-03-Municipalities-EPSG-4326.topojson > Spain-03-Municipalities-EPSG-4326.MIN.topojson
    ```

You can find all these files already processed in the repository the repository [simplechart-maps-source](https://github.com/Lemoncode/simplechart-maps-source).
  
## Glossary

- QGIS: Free, Open Source and cross platform GIS desktop application to create, edit, visualize, analize and publish geospatial information.
- SRS: Spatial Reference System.
- GCS vs PCS: Geographical Coordinate System (unprojected) vs Projected Coordinate System.
- SHP: ESRI Shapefile. Vector format widely used in GIS.
- OTF-Reprojection: On The Fly Reprojection.
- GeoJSON: a format for enconding vector data structures (geometries of different types) in JSON notation.
- TopoJSON: an extension to GeoJSON that encodes topology rather than discrete geometries. Geometries may share a part with any other geometry but these parts are not duplicated and, therefore, redundancy is removed.



