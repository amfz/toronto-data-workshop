---
marp: true
title: "Data Workshop"
author: "A. Mahfouz"
theme: default
footer: A Mahfouz
paginate: true
---

# Geographic Data in Python
  

A Mahfouz
Toronto Data Workshop  
25 June 2020

<!--
Hi, I'm A and I'll be breaking the R streak to talk a bit about working with geographic data in Python.
-->

---
# Agenda

- About me
- Project description
- Google Directions API: APIs, time, build-your-own-geojson
- OpenTripPlanner: more about spatial data representation, simple spatial operations

<!--
An outline of what I'll be talking about today: a little about me, to break the ice, a broad description of the work, and then I'll cover two workflows involved
-->

---
# About Me
- Master of Information student at the University of Toronto
- Geographer by training
- Still not used to writing about me sections

<!--
About me: Like Rohan said, I'm currently a master of information student at U of T. My background is in geography and history. I drifted into data stuff via GIS -- people would hire me to make maps.
-->

---

# Project Description

- Transit analysis for schools in multiple cities in the US
- What would their students' commutes look like?
- What areas of the city are within reasonable commuting distance via transit?

<!-- 
This talk draws from work doing transit analysis for some schools in the US. The specific questions and the work involved varied from school to school, but questions generally fell into two bins. The first involved details of individual commutes: if students took public transit, how long would the trip take, how many times would they have to transfer, how far would they have to walk. The second category focused on general transit accessibility: questions of what parts of the city are within reasonable commutes via transit. What constituted 'reasonable' could vary.
-->

--- 
# Toolset

- Python
- Google Directions API
- OpenTripPlanner
- QGIS

<!-- 
To answer these questions, we used the Google Directions API, OpenTripPlanner, Python, and QGIS, but all of this is doable in R, and frankly I think some aspects might be easier in R.
-->

---
# Skills/Concepts

- Working with APIs
- UNIX time and timezones :scream:
- Build-your-own dataset
- Spatial data representation
- Common file formats
- Basic spatial operations

<!-- 
Along the way, I'll touch upon a general Python workflow for getting data from APIs, with a detour into UNIX time and timezones. We'll build our own dataset from the results, which will require learning a little about spatial data representation and common file formats. And finally, we'll look at a simple spatial operation.
-->

---
# Inputs
### School data
| School Name | School Address | AM Bell Time | PM Bell Time|
|---|---|---|---|
|PS X | 601 N Mesa | 7:30 AM | 2:30 PM |  

### Student data
| ID | Grade | Address | Stop |
|---|---|---|---|
|123456789| 9 | 711 Hills | St Vrain & E Father Rahm

<!-- 
To quickly describe the inputs, we got data describing schools -- including school name, address, and bell times -- and datasets about students, with varying states of quality, so a lot of data cleaning and joining occurred even before getting to the API calling stage. Just as a note, none of the data shown here is actual student data, and most of the locations aren't even schools, either.
-->

---
# A Note on Geocoding
- AKA converting addresses into coordinates
- Unnecessary for the Directions API
- Google's geocoder is pretty error-tolerant
- Some other options: Nominatim, Mapbox, HERE

<!-- 
One common task I won't be going into is geocoding, aka converting addresses into coordinates. It isn't necessary for working with the Directions API, and you can actually extract coordinates from Directions API results.

But if it is necessary to geocode separately, Google's geocoder is quite error-tolerant. That error tolerance means we can pass in an intersection, or just a ZIP code, or fudge student addresses by incrementing the house numbers by a  random integer, and still get usable results.

Some other options include Nominatim, which is built off of OpenStreetMap, and private vendors like Mapbox or HERE.
-->

---
# Working with the Google Directions API
- We'll just use [`requests`](https://requests.readthedocs.io/en/master/)
  - Versatile library for HTTP requests
- There is a Python client ([`googlemaps`](https://github.com/googlemaps/google-maps-services-python)) but it doesn't return the full API response
- Request parameters
  - API key
  - `origin`
  - `destination`
  - `mode`
  - `departure_time` or `arrival_time`

<!-- 
As I mentioned, we used Google Directions to generate indiviual trips. It's not necessarily the best option, but it was chosen for two related reasons. One, people tend to take Google results for the truth, and two, the results are easy to verify. If anyone thinks a result looks funny, they can go and confirm it.

To make API calls, we used the requests library. It's a versatile python library for HTTP requests, so we actually used it to interface with OpenTripPlanner as well. There is a Python Google Maps client, but it doesn't return the full API response, and we wanted some of the pieces it leaves out.

If you've ever gotten directions from Google, you can guess the parameters needed to pass to the API. Besides the key, there's the origin and destination locations, which can be addresses.  To answer our questions, we also had to specify the travel mode -- transit here -- and either the arrival time for AM trips or the departure time for PM trips.
-->

---
# Do the Time Warp

- We have bell times as text (e.g., "7:30 AM") but need UNIX timestamps (seconds since midnight 1/1/1970 UTC)
- Specific travel dates aren't important, but constraints apply:
  - Must be a non-holiday weekday
  - Cannot be too far into the past or future
- Schools are in all different time zones

<!-- 
But there's a catch. The times need to be in UNIX time, which is the number of seconds since midnight on January 1, 1970, Coordinated Universal Time. The bell times we have are all in text. We don't have dates, and the specific date chosen doesn't matter too much. Since we're interested in commutes, and transit schedules differ on weekends and holidays, we need to use non-holiday weekdays. The dates can't be too far into the past or future, either. And, just for added fun, the schools are in different timezones.
-->

---
# Do the Time Warp
```python
from datetime import date, datetime
from dateutil import parser, tz
from dateutil.relativedelta import WE, relativedelta

def get_next_wednesday():
    today = date.today()
    delta = relativedelta(days=1, weekday=WE(1))
    next_wednesday = today + delta
    return next_wednesday


def create_timestamp(str_time, tz):
    """
    Args:
        str_time (str): Time to convert. Expects hours and minutes; seconds are optional.
          If AM/PM is not specified, a 24-hour clock is assumed.
        tz (str): 3-character timezone string.
    Returns:
        int: a UNIX timestamp (seconds from January 1, 1970 UTC) representing str_time, 
        next Wednesday from when the function is called.
    """

    tzinfo = {'EDT': tz.gettz('US/Mountain'),
              'CDT': tz.gettz('US/Central'),
              'MDT': tz.gettz('US/Mountain')}
    str_time = str_time + ' ' + tz
    n = get_next_wednesday()
    full_dt = str(n) + ' ' + str_time
    timestamp = datetime.timestamp(parser.parse(full_dt, tzinfos=tzinfo))
    return int(timestamp)
```

<!-- 
We didn't want to hard code UNIX dates into the script, so from the department of over-engineering came a couple of helper functions to turn bell times into that time next Wednesday. Wednesday was chosen just because there weren't any holidays coming up on Wednesdays. Shout-out to the dateutil library for doing the heavy lifting here.
-->

---
# Making the Call

```python
# simplified example code
def format_params(record, ampm):
    params = {'key': GKEY, 'mode': 'transit'}
    if ampm=='am':
        # add origin, destination, arrival time for AM
        params['origin'] = '{}'.format(record['home_address'])
        params['destination'] = '{}'.format(record['school_address'])
        # format arrival time to be school session time next Weds                                  
        params['arrival_time'] = create_timestamp(record['am_bell_time'], record['tz'])
    return params


def query_dir_api(params):
    data = {}
    r = requests.get(dir_api, params=params)
    if r.status_code == 200:
        data = r.json()
    return data


def batch_process(df, ampm):
    all_journeys = []
    for idx, row in df.iterrows():
        params = format_params(row, ampm)
        response = query_dir_api(params)
        all_journeys.append(response)
    return all_journeys
```

<!-- 
Finally, we're set to make API calls. I'm not going to dwell too long here -- it's a fairly standard workflow of iterating through a pandas DataFrame containing the data to make calls, formatting the parameters, making the request, parsing the results into json objects and writing them to file.
-->

---
### Caveat 1 
## Equivalent addresses and coordinates may yield different results
![100% bg vertical](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/directions.png) 
![100% bg right:55%](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/no_directions.png)

<!-- 
Before looking at the results, I want to mention two caveats. First, I got better results passing in addresses than coordinates, even when those coordinates came from Google. We geocoded the schools for something else and figured may as well use the coordinates because they're more precise, but the results ended up being worse. On the right is an extreme example replicated in the web interface. They're results for the same journey, the only difference is up top two addresses were supplied, while at the bottom I entered the coordinates I got from Google's geocoder for the same address.
-->

---
### Caveat 2
## Results may meet the letter but not the spirit of the request
![95% bg right:55%](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/pm_trip.png)

<!-- 
Second caveat is check all the trip attributes because the API really wants to return a result, even if it's not appropriate for your purpose. So here, we wanted to model a morning commute, arriving at the destination by 7:30 AM. Google did get a response....you just have to overnight it at your destination.
-->

---
# Google Directions Results

``` json
{
    "geocoded_waypoints": [...],
    "routes": [
        {
          "legs": [
               {
                   "arrival_time": {"text": "3:16pm", "time_zone": "America/Denver", "value": 1593641805},
                   "departure_time": {...},
                   "distance": {"text": "7.2 mi", "value": 11586},
                   "duration": {"text": "43 mins", "value": 2603},
                   "end_address": "6209 Weems Ct, El Paso, TX 79905, USA",
                   "end_location": {...},
                   "start_address": "601 N Mesa St, El Paso, TX 79901, USA",
                   "start_location": {...},
                   "steps": [...]
                }
            ],
            "overview_polyline": {
                 "points": "woz`Evy}hS~F_FfFgEb@e@zBiB@M`BuAe@yCm@sEu@wESoAIe@IMi@eAa@w@HIIHgAoBsCeF
                 iBgDk@b@_@ZeA~@qC|BwByDaRu\\_OyWmGaLuLiTqDwGg@aAGi@DaFD_DBoAHe@?UU@]B{A@yDDqB@]Cu@W
                 UOm@w@a@iAe@aAGM_@]m@[SEg@CmBAsOCwKCAbGCj@RbAn@lA|AbC}@t@{@yAKSRQK[Uu@MY@SDI?[Ac@j@
                 ?xC?jCD`AChED|G@zIBvDLtBBxHIDk@FeG?eAC@?uC@gKDqXF}X@yR?iC?o@AaD@}A@yCNoAZm@vAmBfAsA
                 `@o@Lo@De@`@iFh@mGRaCDUbAcEnAgFpA}FzAsIRaARkA\\_CdBcMJ[PUNKHJJDN?LELSBOAOEMGKE}@BkD
                 Hy@JIDM@SAOIORgBt@sFTqB^cDvAiKdBgNxA{K|@yGn@{E\\kCB@B@DYLy@YCQEALyBEoCG?eJAk@"
              }
        }
    ],
    "status": "OK"
}
```

<!-- 
So, when results come back successfully, they look something like this. 

You have the status at the bottom -- here it's okay, but if one of the points couldn't be found or there aren't any results, the status will reflect that. 

Up top are the geocoded waypoints. If you give addresses, this section will just have google's proprietary place IDs and some place categories. If you supply coordinates, they'll reverse geocode here. 

The key part is the routes section. By default, there will only be one route in the result. 

Within the route, the key pieces are an overview polyline representing the shape of the journey, and the legs attribute, which contains overall trip distance, duration, start and end locations, and steps, which break down the trip down further. Our trips only have a start and endpoint, with no intermediate stops, so there is only one leg.

In the leg, the steps attribute contains detailed information like what routes are taken, what turns to make, and which parts of the journey are walking, transit, etc.
-->

---
# Google Directions Results Components

- `status`
- `geocoded_waypoints`
  - Geocoder status message
  - If a location was found, Google's place ID for it
  - If coordinates were passed, the reverse geocoding result
- `routes`
  - By default only one route is returned
  - `legs`: itinerary information
  - `overview_polyline`: geographic shape

<!-- 

-->

---

# Great! But we can't map this.

<!-- 
All this is great, but we can't map this as-is. In order to get it to a mappable state, we need to talk about spatial data representation.
-->

---
 
# Simple Features
![90% bg right:45% vertical](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/sf_lines.png)
![90% bg](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/sf_multilines.png)
![90% bg](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/sf_polygons.png)
![90% bg](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/sf_multipolygons.png)

- [ISO/OGC standard](https://www.ogc.org/standards/sfa) for spatial data representation
- Geometry types
  - Point and Multipoint
  - LineString and MultiLineString
  - Polygon and MultiPolygon
  - GeometryCollection
- Most geographic information systems use this model
  - Notable exceptions: raster data, OpenStreetMap

<!-- 
The most common spatial data model is the simple features model. Simple features is an ISO/OGC standard for vector data -- geographic data that can be represented as discrete objects. 

Spatially, data can be modeled as points -- for example, cities -- 

lines -- e.g. streets or rivers -- 

or polygons, like park space or countries. 

The multi- variants are for entities like, say, Japan, that are best represented with more than one shape. 

Finally, a geometry collection is a catch-all for heterogenous shapes. 

No matter what though, geometries are represented with coordinate pairs, so that polyline string isn't going to cut it. 

Most geographic information systems and mapping software use this model, or a variant. Some big exceptions are raster data, so things like temperatures or elevations that are better modeled as continuous fields, and OpenStreetMap, which uses its own data model that better serves its purposes. Moving between OSM and simple features is a common enough task that there are several tools to help with the conversion. It's just good to know that that is a step.
-->


---

# geoJSON

  - [Standardized format](https://tools.ietf.org/html/rfc7946) for representing simple features

Skeleton:
  ```json
  {
    "type": "FeatureCollection",
    "features": [
      {
        "geometry": {
           "type": "LineString",
           "coordinates": [...]
          },
        "properties": {...}
      }
    ]
  }
  ```

<!-- 
There are file formats that are simple features compliant. We decided to convert the API results into geoJSON. 

GeoJSON, like JSON, is relatively easy to create programmatically. 

Up here is a basic skeleton -- we specify that the file is a feature collection, rather than a single feature. 

Then features contains a list of the features. Each feature consists of a geometry attribute, where we'll need to put in the coordinate pairs, and properties, where we'll put trip attributes like duration.
-->

---

  # Decoding Polylines
  - https://developers.google.com/maps/documentation/utilities/polylinealgorithm
  - `polyline` to the rescue

  ```python
  import polyline

  def extract_overview_line(api_response):
    route_line = api_response['routes'][0]['overview_polyline']['points']
    pts = polyline.decode(route_line, geojson=True)
    return pts
  ```

<!-- 
First, we need to get the polylines into coordinate pairs. Google documented the encoding algorithm, but instead of reinventing the wheel, we used the polyline library to write a decoding function.
-->

---
# Extracting Attributes

```python
def extract_properties(result):
    if len(result['routes']) > 0:
        leg = result['routes'][0]['legs'][0]
        start_address = leg['start_address']
        end_address = leg['end_address']
        # account for walking-only routes not having departure and arrival times
        dep_time = leg.get('departure_time', {}).get('text')
        arr_time = leg.get('arrival_time', {}).get('text')
        duration_minutes = round(leg['duration']['value'] / 60, 1)
        dist_miles = round(leg['distance']['value'] * 0.00062137, 2)
        properties = {'origin': start_address,
                      'dest': end_address,
                      'departure_time': dep_time,
                      'arrival_time': arr_time,
                      'total_minutes': duration_minutes,
                      'total_miles': dist_miles,
                      'notes': ''}
    else:   
        properties = {'notes': result['status']}
    return properties
```

<!-- 
Then, another function to extract the attributes,
-->

---
# Putting It All Together

```python
def make_feature(result):
    """Convert a Google Directions API result into a geoJSON feature."""
    obj = {'type': 'Feature'}
    if len(result['routes']) > 0:
        geom = extract_overview_line(result)
        obj['geometry'] = {'type': 'LineString',
                           'coordinates': geom}
    else:
        obj['geometry'] = {'type': 'GeometryCollection',
                           'coordinates': []}
    obj['properties'] = extract_properties(result)
    return obj


def gen_geojson(data):
    """Turn a list of Directions results into a geoJSON"""
    out_geojson = {
        "type": "FeatureCollection",
        "features": []
    }
    for journey in data:
        feature = make_feature(journey)
        out_geojson['features'].append(feature)

    return out_geojson
```

<!-- 
and finally pop geometries and properties into a feature and add it to the features list. From there we wrote them to file.
-->

---

# Results!
# 
# 
# 
# 
# 
# 
# 
# 
# 
# 

![100% bg](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/decoded_polylines.png)

<!-- 
So, for a much simpler dummy dataset, the geoJSONs look like this. From there, they can be put on a map and styled by whatever attribute is of interest. Here, trips that are more than half an hour long are orange, while shorter ones are purple. The trip to the southwest corner of the map is the polyline in the example result earlier.
-->

---
# That works for individual trips. What if I want something more general?

<!-- 
So that gets us mappable data for individual trips. But what if the question is more general? What if, instead of focusing on individual trips, we wanted to show what parts of the city are within reasonable commuting times? In other words, we need isochrones.
-->

---
# [OpenTripPlanner]()

![90% bg left:40% opacity:0.5](https://docs.opentripplanner.org/en/latest/otp-logo.svg)

- Open source multimodal routing software written in Java
- Works with OpenStreetMap and GTFS data
- APIs for routing and isochrone creation
- Excellent R resources: `otpr`, `opentripplanner`, [tutorial](https://github.com/marcusyoung/otp-tutorial)

<!-- 
Google doesn't have an API for that. Really, for transit isocrhones, low-cost API options are slim. So we went with OpenTripPlanner, which is open source, Java-based multimodal routing software. It works with OpenStreetMap data and GTFS, or generalized transit feed specification, data. It's meant to run server-side, so there are API endpoints for routing and isochrone creation, which means we can interact with it using requests. For the R users in the audience, there are excellent resources.
-->

---

##   
## 
#    
# 
# 
# 
# 
# 
# 
# 
# 
![bg 92%](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/am_isos.png)
![bg 92%](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/pm_isos.png)

<!-- 
Building the router and calling the API could be its own talk, so in the interest of time I'll cut to what the isochrones look like. These were morning and afternoon isochrones for 15-minute intervals from 30 to 90 minutes. There were some additional restrictions, like capping the amount of walking, but you can see that this city's transit system really wasn't designed for early school starts.
-->

---
# One Little Problem...

![img](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/polygon_error.png)

<!-- 
There is a hiccup here. When I handed the files off to a colleague to work with in QGIS, this happened. 

The error is a bit small, but it says that there is an invalid geometry.

It turns out that the isochrones look good, but they don't always comply with the simple feature specification. They sometimes do things polygons aren't supposed to do, like intersect themselves. 

When shapes don't follow the simple features spec, you can still map them, but you can get errors like this when you try to perform spatial operations.
-->


---
# [Geopandas](https://geopandas.org/index.html) to the Rescue

- Python library for spatial data manipulation
- Think `pandas` with simple features support
  - Read/write spatial data files
  - Project coordinate reference systems
  - Perform spatial operations

<!-- 
The problem is fixable within QGIS, but you don't want to do this for hundreds of files. 

Fortunately, there's a hacky-but-simple fix in Python. We'll need to geopandas library to do this. Geopandas essentially extends pandas and its data frame model to provide support for simple features -- points, lines, and polygons. 

It's useful for reading and writing spatial data files, managing coordinate reference systems, and performing spatial operations. 

Just a quick aside about spatial reference systems -- they're important, but because all the data here is meant to work on web maps, they have the same reference system.
-->

---
# Geopandas DataFrames

```python
import geopandas as gpd

isos = gpd.read_file('pm_isochrones.geojson')
print(isos)
```

```
                               id  time                                           geometry
0  fid-1681139f_172e865dd6a_-7ffc  5400  MULTIPOLYGON (((-106.53650 31.81310, -106.5356...
1  fid-1681139f_172e865dd6a_-7ffd  4500  MULTIPOLYGON (((-106.53650 31.81310, -106.5356...
2  fid-1681139f_172e865dd6a_-7ffe  3600  MULTIPOLYGON (((-106.52230 31.81850, -106.5217...
3  fid-1681139f_172e865dd6a_-7fff  2700  MULTIPOLYGON (((-106.44180 31.79880, -106.4426...
4  fid-1681139f_172e865dd6a_-8000  1800  MULTIPOLYGON (((-106.46880 31.78890, -106.4695...
```

<!-- 
If you load spatial data to a geopandas dataframe you get something like this. You'll have whatever attributes from the data, with the shape represented as a geometry attribute like we've got all the way to the right here.
-->

---

# Shape Shifting

- One operation that _does_ work on malformed polygons: **buffering**
- Buffer creates an area _x_ distance away from a shape
- It also fixes our geometry errors

```python
import geopandas as gpd

isos = gpd.read_file('pm_isochrones.geojson')
isos['geometry'] = isos.buffer(0)
isos.to_file('fixed_pm_isochrones.geojson', driver='GeoJSON')
```
<!-- 
One operation that does work is buffering. What buffers normally do is create an area x distance away from a shape as-the-crow-flies. So, for example, everything in a 1-km radius of a point, or the area within 100 miles of a border. Here, we'll just buffer by 0 to fix the geometry errors.
-->

---

![bg](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/spatial_overlay.png)

<!--- 
It's kind of hard to show the lack of an error message, but now that the polygons are all sorted out it's possible to perform spatial operations on them. Because the OTP isochrones give you more control over constraints than google directions, one question that pops up is which students or stops fall within reasonable commuting range.

This is something that can be answered using a spatial join. Just like an regular join merges datasets based on the relationships between attribute values, a spatial join merges datasets based on spatial relationships between records.

So here are some randomly generated points over the data...

--->

---

![bg](https://raw.githubusercontent.com/amfz/toronto-data-workshop/master/img/spatial_join.png)

<!-- 
...and here's what the result of a spatial join looks like. Points are color-coded by the isochrone they fall in. So you don't have the particulars of individual journeys, but you do get an understanding of commutes and mobility for no API credits.
-->

--- 

# Spatial Joining in Geopandas

```python
import geopandas as gpd

isos = gpd.read_file('fixed_pm_isochrones.geojson')
pts = gpd.read_file('demo_pts.geojson')

pts_in_isos = gpd.sjoin(pts, isos, how='left', op='within')
pts_in_isos.to_file('pm_joined.geojson', driver='GeoJSON')

```

<!-- 
And just to round things out, this is easily doable within geopandas.
-->

---

# Thank you!

---
# Useful links and references

#### Google Documentation
- Directions API: https://developers.google.com/maps/documentation/directions/start
- Polyline encoding: https://developers.google.com/maps/documentation/utilities/polylinealgorithm

#### OpenTripPlanner
- OTP documentation: http://docs.opentripplanner.org/en/latest/
- Isochrone API reference: http://dev.opentripplanner.org/apidoc/1.4.0/resource_LIsochrone.html
- Marcus Young's OTP tutorial in R: https://github.com/marcusyoung/otp-tutorial

---
# Useful links and references 
#### Python Libraries
- requests: https://requests.readthedocs.io/en/master/
- geopandas: https://geopandas.org/index.html
- dateutil: https://dateutil.readthedocs.io/en/stable/ 
- polyline: https://pypi.org/project/polyline/

#### Spatial Data
- Simple Features: https://www.ogc.org/standards/sfa
- geoJSON: https://geojson.org/
- Spatial reference systems: https://epsg.io/

---
# Useful links and references

#### GIS Software
- QGIS: https://www.qgis.org/en/site/

#### Misc
- This presentation was made with Marp: https://marp.app/