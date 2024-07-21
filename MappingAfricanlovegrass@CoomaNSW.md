# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API.

The code snippets can be copied into GEE Code Editor to run.
The image data used for the project is available upon request through nii.azu.crabbe@gmail.com. 
Unless you have the image data, the part of the code that requires loading the datasets might not work for you.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

## **Mapping African lovegrass at Cooma: Bunyan Gliding Club and McDonald Winery**
In this section, the data sets required for the project are loaded up into the Code Editor
The data sets include:
- SkySat satellite imagery for the main analysis
- UAV imagery over the study area to retrieve the study area (*aka* region of interest-ROI)

```JavaScript
//load the SkySar imagery
var coomaSkysat= ee.Image('projects/ee-richcrabbe/assets/ALG_GLIDINGCLUB_MCDONALDS_20240417_232520_skysat_SR')

Map.addLayer(coomaSkysat, {bands:['b3','b2', 'b1'], min:600, max:2000}, 'Cooma Skysat')
// load up the study area polygon
var coomaROI = ee.FeatureCollection('projects/ee-richcrabbe/assets/ALG_Cooma_ROI_19062024')
//print(coomaROI, 'Cooma ROI')

//export the ROI to Google Drive
Export.table.toDrive({
  collection: coomaROI,
  description:'Cooma_GlidingClub_McDonalds_ROI',
  //crs: 'EPSG:32753',
  folder: 'CSU',
  fileFormat: 'SHP'
});


// trim the image
var coomaSkysat=coomaSkysat.clip(coomaROI)
//Map.addLayer(coomaSkysat, {bands:['b3','b2', 'b1'], min:600, max:2000}, 'Cooma Skysat 2')

//load up the reference data
var glidingClubRefData =ee.FeatureCollection('projects/ee-richcrabbe/assets/ALG_glidingClub_referenceData')
//print( glidingClubRefData, 'GlidingClubRefData')
var macdonaldsRefData =ee.FeatureCollection('projects/ee-richcrabbe/assets/ALG_macDonalds_referenceData')
//print(macdonaldsRefData, 'macdonaldsRefData')

//merge the reference data
var referenceData = glidingClubRefData.merge(macdonaldsRefData)
//print(referenceData, 'referenceData')

// select the individual bands of the skysat imagery  
var blue = coomaSkysat.select('b1'); // selects the blue band only
var green = coomaSkysat.select('b2'); // selects the green band
var red = coomaSkysat.select('b3');  // selects the red band
var nir = coomaSkysat.select('b4'); //selects the near infrared band

// a function that computes vegetation indices  
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

// feature engineering
var engineer_model_features = vegetation_indices(coomaSkysat);
//print (engineer_model_features, 'engineer_model_features');

// extract the NDVI band
var engineer_model_features_NDVI = engineer_model_features.select('NDVI');
//print (engineer_model_features_NDVI, 'engineer_model_features_NDVI');
///Map.addLayer(engineer_model_features_NDVI, {}, "engineer_model_features_NDVI");



// Define a neighborhood with a kernel
// scale the NDVI to 8-bits as this is required for glcm to work
var engineer_model_features_NDVI_8bits = engineer_model_features_NDVI.unitScale(-1, 1).multiply(255).toByte();
var square_munmorah = ee.Kernel.square({radius: 3});


// Compute the gray-level co-occurrence matrix (GLCM), get contrast.

var glcm_munmorah = engineer_model_features_NDVI_8bits.glcmTexture({size: 3});
//print(glcm_munmorah, 'glcm_munmorah');

//select the relevant glcm bands and print this to the Console
var glcm_munmorah_selectedBands = glcm_munmorah.select(['NDVI_asm', 'NDVI_contrast', 'NDVI_corr', 'NDVI_var', 'NDVI_idm', 'NDVI_savg', 'NDVI_ent']);
//print(glcm_munmorah_selectedBands,'glcm_munmorah_selectedBands');

// make the glcm data an image collection to be used for the pca
var glcm_munmorah_selectedBands =ee.ImageCollection(glcm_munmorah_selectedBands);
//print (glcm_munmorah_selectedBands, 'glcm_munmorah_selectedBands ');

//**********************************************

// Principal components analysis- source:https://stackoverflow.com/questions/62436252/performing-pca-per-image-over-imagecollection-in-google-earth-engine
// Display AOI
//var point = ee.Geometry.Point([151.602,-33.209]);
//Map.centerObject(point,10);

// Preparing imagery for PCA
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

  // PCA function
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
//print("PCA imagery: ",Preped);

// select pca 1
var glcmBands_plus_pcaBands = Preped.toBands();
//print('glcmBands_plus_pcaBands', glcmBands_plus_pcaBands);

// image to classify
var engineer_model_features=engineer_model_features.addBands(glcmBands_plus_pcaBands.select('0_pc1'));
//print(engineer_model_features, 'engineer_model_features2'); 

// segmement the image using SNIC
var snic_seeds = ee.Algorithms.Image.Segmentation.seedGrid({
  size:20,
  gridType:"square"
  });

var snic = ee.Algorithms.Image.Segmentation.SNIC({
  image: coomaSkysat,
  compactness: 0,
  connectivity: 8,
  neighborhoodSize: 256,
  //size: 3,
  seeds: snic_seeds
}).reproject({
  crs: 'EPSG:32756',
  scale: 0.5
});


var snicClusters = snic.select('clusters'); 
//Map.addLayer(snicClusters .randomVisualizer(), null, "snicClusters ");

// add the segmentation layer
var engineer_model_features = engineer_model_features.addBands(snicClusters)
print(engineer_model_features, 'image2classifyNotNormalised');

// normalise the image to use for the classification. below is a function to do so:
function normalize(image){
  var bandNames = image.bandNames();
  // Compute min and max of the image
  var minDict = image.reduceRegion({
    reducer: ee.Reducer.min(),
    geometry: coomaROI,
    scale: 0.5,
    maxPixels: 1e9,
    bestEffort: true,
    tileScale: 16
  });
  var maxDict = image.reduceRegion({
    reducer: ee.Reducer.max(),
    geometry: coomaROI,
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

// image to use for the classification. it is now normalized
var composite = normalize(engineer_model_features);
print(composite, 'Image2Classify')

// create a variable that holds the relevant model features
var bands= ['Blue', 'Green', 'Red', 'NIR', 'NDVI', 'NDYI', 'RBNI', 'MYI', 'HRFI', 'GLCM', 'Cluster']


//rename bands
var composite =composite.rename(bands)
print(composite, 'imageComposite')

// create a variable that holds the relevant model features
//var bands = ['b1', 'b2', 'b3', 'b4', 'NDVI', 'NDYI', 'RBNI', 'MYI', 'HRFI', '0_pc1']; 


// sample for training areas
var landcover2 = composite.sampleRegions({
  collection: referenceData,
  properties: ['class_1'],
  scale: 0.5
  //tileScale: 16
});



//remap the 'class_1' property to align with default JavaScript format- meaning start labeling from zero
var landcover3 = landcover2.remap([1,2,3], [0,1,1], 'class_1'); //ALG, native veg, and non-veg, respectively
print(landcover3, 'landcover3');

//filter collection by class
var landcoverClass0 = landcover3.filter(ee.Filter.eq('class_1', 0))

//filter class 1 and limit class size to match class 0
var landcoverClass1 = landcover3.filter(ee.Filter.eq('class_1', 1)).limit(345)
print(landcoverClass1, 'landcoverClass1')

//merge landcoverClass0 and landcoverClass1
var landcover4 = landcoverClass0.merge(landcoverClass1)
print(landcover4, 'landcover4')

//split data into training and test sets
var landcover5 = landcover4.randomColumn()
var trainingSample = landcover5.filter('random <= 0.8')
var testSample = landcover5.filter('random > 0.8')
print(trainingSample, 'trainingSample')

// hyper-parameter tuning. we tuned 'numTrees' and 'bagFraction'


// a list of number of trees to explore. the number is in incremental by 10
var numTreesList = ee.List.sequence(10, 150, 10);

//a list of out of bag fractions to use, this is in incremental by 10%
var bagFractionList = ee.List.sequence(0.1, 0.9, 0.1);


// tune the 2 parameters 
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
        classProperty: 'class_1',
        inputProperties: composite.bandNames()
      });

    // Here we are classifying a table instead of an image
    // Classifiers work on both images and tables
    var accuracy = testSample
      .classify(classifier)
      .errorMatrix('class_1', 'classification')
      .accuracy();
    return ee.Feature(null, {'accuracy': accuracy,
      'numberOfTrees': numTrees,
      'bagFraction': bagFraction})
  })
}).flatten()

//print(accuracies, ' ACCURACIES');

// get the result as a feature collection
var resultFc = ee.FeatureCollection(accuracies)
//print(resultFc, 'resultFc')

// select the parameters that result in the highest accuracy
var resultFcSorted = resultFc.sort('accuracy', false);
var highestAccuracyFeature = resultFcSorted.first();
var highestAccuracy = highestAccuracyFeature.getNumber('accuracy');
var optimalNumTrees = highestAccuracyFeature.getNumber('numberOfTrees');
var optimalBagFraction = highestAccuracyFeature.getNumber('bagFraction');

print(highestAccuracy,'highestAccuracy')
print(optimalNumTrees,'optimalNumTrees')
print(optimalBagFraction,'optimalBagFraction')

// Export the result as CSV
Export.table.toDrive({
  collection: resultFc,
  description: 'Multiple_Parameter_Tuning_Results',
  folder: 'CSU',
  fileNamePrefix: 'numtrees_bagfraction',
  fileFormat: 'CSV'});


// Use the optimal parameters in a model and perform final classification
var optimalModel = ee.Classifier.smileRandomForest({
  numberOfTrees: optimalNumTrees,
  bagFraction: optimalBagFraction
}).train({
  features: trainingSample,  
  classProperty: 'class_1',
  inputProperties: composite.bandNames()
});

// classify the image
var finalClassification = composite.classify(optimalModel);

/*
// export the classification image to GEE assests
Export.image.toAsset({
  image: finalClassification,
  //description: 'image_export_crstransform',
  assetId: 'projects/ee-richcrabbe/assets/bitouRFclassification'  //
  //region: region,
  //crsTransform: [30, 0, -2493045, 0, -30, 3310005],
  //crs: 'EPSG:5070'
});
*/

// display the classified image
Map.setCenter(149.137,-36.130, 16) 
Map.addLayer(finalClassification, {min: 0, max:1, palette: ['red', 'green']}, 'Random Forest Classification of ALG'); // just to reiterate that 0 = low infestation 1=high inffestation
//Map.addLayer(landcover2, {}, 'reference data')

/*
//export classification image to google drive
var viz_1 = {min: 0, max: 1, palette: ['red', 'green']};
Export.image.toDrive({
    image: finalClassification.visualize(viz_1),
    description: 'ALG_Cooma_RF2',
    scale: 0.5,
    crs: 'EPSG:32753',
    maxPixels: 1e13
});
*/

// display the reference points with each class coloured differently- 
// function to conditionally set properties of feature collection
var setFeatureProperties = function(feature){ 
  var label = feature.get("class_1") // 0,1, or 2
  var mapColours2Use = ee.List(['yellow', 'black']); // class 0 should be blue, class 1 should be red
  
  // use the class as index to lookup the corresponding display color
  return feature.set({style: {color: mapColours2Use.get(label),fillColor: '00000000'}})
}

// apply the function and view the results on map
var referenceData = referenceData.remap([1,2,3], [0,1,1], 'class_1') // just so polygon features would be displayed
var styled = referenceData.map(setFeatureProperties)
Map.addLayer(styled.style({styleProperty: "style"}), {}, 'Reference Areas')

// assesss performance of the optimal model 
var accuracy2 = testSample
      .classify(optimalModel)
      .errorMatrix('class_1', 'classification')


//Print the user's accuracy to the console
print('Validation Overall accuracy: ', accuracy2.accuracy())
print('Validation Consumer accuracy: ', accuracy2.consumersAccuracy())
print('Validation Producer accuracy: ', accuracy2.producersAccuracy())
print('Validation Kappa: ', accuracy2.kappa())
print('Validation fscore: ', accuracy2.fscore(1))


// chart the accuracy metrics
// make arrays
var overallAccuracy = accuracy2.accuracy()
var precision = ee.Number(accuracy2.producersAccuracy().get([0,0])).multiply(100)
var recall = ee.Number(accuracy2.consumersAccuracy().get([0,0])).multiply(100)
var fscore = ee.Number(accuracy2.fscore().get([0])).multiply(100)

//var array1 = [precision, recall, fscore]
//print(precision, 'precision')
//print(recall, 'recall')
//print(fscore, 'fscore')

//print(array1, 'array1')
var precision2 = ee.Number(accuracy2.producersAccuracy().get([-1,-1])).multiply(100)
var recall2 = ee.Number(accuracy2.consumersAccuracy().get([-1,-1])).multiply(100)
var fscore2 = ee.Number(accuracy2.fscore().get([1])).multiply(100)

//print(precision2, 'precision2')
//print(recall2, 'recall2')
//print(fscore2, 'fscore2')

//create a data table to define a chart in the end
var dataTable = [
 ['Metric', " No-Infestation", "Infestation"],
  ['Precision', precision.getInfo(), precision2.getInfo()],  //problematic. spit error message
  ['Recall', 
  recall.getInfo(), recall2.getInfo()], 
  ['F-score', fscore.getInfo(), fscore2.getInfo()], 
];


// Define the chart and print it to the console.
var chart = ui.Chart(dataTable).setChartType('ColumnChart').setOptions({
  colors: ['red', 'green'],
  title: 'The performance of random forest classification of ALG, Gliding Club and McDonald Winery',
  //legend: {position: 'none'},
  hAxis: {title: 'Metric', titleTextStyle: {italic: false, bold: true}},
  vAxis: {viewWindow: {min: 50, max: 100},title: 'Accuracy (%)', titleTextStyle: {italic: false, bold: true}}
  
});
print(chart);


///// post-hoc analysis //////////////////////////////////////////

//optional- apply spatial filter to the classified image to remove speckles


//************************************************************************** 
// Feature Importance
//************************************************************************** 

// Run .explain() to see what the classifer looks like
print(optimalModel.explain())

// Calculate variable importance
var importance = ee.Dictionary(optimalModel.explain().get('importance'))

// Calculate relative importance
var sum = importance.values().reduce(ee.Reducer.sum())

var relativeImportance = importance.map(function(key, val) {
   return (ee.Number(val).multiply(100)).divide(sum)
  })
print(relativeImportance)

// Create a FeatureCollection so we can chart it
var importanceFc = ee.FeatureCollection([
  ee.Feature(null, relativeImportance)
])

var chart2 = ui.Chart.feature.byProperty({
  features: importanceFc
}).setOptions({
      title: 'Feature Importance',
      vAxis: {title: 'Importance'},
      hAxis: {title: 'Feature'}
  })
print(chart2)

print('End of Code')
```
