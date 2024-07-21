# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API.

The code snippets can be copied into GEE Code Editor to run.
The image data used for the project is available upon request through nii.azu.crabbe@gmail.com. 
Unless you have the image data, the part of the code that requires loading the datasets might not work for you.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

## **Mapping bitoubush**
In this section, the data sets required for the project are loaded up into the Code Editor
The data sets include:
- SkySat satellite imagery for the main analysis
- UAV imagery over the study area to retrieve the study area (*aka* region of interest-ROI)

|Site | Lat | Lon |
|---|---|---|
| Birdies Beach| -33.23352°|151.57426°|

Add an image
![description] (path)

**Loading the data sets into the Code Editor**
Caution, the paths for the data sets must be changed to have the code working.

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
To do this, use the **Map.addLayer()** function

```JavaScript
Map.addLayer(munmorah, {bands:["b4","b3","b2"], min:0, max:2000}, 'SkySat RGB Munmuorah');
````
You might have observed that in the above code that the bands have been specified in the order of RGB with the R = "b3", G ="b4", and B ="b2". Note, R= red, G= green, and B= blue band. The computer can display three bands at a time.
Additionally, the range of brightness values have been specified as min:0, max:2000. The output layer is labelled as 'SkySat RGB Munmuorah'. This is the name you would see in the layer manager upon running the code.

### Define a polygon for the region of interest

In this case, the outline of the UAV image, which covers the study area, was retrieved. 

```JavaScript
var mm_uav_ms_oneBand = mm_uav_ms.select('b1').toInt(); // selects any of the bands to use, b1 used here
var roi= mm_uav_ms_oneBand.reduceToVectors({
  bestEffort: true
  
});
```

Create a symoblogy that makes the polygon geometry transparent and display outline of the study image 

```JavaScript
var symbology = {color: 'black', fillColor: '00000000'};
Map.addLayer(roi.style(symbology), {}, 'Munmorah Geometry');
``` 
Load the ground reference label.  This is a field observation data on botanical composition, showing the proportion of bitoubush in the mix

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

The SkySat satellite used in this project has four useful spectral band: R, G, B and NIR. Additional spectral information is required to improve 
the detectability of the target. Thus, spectral indices sensitive to plants were derived to provide additional spectral information.
A function called `vegetation_indices` was created to produce a list of vegetation indices from the SkySat Imagery

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

* Textural measures using Grey-level Co-occurence Matrix (GLCM)\
  GLCM is used to capture the spatial relationships between the objects in the image. The GLCM analysis requires a single-band image as an input layer.
  The NDVI was used as it is a single-band layer and widely used in several areas, including invasive plant detection. 

  * Extract the NDVI layer
    ```JavaScript
    var engineer_model_features = vegetation_indices(munmorah);
    
    //print the layer to the Console
    print (engineer_model_features_NDVI, 'engineer_model_features_NDVI');
    
    //add the layer to the base map
    Map.addLayer(engineer_model_features_NDVI, {}, "engineer_model_features_NDVI");
    ```

  * scale the NDVI layer to 8-bits- this is a requirement for Earth Engine to work
    ```JavaScript
    var engineer_model_features_NDVI_8bits = engineer_model_features_NDVI.unitScale(-1, 1).multiply(255).toByte();
    ```
  * define a neighborhood with a square kernel, here, 3x3 kernel size is used to preserve spatial details

    ```JavaScript
    var square_munmorah = ee.Kernel.square({radius: 3}); 
    ```
  * compute the GLCM and print the result to the Console

    ```JavaScript
    var glcm_munmorah = engineer_model_features_NDVI_8bits.glcmTexture({size: 3});
    print(glcm_munmorah, 'glcm_munmorah');
    ```
* Several GLCM textural bands were produced but based on literature only the bands relevant to the project were selected 
   * select the required GLCM bands and print result to the Console
     ```JavaScript
     var glcm_munmorah_selectedBands = glcm_munmorah.select(['NDVI_asm', 'NDVI_contrast', 'NDVI_corr', 'NDVI_var', 'NDVI_idm', 'NDVI_savg', 'NDVI_ent']);
     print(glcm_munmorah_selectedBands,'glcm_munmorah_selectedBands');
     ```
* Principal components analysis of the selected GLCM features as they are usually highly corrrelated 
   * create an image collection for the selected bands
     ```JavaScript
     var glcm_munmorah_selectedBands =ee.ImageCollection(glcm_munmorah_selectedBands); // this stacks the selected GLCM bands together to produce an image collection
     print(glcm_munmorah_selectedBands, 'glcm_munmorah_selectedBands');
     ```
    
```JavaScript
// prepare the imagery for the pca    
var Preped = glcm_munmorah_selectedBands.map(function(image){
  var orig = image;
  var region = image.geometry();
  var scale = 0.5;
  var bandNames = ['NDVI_asm', 'NDVI_contrast', 'NDVI_corr', 'NDVI_var', 'NDVI_idm', 'NDVI_savg', 'NDVI_ent'];
  var meanDict = image.reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: region,
    scale: scale,
    maxPixels: 1e12
  });
  var means = ee.Image.constant(meanDict.values(bandNames));
  var centered = image.subtract(means);
  var getNewBandNames = function(prefix) {
  var seq = ee.List.sequence(1, 7);
  return seq.map(function(b) {
    return ee.String(prefix).cat(ee.Number(b).int());
    });
  };
  
  // create a PCA function
  var getPrincipalComponents = function(centered, scale, region) {
    var arrays = centered.toArray();
    var covar = arrays.reduceRegion({
      reducer: ee.Reducer.centeredCovariance(),
      geometry: region,
      scale: scale,
      maxPixels: 1e12
    });
    var covarArray = ee.Array(covar.get('array'));
    var eigens = covarArray.eigen();
    var eigenValues = eigens.slice(1, 0, 1);
    var eigenVectors = eigens.slice(1, 1);
    var arrayImage = arrays.toArray(1);
    var principalComponents = ee.Image(eigenVectors).matrixMultiply(arrayImage);
    var sdImage = ee.Image(eigenValues.sqrt())
    .arrayProject([0]).arrayFlatten([getNewBandNames('sd')]);
    return principalComponents.arrayProject([0])
    .arrayFlatten([getNewBandNames('pc')])
    .divide(sdImage);
    };

// apply the pca function
  var pcImage = getPrincipalComponents(centered, scale, region);
  return ee.Image(image.addBands(pcImage));
});

// print result to the Console
print("PCA imagery: ",Preped);

// select the first principal component as this explains considerable variations in the data
var glcmBands_plus_pcaBands = Preped.toBands();

// print the result
print('glcmBands_plus_pcaBands', glcmBands_plus_pcaBands);
```

* Image Segmentation using SNIC \
Image segmentation would groups pixels of similar spectral characteris to create geographic objects. The multispectral SkySat image was segmented using the simple non-iterative clustering (SNIC) to produce geographic objects to use as input variables. The SNIC parameters used were based on literature.

```JavaScript
 // define the seed image 
var snic_seeds = ee.Algorithms.Image.Segmentation.seedGrid({
  size:20,
  gridType:"square"
  });

// apply the SNIC algorithm to segment the skysat imagery
var snic = ee.Algorithms.Image.Segmentation.SNIC({
  image: munmorah,
  compactness: 0,
  connectivity: 8,
  neighborhoodSize: 256,
  //size: 3,
  seeds: snic_seeds
}).reproject({
  crs: 'EPSG:32756',
  scale: 0.5
});

// select the clusters, the result of the SNIC segmentation
var snicClusters = snic.select('clusters'); 

```

### Predictor Variables
Combine all the features, including spectral bands, vegetation indices, GLCM, and SNIC clusters, to produce a list of predictor variables.

```JavaScript
// add the glcm features to the spectral features
var engineer_model_features=engineer_model_features.addBands(glcmBands_plus_pcaBands.select('0_pc1'));
//print(engineer_model_features, 'engineer_model_features2'); 

// add the segmentation layer, too
var engineer_model_features = engineer_model_features.addBands(snicClusters)
print(engineer_model_features, 'image2classifyNotNormalised');
```

* Normalise the predictor variables

```JavaScript
  // this is a function to normalise the input features
function normalize(image){
  var bandNames = image.bandNames();
  // Compute min and max of the image
  var minDict = image.reduceRegion({
    reducer: ee.Reducer.min(),
    geometry: roi,
    scale: 0.5, 
    maxPixels: 1e9,
    bestEffort: true,
    tileScale: 16
  });
  var maxDict = image.reduceRegion({
    reducer: ee.Reducer.max(),
    geometry: roi,
    scale: 0.5,
    maxPixels: 1e9,
    bestEffort: true,
    tileScale: 16
  });
  var mins = ee.Image.constant(minDict.values(bandNames));
  var maxs = ee.Image.constant(maxDict.values(bandNames));

  var normalized = image.subtract(mins).divide(maxs.subtract(mins));
  return normalized;
}

// apply the 'normalize' function to normalise the input features
var composite = normalize(engineer_model_features);
```  
Rename the variables
```JavaScript

// create a variable that holds the relevant model features; here a more meaning band name is specified.
var bands= ['Blue', 'Green', 'Red', 'NIR', 'NDVI', 'NDYI', 'RBNI', 'MYI', 'HRFI', 'GLCM','Clusters']

// rename bands
var composite =composite.rename(bands)
```` 
### Creating a Sample of Training Areas in the Image
```JavaScript

/*load the ground reference data; this data was collected by weed experts through visual assessment of plants, 
recorded the botanical composition (in %) and GPS locations of the species.
based on the species composition, a sampling plot (1m x 1m) was labeled as bitoubush or no-bitoubush.
*/

var landcover= ee.FeatureCollection('projects/ee-richcrabbe/assets/BB-BIRDIES-REFgroundCHECKED');
print(landcover, 'landcover');

// remap the 'label'property to align with default JavaScript format
var landcover = landcover.remap([1, 2], [1, 0], 'label'); // 0 = no-bitoubush and 1 = bitoubush
print(landcover, 'landcover'); // note, there were more cases for no-bitoubush 

// sample training areas so spectral information can be assigned to each pixel
var landcover2 = composite.sampleRegions({
  collection: landcover,
  properties: ['label'],
  geometries:true,
  scale: 0.5
});
```
### Make the Class Size Equal 
```JavaScript
// filter the training areas by class to examine the number of cases or pixels for each class
var landcoverClass1 = landcover2.filter(ee.Filter.eq('label', 0))
var landcoverClass2 = landcover2.filter(ee.Filter.eq('label', 1))
print(landcoverClass1.size(), 'landcover1'); // shows the number of cases in for class 1
print(landcoverClass2.size(), 'landcover2'); // shows the number of cases in for class 2

// the number of cases were uneven, so match the size of class 2 to class 1
var landcoverClass1 = landcoverClass1.limit(364)

// merge the feature collection
var landcover2 = landcoverClass2.merge(landcoverClass1)
```

### Partition data to training and test samples  

```JavaScript
// partition training areas into training and test sets: apply 80-20 rule
var landcover2 = landcover2.randomColumn()
var trainingSample = landcover2.filter('random <= 0.8')// 80% of the data would be for model training
var testSample = landcover2.filter('random > 0.8')  // 20% of data for model testing
``` 
### Hyperparameter tuning
The optimal values for the number of trees to grow and out of bag fraction were obtained through a grid search.\
Only these two hyperparameters were tuned; not only because they have a major influence on the model performance but this was done to save computation time.

```JavaScript
// a list of number of trees to explore, the number is in incremental by 10
var numTreesList = ee.List.sequence(10, 150, 10);

//a list of out of bag fractions to use, this is in incremental by 10%
var bagFractionList = ee.List.sequence(0.1, 0.9, 0.1);

// start tuning for the hyperparameters 
var accuracies = numTreesList.map(function(numTrees) {
  return bagFractionList.map(function(bagFraction) {
    // define an RF using these parameters
     var classifier = ee.Classifier.smileRandomForest({
       numberOfTrees: numTrees,
       bagFraction: bagFraction
     })
     // train the RF
      .train({
        features: trainingSample,
        classProperty: 'label',
        inputProperties: composite.bandNames()
      });

    // compute error matrix over a classifcation
    var accuracy = testSample
      .classify(classifier)
      .errorMatrix('label', 'classification')
      .accuracy();
    return ee.Feature(null, {'accuracy': accuracy,
      'numberOfTrees': numTrees,
      'bagFraction': bagFraction})
  })
}).flatten()
```

Explore the results of the grid search (i.e., hyperparameter tuning) 

```JavaScript
//print the accuracies
print(accuracies, ' ACCURACIES');

// get the result as a feature collection
var resultFc = ee.FeatureCollection(accuracies)
print(resultFc, 'resultFc')

// select the parameters that result in the highest accuracy
var resultFcSorted = resultFc.sort('accuracy', false);
var highestAccuracyFeature = resultFcSorted.first();
var highestAccuracy = highestAccuracyFeature.getNumber('accuracy');
var optimalNumTrees = highestAccuracyFeature.getNumber('numberOfTrees');
var optimalBagFraction = highestAccuracyFeature.getNumber('bagFraction');

// print the results after the grid search to the Console
print(highestAccuracy,'highestAccuracy')
print(optimalNumTrees,'optimalNumTrees')
print(optimalBagFraction,'optimalBagFraction')

```
### Create a Random Forest Classification Model
```JavaScript
// use the optimal parameters in a model and perform final classification
var optimalModel = ee.Classifier.smileRandomForest({
  numberOfTrees: optimalNumTrees,
  bagFraction: optimalBagFraction
}).train({
  features: trainingSample,  
  classProperty: 'label',
  inputProperties: composite.bandNames()
});

```
### Classify the imagery using the RF Model
```JavaScript
var finalClassification = composite.classify(optimalModel);
```
* Explore the classification image

  ```JavaScript
  // display the classification image, clip this to the ROI
  
  Map.setCenter(151.60,-33.21,14);
  
  Map.addLayer(finalClassification.clip(roi), {min: 0, max: 1, palette: ['green', 'red']}, 'Random Forest Classification of Bitou');

  // display the reference features with each class coloured differently-function to conditionally set properties of feature collection
  var setFeatureProperties = function(feature){var label = feature.get("class_1")
  var mapColours2Use = ee.List(['yellow', 'black']); 
  
  // use the class as index to lookup the corresponding display color
  return feature.set({style: {color: mapColours2Use.get(label),fillColor: '00000000'}})}

  //apply the function and view the result on base map
  var styled = landcover.map(setFeatureProperties)
  Map.addLayer(styled.style({styleProperty: "style"}), {}, 'Reference Areas')
  
  ```
### Evaluate the RF Model
The perfromance of the RF model was assessed based on the test sample in that error matrix was computed.\
The evaluation metrics reported include accuracy, precision, recall, and F-score.

```JavaScript
var accuracy2 = testSample
      .classify(optimalModel)
      .errorMatrix('label', 'classification');
print(accuracy2, ' accuracy2') 

// assess the performance of the model 
print('Validation Consumer accuracy: ', accuracy2.consumersAccuracy())
print('Validation Producer accuracy: ', accuracy2.producersAccuracy())
print('Validation Overall accuracy: ', accuracy2.accuracy())
print('Validation Kappa: ', accuracy2.kappa())
print('Validation fscore: ', accuracy2.fscore(1))

```

### Bitoubush at Birdies Beach


### Bitoubush at Scotts Head



## **Mapping African Lovegrass (ALG)**



### ALG at Cooma, NSW


### ALG at Kuma Nature Reserve and TSR, NSW
