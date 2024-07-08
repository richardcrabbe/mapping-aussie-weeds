# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API

The code snippets can be copied into GEE Code Editor to run.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

## **Mapping bitoubush**
In this section, the data sets required for the project are loaded up into the GEE Code Editor
The data sets include:
- SkySat satellite imagery for the main analysis
- UAV imagery over the study area to retrieve the study area (*aka* region of interest-ROI)

|Site | Lat | Lon |
|---|---|---|
| Birdies Beach| -33.23352°|151.57426°|

Add an image
![description] (path)

**Loading the data sets into EE Code Editor**

The code snippet below loads the SkySat imagery to a variable called `munmorah`
```JavaScript
var munmorah = ee.Image("projects/ee-richcrabbe/assets/mmResize_SR");
````
Print the **munmorah** to the Console to inspect the attributes of the image
```JavaScript
print(munmorah);
```
Another image data required was the drone imagery. This is used to define the boundary layer of the project. UAV multipsectral drone was flown to sample the field prior to SkySat acquisition.

Load the UAV imagery to a variable called **mm_uav_ms**
```JavaScript
var mm_uav_ms = ee.Image("users/richcrabbe/birdiesBB-MS-ROI");
print(mm_uav_ms,'Munmorah UAV MS' );
```

### Visualisation of an imagery
Once you have a good understanding about the image via its attributes, including the number of bands and resolution, the next step is to display the image to view it.
To do this, use the **Map.addLayer()** function within the GEE

```JavaScript
Map.addLayer(munmorah, {bands:["b4","b3","b2"], min:0, max:2000}, 'SkySat RGB Munmuorah');
````
You might have observed that in the above code that the bands have been specified in the order of RGB with the R = "b3", G ="b4", and B ="b2". Note, R= red, G= green, and B= blue band. The computer can display three bands at a time.
Additionally, the range of brightness values have been specified as min:0, max:2000. The output layer is labelled as 'SkySat RGB Munmuorah'. This is the name of would see in the layer manager upon running the code.

### Define a polygon for the region of interest

In this case, the outline of the UAV image, which covers the study area, was retreived. 

```JavaScript
var mm_uav_ms_oneBand = mm_uav_ms.select('b1').toInt(); // selects any of th bands to use, b1 used here
var roi= mm_uav_ms_oneBand.reduceToVectors({
  bestEffort: true
  
});
```

Create a symoblogy that makes the polygon geomerty transparent and display outline of the study image 
```JavaScript
var symbology = {color: 'black', fillColor: '00000000'};
Map.addLayer(roi.style(symbology), {}, 'Munmorah Geometry');
``` 
Load the ground reference label.  This is a field observation data on species composition. It tells the proportion of bitoubush in the mix
```JavaScript
var landcover= ee.FeatureCollection('projects/ee-richcrabbe/assets/BB-BIRDIES-REFgroundCHECKED');
print(landcover, 'landcover');
```
Remap the 'label'property to align with default JavaScript format
```JavaScript
var landcover = landcover.remap([1, 2], [1, 0], 'label'); // zero is no-bitou and one = bitou
print(landcover, 'landcover');
```

### Feature Engineering

We defined a function called `vegetation_indices` that creates a list of vegetation indices from the SkySat Imagery

```JavaScript

var vegetation_indices = function(image) {
  var blue = image.select('b1'); // selects the blue band only
  var green = image.select('b2'); // selects the green band
  var red = image.select('b3');  // selects the red band
  var nir = image.select('b4'); //selects the near infrared band
  var ndyi = green.subtract(blue).divide(green.add(blue)).rename('NDYI'); //use this
  var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
  var rbni = red.subtract(blue).divide(red.add(blue)).rename('RBNI');
  //var hrfi2 = red.subtract(blue).divide(green.add(blue)).rename('HRFI2');
  var hrfi = red.subtract(blue).multiply(green.subtract(blue)).rename('HRFI'); //use this
  var myi = red.multiply(green).divide(blue).rename('MYI'); //use this
  //add the output of a vegetation index as a band to the original bands of the image and return an image with more bands
  return image.addBands(ndvi).addBands(ndyi).addBands(rbni).addBands(myi).addBands(hrfi);
};
```
Apply the function to the SkySat imagery and print the result to the Console

```JavaScript
var engineer_model_features = vegetation_indices(munmorah);

print (engineer_model_features, 'engineer_model_features');

```

* Textural measures using Grey-level Co-occurence Matrix

Extract the NDVI layer

```JavaScript
var engineer_model_features_NDVI = engineer_model_features.select('NDVI');
print (engineer_model_features_NDVI, 'engineer_model_features_NDVI');
Map.addLayer(engineer_model_features_NDVI, {}, "engineer_model_features_NDVI");
```






### Bitoubush at Birdies Beach


### Bitoubush at Scotts Head



## **Mapping African Lovegrass (ALG)**



### ALG at Cooma, NSW


### ALG at Kuma Nature Reserve and TSR, NSW
