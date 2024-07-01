# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API

The code snippets cane be copied into GEE Code Editor to run.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

## **Mapping bitoubush**
In this section, the data sets required for the project are loaded up into the GEE Code Editor
The data sets include the SkySat Imagery and a shapefile for the study area (*aka* region of interest-ROI)
The code snippet below loads the SkySat imagery to a variable called **munmorah**
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

### Bitoubush at Birdies Beach


### Bitoubush at Scotts Head



## **Mapping African Lovegrass (ALG)**



### ALG at Cooma, NSW


### ALG at Kuma Nature Reserve and TSR, NSW
