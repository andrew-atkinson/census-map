**D3data-driven documents**


#visual encodings


*chart types*

a. abela - extremepresentation.com/uploads/documents/choosing_a_good_chart.pdf

basically choose between:
  -comparison, 
  -relationship, 
  -distribution, 
  -composition. 
  
*visualation for Data Scientist*

simple solutions, 
choose the right chart

visual encodings + datatypes + relationship between them = chart types!

choose datatype and the dimensions of the chart. (am I showing the distribution of data, or category, changing over time...etc)

small multiple plots - they generalise well into comparison. 
box plots - used for EDA - exploratory data analysis, used to learn about from the data. As opposed to visualisation which is about communication of findings. 




*mike bostock chloropleth tutorial*

1. pick a metric(pop. density), a geographic region(census tract), a source(cense data). 
2. download shapefile data from the census bureau. https://www.census.gov/geo/maps-data/data/tiger-line.html
3. use http://mapshaper.org/ to make sure the state is by census tract. 
4. `npm install -g shapefile` installs a shapefile-to-GEOJson converter. 
5. `shp2json cb_2014_36_tract_500k.shp -o ny.json` creates the GEOJson file. Also uses the `.dbf` associated file for defining feature properties. 
6. `npm install -g d3-geo-projection` - installs the projection. pre-projects for performance. 
7. `geoproject 'd3.geoTransverseMercator().rotate([76 + 35 / 60, -40]).fitSize([960,960], d)' < ny.json > ny-stateplane.json` performs the projection. outputs as ny-stateplane.json. 
8. `geo2svg -w 960 -h 960 < ny-stateplane.json > ny-stateplane.svg` previews the plane in an SVG. 
9. `npm install -g ndjson-cli` - converts GeoJSON in newline-delimited JSON. 
10. `ndjson-split 'd.features' < ny-stateplane.json > ny-stateplane.ndjson` - breaks the JSONs into JSONS with newlines... that's all. 
11. `ndjson-map 'd.id = d.properties.GEOID.slice(2), d' < ny-stateplane.ndjson > ny-stateplane-id.ndjson` takes the GEOID property, slices the first two digits off. The first two digits are the StateFP (state id), the rest, I assume, is the census tracts. 
12. request a key from 'http://api.census.gov/data/key_signup.html' for accessing the API
13. `curl 'http://api.census.gov/data/2014/acs5?get=B01003_001E&for=tract:*&in=state:36' -o cb_2014_36_tract_B01003.json` this requests the population (B01003) 'for' the tracts 'in' state 36 (NY). -o is the output file. 
14. file reformatting. ndjson concats (remove newlines), then splits (into multiple lines), then maps (by adding the second and third lines -id-, and putting the population at the end under B01003).
```
ndjson-cat cb_2014_36_tract_B01003.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[2] + d[3], B01003: +d[0]}' \
  > cb_2014_36_tract_B01003.ndjson
```
15. joins together two files - one with state population info, one with geometry for the map
```
ndjson-join 'd.id' ny-stateplane-id.ndjson cb_2014_36_tract_B01003.ndjson > ny-stateplane-join.ndjson
```
16. maps the files. density is calc'd as population (d[1])/(area (d[0].etc)*constant-changing-metres-to-miles (2589975...)).the last d[0] takes the existing d[0] values and tacks them on.
```
ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < ny-stateplane-join.ndjson \
  > ny-stateplane-density.ndjson
```
17. Turns a NDjson into regular JSON. End of the process, so the newline-delimited. Data is now ready to tested on 'http://mapshaper.org/'... Mapshaper takes JSON, not NDJSON. 
```
ndjson-reduce 'p.features.push(d), p' '{type: "FeatureCollection", features: []}' \
  < ny-stateplane-density.ndjson \
  > ny-stateplane-density.json
```
18. install D3 ```npm i -g d3```
19. uses map to 'fill' each json object, on a color scale called Viridis
```ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)' < ny-stateplane-density.ndjson > ny-stateplane-color.ndjson
```
20. convert ndjson -> svg (inputting an ndjson means each feature is rendered as a separate path element, with .json, there's just one)
```
geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ny-stateplane-color.ndjson \
  > ny-stateplane-color.svg
```
21. BUT... This generates a (too) large datafile. we need to
  a. *simplify. remove coordinates
  b. *quantize. reduce numerical specificity
  c. *compress. remove redundant geomtry.
17. `npm i -g topojson` topojson reduces the filesize by 80% typically. 
18. `geo2topo -n tracts=ny-stateplane-density.ndjson > ny-tracts-topo.json` this converts to TopoJSON. 
19. `toposimplify -p 1 -f < ny-tracts-topo.json > ny-simple-topo.json` -p 1 specifies a planar method of 1 pixel. But only use because of the previously applied equal-area project (step 6 or 7). otherwise, use '-s' and specify a minimum-area threshold in steradians. steradians are an SI unit for solid angle. 1 steradian is 1/4PI sr. -f removes small things. step 19 performs 21.a.
20. `topoquantize 1e5 < ny-simple-topo.json > ny-quantized-topo.json` performs 21b. 
21. `topomerge -k 'd.id.slice(0, 3)' counties=tracts < ny-quantized-topo.json  > ny-merge-topo.json` overlays county boundaries. -k is a key expression, that will evaulate to group features from tracts objects before merging. (similar to nest.key). 
22. `topomerge --mesh -f 'a !== b' counties=counties < ny-merge-topo.json > ny-topo.json` -f filters if 'a' isn't equal 'b'. by convention 'a' and 'b' are the same exterior arcs. 
23. Use topo2geo to extract the simplified tracts from the topology, pipe to ndjson-map to assign the fill property for each tract, pipe to ndjson-split to break the collection into features, and lastly pipe to geo2svg.
```
topo2geo tracts=- < ny-topo.json \
  | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolatePlasma).domain([0, 12000]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ny-tracts-color.svg
```
24. The above creates a linear relationship between the color and population. This creates a very split place graphic.
25. applies a sqrt to the to the fill... (of a new domain of 0-100)
```topo2geo tracts=- \
  < ny-topo.json \
  | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0, 100]), d.features.forEach(f => f.properties.fill = z(Math.sqrt(f.properties.density))), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 1260 -h 1260 \
  > ny-tracts-sqrt.svg
```



**links**

http://infosthetics.com/
http://www.storytellingwithdata.com/
http://www.vizwiz.com/2013/01/alberto-cairo-three-steps-to-become.html
https://www.targetprocess.com/articles/visual-encoding/
http://flowingdata.com/2010/03/20/graphical-perception-learn-the-fundamentals-first/
http://rawgraphs.io/
http://www.gapminder.org/
http://code.shutterstock.com/rickshaw/
http://nvd3.org/
https://bost.ocks.org/mike/bar/
http://www.perceptualedge.com/articles/misc/Graph_Selection_Matrix.pdf
