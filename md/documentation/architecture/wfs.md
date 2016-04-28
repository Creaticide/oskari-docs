# WFS 2

## Short description

WFS 2 contains separate backend from other backend action routes and map portlet with own frontend bundle implementations. To build WFS 2 needs the oskari-server's oskari-base package. The communication between backend and frontend is done with Bayeux protocol supporting websocket with ajax fallback except image route. Service implements JSON API through Bayeux channels. Backend gets layer configurations and user permissions from the oskari-backend with HTTP GET requests if there is no data in redis.

## Dependencies

WFS2's front is built with Javascript as Oskari bundle that handles its inner communication with Oskari's events and requests and communication with server is mostly handled with CometD connection that tries first to initialize Websocket handshake and fallbacks to callback-polling that has cross-domain support. Backend is implemented with Java as an independent part of oskari-server meaning that some packages and classes that WFS2 transport uses are shared with other backend services.

XML Parsing is handled at the backend with Axiom that is a StAX (Streaming API for XML) Parser for Java and Geotools. Axiom was selected because of small memory print and fast XML processing especially with medium and large XML documents.

We decided to not use Geotools' libraries as much as we could. Geometry transformation, drawing and styling are done with Geotools. As of generating XML payload for WFS requests we use OGC Filter and it's configuration from Geotools. Even if we have developed our own XML Parser that outputs Geotools' own SimpleFeatureCollection and uses GML Parser to just parse geometries, we have support to use GML Parser for the whole WFS XML response parsing. Our own parser handles some special cases that aren't supported in SimpleFeature format, for example possibility to have features in features.

Jedis is our choice for handling our Redis cache connections. Criteria for choosing were simplicity and usability that come with Jedis. We have wrapped all the needed Jedis actions inside our own JedisManager that handles connections with JedisPool. Every action always gets a connection from the pool and releases it when the action is finalizing.

JSON handling is partly done with CometDs parameter handling but mostly with Jackson that is easy and fast Java JSON parser and data binder. All cached data in Redis are in JSON format that can be easily serialized back to usable Java objects.

WFS2 backend is hosted with Jetty. CometDs Websocket is supported from Jetty 7. WFS2 doesn't make any other restrictions for choosing platform than what CometD needs.

## Rendering and data request

By default WFS layers are rendered on a map and feature data is retreaved when layer is selected in Oskari application.

There are various setups to override this default behaviour
<table class="table table-striped">
	<tr>
		<th>Table</th>
		<th>column</th>
		<th>value</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>portti_wfs_layer</td>
    	<td>get_map_tiles</td>
    	<td>false (default true)</td>
    	<td>WFS-layer is not rendered on a map</td>
	</tr>
	<tr>
    	<td>portti_wfs_layer</td>
        <td>get_feature_info</td>
        <td>false (default true)</td>
        <td>Feature data is not retreaved for WFS-layer on a map</td>
    </tr>
    	<tr>
        	<td>oskari_maplayer</td>
            <td>attributes</td>
            <td>{"manualRefresh":true}</td>
            <td>Rendering WFS-layer and retreaving feature data is managed by user request in Oskari application</td>
        </tr>
</table>

## Interfaces

The channels have been separated so that all the information coming to the server goes through service channels and information to the clients are sent through normal channels. Service channels are sort of 'setters' and client channels are 'getters'. Also Bayeux protocol adds additional meta channels that send information about the client's connection state.

### Service routes

#### /image

Client can make a HTTP request to get an image in png format with this route. Mainly the images are fetched from cache and returned only if found except for highlight images with featureIds that will be rendered if not found from cache.

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>session</td>
		<td>String</td>
		<td>Liferay portlet's JSESSIONID</td>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>type</td>
		<td>String</td>
		<td>Image type eg. "normal" or "highlight". Defaults to "normal".</td>
	</tr>
	<tr>
		<td>style</td>
		<td>String</td>
		<td>Layer's style. Defaults to "default".</td>
	</tr>
	<tr>
		<td>srs</td>
		<td>String</td>
		<td>Spatial reference system eg. EPSG:3067</td>
	</tr>
	<tr>
		<td>bbox</td>
		<td>ArrayList&lt;Double&gt;</td>
		<td>Bounding box values in order: left, bottom, right, top</td>
	</tr>
	<tr>
		<td>zoom</td>
		<td>long</td>
		<td>zoom level</td>
	</tr>
	<tr>
		<td>featureIds</td>
		<td>ArrayList&lt;String&gt;</td>
		<td>FeatureIds that need to be highlighted</td>
	</tr>
	<tr>
		<td>width</td>
		<td>int</td>
		<td>map width in pixels</td>
	</tr>
	<tr>
		<td>height</td>
		<td>int</td>
		<td>map height in pixels</td>
	</tr>
</table>

##### Example

###### normal image (cached)

/image?session=test&layerId=216&style=default&srs=EPSG:3067&bbox=386560.0,6671360.0,389120.0,6673920.0&zoom=7

###### highlight image

/image?session=test&layerId=142&type=highlight&srs=EPSG:3067&bbox=375584.224,6677855.5745,377311.224,6678850.5745&zoom=10&featureIds=FI.KTJkii-PalstanTietoja-15254706720131115,FI.KTJkii-PalstanTietoja-15254745920131115,FI.KTJkii-PalstanTietoja-15254746020131115,FI.KTJkii-PalstanTietoja-15254845120131115,FI.KTJkii-PalstanTietoja-15254845620131115&width=1727&height=995

### Service channels

Service channels are implemented so that every channel does just one specific task. Almost every client event have their own service channel to send information to the server.

1. /service/wfs/init
2. /service/wfs/addMapLayer
3. /service/wfs/removeMapLayer
4. /service/wfs/setLocation
5. /service/wfs/setMapSize
6. /service/wfs/setMapLayerStyle
7. /service/wfs/setMapClick
8. /service/wfs/setFilter
9. /service/wfs/setMapLayerVisibility
10. /service/wfs/highlightFeatures
11. /service/wfs/setMapLayerCustomStyle

#### /service/wfs/init

Client sends the starting state to the server when the /meta/handshake is triggered. Inits the client's session in the server and starts the neccessary jobs for defined WFS layers.

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>session</td>
		<td>String</td>
		<td>Liferay portlet's JSESSIONID</td>
	</tr>
	<tr>
		<td>browser</td>
		<td>String</td>
		<td>Client's browser name</td>
	</tr>
	<tr>
		<td>browserVersion</td>
		<td>long</td>
		<td>Client's browser version number</td>
	</tr>
	<tr>
		<td>location.srs</td>
		<td>String</td>
		<td>Spatial reference system eg. EPSG:3067</td>
	</tr>
	<tr>
		<td>location.bbox</td>
		<td>ArrayList&lt;Double&gt;</td>
		<td>Bounding box values in order: left, bottom, right, top</td>
	</tr>
	<tr>
		<td>location.zoom</td>
		<td>long</td>
		<td>zoom level</td>
	</tr>
	<tr>
		<td>grid.rows</td>
		<td>int</td>
		<td>row count of bounds</td>
	</tr>
	<tr>
		<td>grid.columns</td>
		<td>int</td>
		<td>column count of bounds</td>
	</tr>
	<tr>
		<td>grid.bounds</td>
		<td>ArrayList&lt;ArrayList&lt;Double&gt;&gt;</td>
		<td>bounds of the tiles</td>
	</tr>
	<tr>
		<td>tileSize.width</td>
		<td>int</td>
		<td>tile width in pixels</td>
	</tr>
	<tr>
		<td>tileSize.height</td>
		<td>int</td>
		<td>tile height in pixels</td>
	</tr>
	<tr>
		<td>mapSize.width</td>
		<td>int</td>
		<td>map width in pixels</td>
	</tr>
	<tr>
		<td>mapSize.height</td>
		<td>int</td>
		<td>map height in pixels</td>
	</tr>
	<tr>
		<td>mapScales</td>
		<td>ArrayList&lt;Double&gt;</td>
		<td>map scales list which indexes are zoom levels</td>
	</tr>
	<tr>
		<td>layers</td>
		<td>Object</td>
		<td>keys are maplayer_ids (long) and values include definition of style</td>
	</tr>
</table>

##### Example

```javascript
{
		"session" : "87D1AB34CEEFEA59F00D6918406C79EA",
		"browser" : "safari",
		"browserVersion" : 537,
		"location": {
				"srs": "EPSG:3067",
				"bbox": [385800, 6690267, 397380, 6697397],
				"zoom": 8
		},
		"grid": {
				"rows": 5,
				"columns": 8,
				"bounds": [[345600,6694400,358400,6707200]..]
		},
		"tileSize": {
				"width": 256,
				"height": 256
		},
		"mapSize": {
				"width": 1767,
				"height": 995
		},
		"mapScales": [5669294.4, 2834647.2, 1417323.6, 566929.44, 283464.72, 141732.36, 56692.944, 28346.472, 11338.5888, 5669.2944, 2834.6472, 1417.3236, 708.6618],
		"layers": { 216: { "styleName": "default" } }
}
```

##### Response channels

- /wfs/image
- /wfs/properties
- /wfs/feature
- /error


#### /service/wfs/addMapLayer

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>styleName</td>
		<td>String</td>
		<td>loaded style for the given WFS layer</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId": 216,
		"styleName": "default"
}
```

##### Response channels

Doesn't return anything


#### /service/wfs/removeMapLayer

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId": 216
}
```

##### Response channels

Doesn't return anything


#### /service/wfs/setLocation

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>srs</td>
		<td>String</td>
		<td>Spatial reference system eg. EPSG:3067</td>
	</tr>
	<tr>
		<td>bbox</td>
		<td>ArrayList&lt;Double&gt;</td>
		<td>Bounding box values in order: left, bottom, right, top</td>
	</tr>
	<tr>
		<td>zoom</td>
		<td>long</td>
		<td>zoom level</td>
	</tr>
	<tr>
		<td>grid.rows</td>
		<td>int</td>
		<td>row count of bounds</td>
	</tr>
	<tr>
		<td>grid.columns</td>
		<td>int</td>
		<td>column count of bounds</td>
	</tr>
	<tr>
		<td>grid.bounds</td>
		<td>ArrayList&lt;ArrayList&lt;Double&gt;&gt;</td>
		<td>bounds of the tiles</td>
	</tr>
	<tr>
		<td>tiles</td>
		<td>ArrayList&lt;ArrayList&lt;Double&gt;&gt;</td>
		<td>bounds of tiles to render</td>
	</tr>
	<tr>
       		<td>location.manualRefresh</td>
      		<td>boolean</td>
     		<td>reder the requested layer and get wfs data, if wfs layer mode is manual refresh</td>
    </tr>
</table>

##### Example

```javascript
{
		"layerId": 216,
		"srs": "EPSG:3067",
		"bbox": [385800, 6690267, 397380, 6697397],
		"zoom": 8,
		"grid": {
				"rows": 5,
				"columns": 8,
				"bounds": [[345600,6694400,358400,6707200]..]
		},
		"tiles": [[345600,6694400,358400,6707200]..]
}
```

##### Response channels

- /wfs/image
- /wfs/properties
- /wfs/feature
- /error


#### /service/wfs/setMapSize

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>width</td>
		<td>int</td>
		<td>map width in pixels</td>
	</tr>
	<tr>
		<td>height</td>
		<td>int</td>
		<td>map height in pixels</td>
	</tr>
	<tr>
		<td>grid.rows</td>
		<td>int</td>
		<td>row count of bounds</td>
	</tr>
	<tr>
		<td>grid.columns</td>
		<td>int</td>
		<td>column count of bounds</td>
	</tr>
	<tr>
		<td>grid.bounds</td>
		<td>ArrayList&lt;ArrayList&lt;Double&gt;&gt;</td>
		<td>bounds of the tiles</td>
	</tr>
</table>

##### Example

```javascript
{
		"width": 1767,
		"height": 995,
		"grid": {
				"rows": 5,
				"columns": 8,
				"bounds": [[345600,6694400,358400,6707200]..]
		}
}
```

##### Response channels

Only changes session information about the map size. No response.


#### /service/wfs/setMapLayerStyle

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>styleName</td>
		<td>String</td>
		<td>loaded style for the given WFS layer</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId": 216,
		"styleName": "default"
}
```

##### Response channels

- /wfs/image
- /error


#### /service/wfs/setMapClick

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>longitude</td>
		<td>double</td>
		<td>389800</td>
	</tr>
	<tr>
		<td>latitude</td>
		<td>double</td>
		<td>6693387</td>
	</tr>
	<tr>
		<td>keepPrevious</td>
		<td>boolean</td>
		<td>if keeps the previous selections</td>
	</tr>
</table>

##### Example

```javascript
{
		"longitude" : 389800,
		"latitude" : 6693387,
		"keepPrevious": false
}
```

##### Response channels

Doesn't send images

- /wfs/mapClick
- /error


#### /service/wfs/setFilter

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>filter.geojson</td>
		<td>GeoJSON</td>
		<td>http://www.geojson.org/geojson-spec.html</td>
	</tr>
</table>

##### Example

```javascript
{
		"filter" : { "geojson": {..} }
}
```

##### Response channels

Doesn't send images

- /wfs/filter
- /error


#### /service/wfs/setMapLayerVisibility

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>visible</td>
		<td>boolean</td>
		<td>if layer is visible</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId" : 216,
		"visible" : true
}
```

##### Response channels

Sends updated features and image if the layer's visibility has changed to true

- /wfs/image
- /wfs/properties
- /wfs/feature
- /error


#### /service/wfs/highlightFeatures

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>featureIds</td>
		<td>ArrayList&lt;String&gt;</td>
		<td>selected feature ids of the given WFS layer</td>
	</tr>
	<tr>
		<td>keepPrevious</td>
		<td>boolean</td>
		<td>if keeps the previous selections</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId" : 216,
		"featureIds": ["toimipaikat.6398"],
		"keepPrevious": false
}
```

##### Response channels

Sends only image

- /wfs/image
- /error


#### /service/wfs/setMapLayerCustomStyle

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>fill_color</td>
		<td>String</td>
		<td>area fill color</td>
	</tr>
	<tr>
		<td>fill_pattern</td>
		<td>int</td>
		<td>area fill style</td>
	</tr>
	<tr>
		<td>border_color</td>
		<td>String</td>
		<td>area line color</td>
	</tr>
	<tr>
		<td>border_linejoin</td>
		<td>String</td>
		<td>area line corner</td>
	</tr>
	<tr>
		<td>border_dasharray</td>
		<td>String</td>
		<td>area line style</td>
	</tr>
	<tr>
		<td>border_width</td>
		<td>int</td>
		<td>area line width</td>
	</tr>
	<tr>
		<td>stroke_linecap</td>
		<td>String</td>
		<td>line cap</td>
	</tr>
	<tr>
		<td>stroke_color</td>
		<td>String</td>
		<td>line color</td>
	</tr>
	<tr>
		<td>stroke_linejoin</td>
		<td>String</td>
		<td>line corner</td>
	</tr>
	<tr>
		<td>stroke_dasharray</td>
		<td>String</td>
		<td>line style</td>
	</tr>
	<tr>
		<td>stroke_width</td>
		<td>int</td>
		<td>line width</td>
	</tr>
	<tr>
		<td>dot_color</td>
		<td>String</td>
		<td>point color</td>
	</tr>
	<tr>
		<td>dot_shape</td>
		<td>int</td>
		<td>point shape</td>
	</tr>
	<tr>
		<td>dot_size</td>
		<td>int</td>
		<td>point size</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId":216,
		"fill_color":"ffde00",
		"fill_pattern":-1,
		"border_color":"000000",
		"border_linejoin":"mitre",
		"border_dasharray":"",
		"border_width":1,
		"stroke_linecap":"butt",
		"stroke_color":"3233ff",
		"stroke_linejoin":"mitre",
		"stroke_dasharray":"",
		"stroke_width":1,
		"dot_color":"000000",
		"dot_shape":1,
		"dot_size":3
}
```

##### Response channels

Doesn't return anything


### Client channels

Client channels are used to send information from the server to the client. Most of the service channels trigger client channel sends.

1. /wfs/image
2. /wfs/properties
3. /wfs/feature
4. /wfs/mapClick
4. /wfs/filter
5. /error

#### /wfs/image

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>srs</td>
		<td>String</td>
		<td>Spatial reference system eg. EPSG:3067</td>
	</tr>
	<tr>
		<td>bbox</td>
		<td>ArrayList&lt;Double&gt;</td>
		<td>Bounding box values in order: left, bottom, right, top</td>
	</tr>
	<tr>
		<td>zoom</td>
		<td>long</td>
		<td>zoom level</td>
	</tr>
	<tr>
		<td>type</td>
		<td>String</td>
		<td>"normal" or "highlight"</td>
	</tr>
	<tr>
		<td>keepPrevious</td>
		<td>boolean</td>
		<td>if keeps the previous image</td>
	</tr>
	<tr>
		<td>width</td>
		<td>int</td>
		<td>image width in pixels</td>
	</tr>
	<tr>
		<td>height</td>
		<td>int</td>
		<td>image height in pixels</td>
	</tr>
	<tr>
		<td>data</td>
		<td>String</td>
		<td>base64 data of the image</td>
	</tr>
	<tr>
		<td>url</td>
		<td>String</td>
		<td>resource url of the image</td>
	</tr>
</table>

##### Examples

###### for browsers that support base64 images

```javascript
{
		"layerId": 216,
		"srs": "EPSG:3067",
		"bbox": [385800, 6690267, 397380, 6697397],
		"zoom": 7,
		"type": "normal",
		"keepPrevious": false,
		"width": 1767,
		"height": 995,
		"url": <resourceURL>,
		"data": <base64Data>
}
```

###### other browsers

```javascript
{
		"layerId": 216,
		"srs": "EPSG:3067",
		"bbox": [385800, 6690267, 397380, 6697397],
		"zoom": 7,
		"type": "normal",
		"keepPrevious": false,
		"width": 1767,
		"height": 995,
		"url": <resourceURL>
}
```

#### /wfs/properties

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>fields</td>
		<td>ArrayList&lt;String&gt;</td>
		<td>feature property names of the given WFS layer</td>
	</tr>
	<tr>
		<td>locales</td>
		<td>ArrayList&lt;String&gt;</td>
		<td>feature property localizations of the given WFS layer</td>
	</tr>
</table>

##### Examples

```javascript
{
		"layerId" : 216,
		"fields": ["__fid",  "metaDataProperty",  "description",  "name",  "boundedBy",  "location",  "tmp_id",  "fi_nimi",  "fi_osoite",  "kto_tarkennus",  "postinumero",  "kuntakoodi",  "fi_url_1",  "fi_url_2",  "fi_url_3",  "fi_sposti_1",  "fi_sposti_2",  "fi_puh_1",  "fi_puh_2",  "fi_puh_3",  "fi_aoa_poik",  "fi_est",  "fi_palvelu_t",  "palveluluokka_2",  "org_id",  "se_nimi",  "se_osoite",  "se_url_1",  "se_url_2",  "se_url_3",  "se_sposti_1",  "se_sposti_2",  "se_puh_1",  "se_puh_2",  "se_puh_3",  "se_aoa_poik",  "se_est",  "se_palvelu_t",  "en_nimi",  "en_url_1",  "en_url_2",  "en_url_3",  "en_sposti_1",  "en_sposti_2",  "en_puh_1",  "en_puh_2",  "en_puh_3",  "en_aoa_poik",  "en_est",  "en_palvelu_t",  "x",  "y",  "the_geom",  "alku",  "en_osoite"],
		"locales": null
}
```


#### /wfs/feature

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>feature</td>
		<td>ArrayList&lt;Object&gt;</td>
		<td>feature values of the given WFS layer</td>
	</tr>
</table>

##### Examples

###### normal send

```javascript
{
		"layerId" : 216,
		"feature": ["toimipaikat.6398",  null,  null,  null,  null,  null,  6398,  "Yhteispalvelu Vantaa - Korso",  "Urpiaisentie 14",  "",  "01450",  "092",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  518,  811,  "Yhteispalvelu Vantaa - Korso",  "Urpiaisentie 14",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "Yhteispalvelu Vantaa - Korso",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  393893,  6692163,  "POINT (393893 6692163)",  "2013-01-04 10:01:52.202",  "Urpiaisentie 14"]
}
```

###### empty send

```javascript
{
		"layerId" : 216,
		"feature": "empty"
}
```

###### sent before of maxed feature search

```javascript
{
		"layerId" : 216,
		"feature": "max"
}
```


#### /wfs/mapClick

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>features</td>
		<td>ArrayList&lt;ArrayList&lt;Object&gt;&gt;</td>
		<td>selected features of the given WFS layer</td>
	</tr>
	<tr>
		<td>keepPrevious</td>
		<td>boolean</td>
		<td>if keeps the previous selections</td>
	</tr>
</table>

##### Examples

###### normal send

```javascript
{
		"layerId" : 216,
		"features": [
				["toimipaikat.6398",  null,  null,  null,  null,  null,  6398,  "Yhteispalvelu Vantaa - Korso",  "Urpiaisentie 14",  "",  "01450",  "092",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  518,  811,  "Yhteispalvelu Vantaa - Korso",  "Urpiaisentie 14",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "Yhteispalvelu Vantaa - Korso",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  393893,  6692163,  "POINT (393893 6692163)",  "2013-01-04 10:01:52.202",  "Urpiaisentie 14"],
				["toimipaikat.14631",  null,  null,  null,  null,  null,  14631,  "Kela, Vantaa / Korson yhteispalvelu",  "Urpiaisentie 14",  "",  "01450",  "092",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  503,  802,  "Kela, Vantaa / Korson yhteispalvelu",  "Urpiaisentie 14",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "Kela, Vantaa / Korson yhteispalvelu",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  393893,  6692163,  "POINT (393893 6692163)",  "2012-11-15 10:57:39.382",  "Urpiaisentie 14"]
		],
		"keepPrevious": false
}
```

###### empty send

```javascript
{
		"layerId" : 216,
		"features": "empty",
		"keepPrevious": false
}
```



#### /wfs/filter

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>features</td>
		<td>ArrayList&lt;ArrayList&lt;Object&gt;&gt;</td>
		<td>selected features of the given WFS layer</td>
	</tr>
</table>

##### Examples

###### normal send

```javascript
{
		"layerId" : 216,
		"features": [
				["toimipaikat.6398",  null,  null,  null,  null,  null,  6398,  "Yhteispalvelu Vantaa - Korso",  "Urpiaisentie 14",  "",  "01450",  "092",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  518,  811,  "Yhteispalvelu Vantaa - Korso",  "Urpiaisentie 14",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "Yhteispalvelu Vantaa - Korso",  "www.yhteispalvelu.fi",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  393893,  6692163,  "POINT (393893 6692163)",  "2013-01-04 10:01:52.202",  "Urpiaisentie 14"],
				["toimipaikat.14631",  null,  null,  null,  null,  null,  14631,  "Kela, Vantaa / Korson yhteispalvelu",  "Urpiaisentie 14",  "",  "01450",  "092",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  503,  802,  "Kela, Vantaa / Korson yhteispalvelu",  "Urpiaisentie 14",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "Kela, Vantaa / Korson yhteispalvelu",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  "",  393893,  6692163,  "POINT (393893 6692163)",  "2012-11-15 10:57:39.382",  "Urpiaisentie 14"]
		]
}
```

###### empty send

```javascript
{
		"layerId" : 216,
		"features": "empty"
}
```


#### /error

##### Parameters

<table class="table table-striped">
	<tr>
		<th>Name</th>
		<th>Type</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>layerId</td>
		<td>long</td>
		<td>maplayer_id</td>
	</tr>
	<tr>
		<td>once</td>
		<td>boolean</td>
		<td>if should react only once</td>
	</tr>
	<tr>
		<td>message</td>
		<td>String</td>
		<td>key for error message (should be localized in frontend)</td>
	</tr>
</table>

##### Example

```javascript
{
		"layerId" : 216,
		"once": true,
		"message": "wfs_no_permissions"
}
```


### Meta channels

Meta channels are used in both frontend and backend to determine the state of the client's connection.

1. /meta/handshake
2. /meta/subscribe
3. /meta/connect
4. /meta/disconnect

#### /meta/handshake

Handshake of the connection. Tells needed information to the client about the connection. After handshake we can subscribe to channels.

#### /meta/subscribe

Information about subscriptions.

#### /meta/connect

Information about the current connections's state.

#### /meta/disconnect

Sent when disconnected. Used at the backend as a trigger to destroy user's session information.


## Structure

WFS 2 has a separate frontend and backend implementations. Backend is a standalone CometD Servlet (Java) and frontend is built inside Oskari as mapwfs2 bundle (JavaScript).

### mapwfs2 bundle [Frontend]

WfsLayerPlugin is the main component of the bundle. The functionalities are pretty much the same as in the old WFS implementation.

#### Initialization

The bundle doesn't have instance at all because it is started only as a mapfull plugin. Initialization is straight forwarded if the connection can be established without problems. Connection retrying is implemented so that with certain intervals and timeouts. If the connection can't be established it blocks the subscription and publishing init to the backend but the events will be still registered.

1. WfsLayerPlugin is started as a mapfull's plugin
2. Connects to the backend with the best viable connection type available
3. Subscribes to the client channels
4. Publishes initial information through '/service/wfs/init' channel to the backend
5. Registers to listen all the needed events

#### Events

Events can be triggered in any time after the registration. WfsLayerPlugin handles all the needed events for the bundle.

#### Channels

Client messages are listened after subscriptions. Mediator implements all the channel handling for client and service channels. Meta channels are handled in Connection.


### transport [Backend]

#### Class dependencies

![backend_uml](/images/architecture/wfs_backend_uml.png)

#### Initialization

When the servlet is started it initializes Bayeux protocol and adds services that uses it, Redis connection and saved schemas. A service is hooked in a channel which has a linked processing method. Every channel goes through TransportService's processRequest method that has the basic parameter, session handling and forwarding requests to channel specific methods.

#### Channels

The servlet listens to service channels and responses with appropriate client channels. Every sent client channel message links to some service channel publish of the client. So servlet doesn't send any information without a request from client.


## User stories

Possible user actions are basicly frontend events that mapwfs2 handles. Event and request sequences can be traced with following lines in browser's console. Activity of the channels aren't shown in debug. The sequence images in the user stories are simplified.

```javascript
Oskari.$("sandbox").enableDebug();
Oskari.$("sandbox").popUpSeqDiagram();
```

### Moving map - AfterMapMoveEvent

![map_move](/images/architecture/wfs_map_move.png)

Function listening to AfterMapMoveEvent calls for Mediator's setLocation(). Backend gets a message and updates every WFS layer that is in the user's session and answers with updated properties, features and images. Upcoming sends trigger WFSPropertiesEvents, WFSFeatureEvents and WFSImageEvents. Updates the properties and features for object data and draws new tiles.

### Resizing map - MapSizeChangedEvent

Function listening to MapSizeChangedEvent calls for Mediator's setMapSize() and setLocation(). Backend gets a message and saves new map size in user's session. Every WFS layer in user's session are updated. The update process is same than for moving map.

### Adding a WFS layer - AfterMapLayerAddEvent

Function listening to AfterMapLayerAddEvent calls for Mediator's addMapLayer() and calls AfterMapMoveEvent's handler. Backend gets a message and adds the WFS layer to the user's session and answers with the new layer's properties, features and images. Upcoming sends trigger WFSPropertiesEvent, WFSFeatureEvents and WFSImageEvents. Updates the properties and features for object data and draws new tiles.

### Removing a WFS layer - AfterMapLayerRemoveEvent

Function listening to AfterMapLayerRemoveEvent calls for Mediator's removeMapLayer(). Backend gets a message and removes the WFS layer from the user's session and removes layer's ongoing jobs if there is any. Backend doesn't send anything to the client. Layer is also removed from Openlayers.

### Selecting a feature - WFSFeaturesSelectedEvent

![feature_select](/images/architecture/wfs_feature_select.png)

Function listening to WFSFeaturesSelectedEvent calls for Mediator's highlightMapLayerFeatures(). Backend gets a message and draws the features with highlight SLD and sends a higlighted images to the client. Upcoming sends trigger WFSImageEvents. Highlighted image is shown on top of the WFS layer's tile image.

Possible to keep the previous selection by holding CTRL.

### Clicking map - MapClickedEvent

![map_click](/images/architecture/wfs_map_click.png)

Function listening to MapClickedEvent calls for Mediator's setMapClick(). Backend gets a message and collects features of the given location for every active layer. Upcoming sends trigger WFSPropertiesEvent, WFSFeatureEvents. MapClickedEvent triggers also WFSFeaturesSelectedEvent and GetInfoResultEvent and finally ShowInfoBoxRequest that creates a popup with all selected features' information.

Possible to keep the previous selection by holding CTRL.

### Changing a WFS layer's style - AfterChangeMapLayerStyleEvent

Function listening to AfterChangeMapLayerStyleEvent calls for Mediator's setMapLayerStyle(). Backend gets a message and draws the features with new SLD and sends images to the client. Upcoming sends trigger WFSImageEvents. Images with new style replace old WFS layer's tile images.

* TODO: Event throw (not implemented yet for WFS)

### Changing a WFS layer's visibility - MapLayerVisibilityChangedEvent

Function listening to MapLayerVisibilityChangedEvent calls for Mediator's setMapLayerVisibility(). Backend gets a message and reacts if there was a change in visibility. The change is saved into user's session and if the visibility is changed to true then the layer is updated in same way than when map moves. Upcoming sends trigger if visibility is changed to true are WFSPropertiesEvents, WFSFeatureEvents and WFSImageEvents. Updates the properties and features for object data and draws new tiles.

### Changing a WFS layer's opacity - AfterChangeMapLayerOpacityEvent

Function listening to MapLayerVisibilityChangedEvent updates Openlayer's layer opacity. This functionality doesn't communicate with backend at all.

### Refreshing/rendering manual WFS layer - WFSRefreshManualLoadLayersEvent

Calls for Mediator's setLocation() with isManualRefresh : true property. Render manual wfs layer and retreave feature data on that location.

### Finishing selection tool - WFSSetFilter

Function listening to WFSSetFilter calls for Mediator's setFilter(). Backend gets a message and collects features of the given location for active layer. Upcoming sends trigger WFSPropertiesEvent, WFSFeatureEvents. Opens object data flyout highlighting filtered features.


## Cache

Caching is done with redis on backend. Basic key-value storage is used in most of the cases exluding schemas which use hash storage.

### Storages

#### Sessions

* key: Session_#{client}
* unique part is Bayeux client id
* expires in one day

* created for every user
* most of the service channel messages update session information excluding setFilter, setMapClick and highlightMapLayerFeatures
* key is deleted when user disconnects (/meta/disconnect)
* session keys are deleted when service closes

#### WFS Layers

* key: WFSLayer_#{layerId}
* unique part is map layer id
* expires in one day

* created for all requested layers
* data is fetched from multiple database tables

### Permissions

* key: Permission_#{session}
* unique part is JSESSIONID
* expires in one day

* created for all users
* data is fetched from Liferay
* permissions keys are deleted when service closes

### Schemas

* key: hSchemas
* field: schema's URL
* doesn't expire

* data is copied from cache only once in init (time taking operation)

### WFS Tiles

* key: WFSImage_#{layerId}_#{srs}_#{bbox[0]}:#{bbox[1]}:#{bbox[2]}:#{bbox[3]}_#{zoom}
* unique part contains map layer id and location information
* expires in one week

* data is saved if the tile isn't colliding boundary

### WFS Temp Tiles

* key: WFSImage_#{layerId}_#{srs}_#{bbox[0]}:#{bbox[1]}:#{bbox[2]}:#{bbox[3]}_#{zoom}_temp
* unique part contains map layer id and location information
* expires in one week

* data is saved if the tile is colliding boundary or map image (because of IE)
