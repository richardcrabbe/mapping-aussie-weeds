# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API

The code snippets cane be copied into GEE Code Editor to run.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

1, set map centre to make map visualisation easy

```JavaScript
Map.setCenter(151.60,-33.21,14);
Map.addLayer(mm_rgb, {bands:["b3","b2","b1"], min:0, max:255}, 'RGB Munmorah');
```

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

### Visualisation of the image
Once you have a good understanding about the image via its attributes, including the number of bands and resolution, the next step is to display the image to view it.
To do this, use the **Map.addLayer()** function within the GEE

```JavaScript
Map.addLayer(munmorah, {bands:["b4","b3","b2"], min:0, max:2000}, 'Skysat RGB Munmuorah');
````
You might have observed that in the above code that the bands have been specified in the order of RGB with the R = "b3", G ="b4", and B ="b2". Additionally, the range of brightness values have been specificied as min:0, max:2000. The output layer is labelled as 'Skysat RGB Munmuorah'. This is the name of you would see in the layer manager upon running the code.


### Bitoubush at Birdies Beach


### Bitoubush at Scotts Head



## **Mapping African Lovegrass (ALG)**



### ALG at Cooma, NSW


### ALG at Kuma Nature Reserve and TSR, NSW
