# wBim Format Specification 0.0.1

Building Information Model content is often created with a diverse set of tools, each with a proprietary format optimized for **authoring**  the content. This hampers their ability to be shared and viewed easily on any device and in low connectivity environments. While we recognize that tool specific features can sculpt a format intended to be written to, and by necessity make it proprietary, a universal and open *viewing* format is needed for the building industry to effectively collaborate across disciplines.

This format sets out with the assumption that a effective universal viewing format should be:
- Open source and permissively licensed. Developer adoption is paramount. 
- Compact and modular structure, intended for transfer over the wire and viewing on the web.
- Simple in concept, and at least partially human readable in implementation.
- Fast to render, not requiring the parametric calculations that make BIM authoring easier but display slower. 

The glTF format created and maintained by Khronos Group meets most of these requirements as provided, however to meet the needs of BIM users a few additional requirements are added:
- Allow complete access to model metadata and relate metadata to geometry on screen.
- Embrace IoT technologies and allow them to interact with model and be interacted with.

To those ends a new format, wBIM, is created as an extension of the glTF specification, made specifically for the building industry.

## What it does

- Allows anyone, on any device, in any location to view even very large BIM models, interact with their data, and obtain real time information from their real world counterparts.
- Provides an open specification for implementation in exporters for all BIM authoring tools.

## What it doesn't do

- wBIM is not a parametric modeling format. It is not intended for authoring new BIM components.
- Make you lunch

## What it doesn't yet do (but soon!)
- Carry two dimensional drawing data like sheet sets.

# The Specification
## First: glTF
This format is an extension of the glTF format, as such it may help to familiarize yourself with glTF before beginning here. Find the latest glTF specification from Khronos Group's Github [here](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0)

## Additional Top Level Properties
The wBIM format extends glTF by adding the following top level properties:

### octree
A wBIM manifest may optionally provide a uri linking to a serialized octree of the model, allowing the client viewer to defer loading and control visibility by querying the octree with the camera frustum.
TODO: example of this.

### links
A wBIM manifest may contain zero or more _links_ that reference other wBIM or standard glTF files stored seperately from the manifest in question. Links are defined in a `links` array. Links have two required properties:
- name: a string providing a human readable name for the linked model.
- uri: the full uri for retrieving the asset (a .glb or .gltf or .wbim file).

Links may optionally include a bounding-box property `bbox` to aid in debugging and deferred loading. The bbox should be axis-aligned and in the format `[ maxX, maxY, maxZ, minX, minY, minZ ]`

The following example shows a `links` array:

```json
{
    "links": [
	{
		"name": "East Building MEP - Revit",
		"uri": "https://your.path.to/mep.wBIM",
		"bbox": [100,100,100,0,0,0]
	},
	{
		"name": "East Building Sculpture Concept - Sketchup",
		"uri": "https://your.path.to/sculpture.gltf"
	}
    ]
}
```

Links are referenced by a new `link` property added to a glTF Node, using the index of the desired link from the `links` array. The node may still contain all standard glTF properties, including matrix or TRS properties to locate the linked model within the parent model:

```json
{
	"nodes": [
		{
			"name": "MEP Link",
			"link": 0,
			"translation": [
				-125.25,
				-15.75,
				15
			]
		}
	]
}
```
### propertydb
By definition, a BIM model is distinguished from a general 3d model by containing metadata about its elements. To keep file sizes optimal for web transmission, and allow simple querying, the wBIM format accesses this metadata via a linked SQLite database.

The required top level property `propertydb` is an object containing two required properties:
- uri: the full uri for the properties database file OR the REST API for interacting with the database.
- type: either "static" or "dynamic".

The following is a `propertydb` delcaration:

```json
{
	"propertydb": {
		"uri": "http://localhost:8000/mepProperties.wBIMDb
		"type": "static"
	}
}
```
### IoT
To support the increasing use of Digital Twins and IoT devices, the top level `IoT` array contains a list of objects defining connections to resources containing live or queryable data. A sensor may specify  a websocket connection to stream from and / or a REST API to query. A sensor object has 3 properties:
- name (required): a human readable name for the sensor to display in a UI.
- stream (optional): a websocket connection for obtaining a stream of live data from the sensor.
- query (optional): a REST API endpoint for querying historical data from the sensor.

A node may reference zero or more IoT data connections in its `sensors` array property.

The following examples shows the usage of the `IoT` and `sensors` properties:

```json
{
	"IoT": [
		{
			"name": "Unit Temperature",
			"stream": "wss://...",
			"query": "https://..."
		}
	],
	"nodes": [
		{
			"name": "HVAC Unit 001",
			"mesh": 0,
			"sensors": [
				0
			]
		}
	]
}
```

## BIM Properties Database
The BIM Properties Database referenced by `propertydb` is a SQLite file containing two tables:

### PropertyValues
| Field | dtype | Notes |
| -- | -- | -- |
| element_id | integer | The node index of the element
| property_id | integer | The id of the property from the PropertyEnums table.
| value | string | The parsable value of the property.

### PropertyEnums
| Field | dtype | Notes | 
| -- | -- | -- |
| id | integer | A unique id. |
| name | string | The name of the property |
| dtype | string | The data type of the property |
| category | string | One of "Instance", "Type", or "Family" |
