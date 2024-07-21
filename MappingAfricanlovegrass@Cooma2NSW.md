# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API.

The code snippets can be copied into GEE Code Editor to run.
The image data used for the project is available upon request through nii.azu.crabbe@gmail.com. 
Unless you have the image data, the part of the code that requires loading the datasets might not work for you.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

## **Mapping African lovegrass at Cooma: Kuma Nature Reserve and Travelling Stock Reserve**
In this section, the data sets required for the project are loaded up into the Code Editor
The data sets include:
- SkySat satellite imagery for the main analysis
- UAV imagery over the study area to retrieve the study area (*aka* region of interest-ROI)

```JavaScript
////// TSR SITE ///////////////////////////////////////////////

// load the reference data; the data was from both sites
var tsr = ee.FeatureCollection('projects/ee-richcrabbe/assets/ALG_TSR_ReferenceData_ALLcases')

var tsrNative = tsr.filter(ee.Filter.eq('class_1', 1))
var tsrALG =tsr.filter(ee.Filter.eq('class_1', 2))

var tsrALG = tsrALG.randomColumn()
var tsrCrossOut= tsrALG.filter('random <= 0.91')
var tsrALG_useThis = tsrALG.filter('random > 0.91')

// merge the tsr ALG and no-ALG cases
var tsrCases= tsrALG_useThis.merge(tsrNative)

// load the ROI for TSR and Kuma
var tsrROI = ee.FeatureCollection('projects/ee-richcrabbe/assets/alg_tsrSite_roi')
var kumaROI = ee.FeatureCollection('projects/ee-richcrabbe/assets/alg_kumaReserve_roi') 

// merge the two ROIs
var kumaTSR_ROI = kumaROI.merge(tsrROI)
Map.addLayer(kumaTSR_ROI , {}, 'kumaTSR_ROI')

//load reference data that has class size to be same. this data is different to the one uploaded earlier
var landcover=ee.FeatureCollection ('projects/ee-richcrabbe/assets/ALG_KumaNR_ReferenceData_Balance').merge(tsrCases)
print(landcover, 'landcover')

//remap the 'class_1' property to align with default JavaScript format- meaning start labeling from zero
var landcover = landcover.remap([1, 2], [0, 1], 'class_1'); // zero is no alg and one = alg

// display this ground reference data
var symbology = {color: 'black', fillColor: '00000000'};
//Map.addLayer(landcover.style(symbology), {}, 'Ground Reference Features');

// load the skysat imagery
var img_kuma_tsr = ee.Image ('projects/ee-richcrabbe/assets/ALG_TSR_KUMA_20240417_232520_skysat_SR');

//var img_kuma_tsr = ee.Image('projects/ee-richcrabbe/assets/ALG_GLIDINGCLUB_MCDONALDS_20240417_232520_skysat_SR')

// resize the skysat imagery to ROI
var img_kuma_tsr = img_kuma_tsr.clip(kumaTSR_ROI)

// set map centre to zoom in on ROI 
Map.setCenter(149.16463983008765,-36.26771713238252,16)
Map.addLayer(img_kuma_tsr, {min:0, max:2000, bands:['b4','b3', 'b2']}, 'Skysat Image')


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

// feature engineering for Kuma NR
var engineer_model_features = vegetation_indices(img_kuma_tsr);
//print (engineer_model_features, 'engineer_model_features');

// extract the NDVI band
var engineer_model_features_NDVI = engineer_model_features.select('NDVI');

// scale the NDVI to 8-bits as this is required for glcm to work
var engineer_model_features_NDVI_8bits = engineer_model_features_NDVI.unitScale(-1, 1).multiply(255).toByte();
var square_munmorah = ee.Kernel.square({radius: 3});

// compute the gray-level co-occurrence matrix (GLCM)
var glcm_munmorah = engineer_model_features_NDVI_8bits.glcmTexture({size: 3});

//select the relevant glcm bands and print this to the Console
var glcm_munmorah_selectedBands = glcm_munmorah.select(['NDVI_asm', 'NDVI_contrast', 'NDVI_corr', 'NDVI_var', 'NDVI_idm', 'NDVI_savg', 'NDVI_ent']);
print(glcm_munmorah_selectedBands,'glcm_munmorah_selectedBands');

// make the glcm data an image collection to be used for the pca
var glcm_munmorah_selectedBands =ee.ImageCollection(glcm_munmorah_selectedBands);

// principal components analysis
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

// print("PCA imagery: ",Preped);

// select pca 1
var glcmBands_plus_pcaBands = Preped.toBands();
//print('glcmBands_plus_pcaBands', glcmBands_plus_pcaBands);

// segmement the image using SNIC
var snic_seeds = ee.Algorithms.Image.Segmentation.seedGrid({
  size:20,
  gridType:"square"
  });

var snic = ee.Algorithms.Image.Segmentation.SNIC({
  image: img_kuma_tsr,
  compactness: 0,
  connectivity: 8,
  neighborhoodSize: 256,
  seeds: snic_seeds
}).reproject({
  crs: 'EPSG:32756',
  scale: 0.5
});


var snicClusters = snic.select('clusters'); 

// image to classify but not normalised
var engineer_model_features=engineer_model_features.addBands(glcmBands_plus_pcaBands.select('0_pc1'));
print(engineer_model_features, 'engineer_model_features2'); 

// add the segmentation layer
var engineer_model_features = engineer_model_features.addBands(snicClusters)
print(engineer_model_features, 'image2classifyNotNormalised');

// normalise the image to use for the classification. below is a function to do so
function normalize(image){
  var bandNames = image.bandNames();
  // Compute min and max of the image
  var minDict = image.reduceRegion({
    reducer: ee.Reducer.min(),
    geometry: kumaTSR_ROI,
    scale: 0.5,
    maxPixels: 1e9,
    bestEffort: true,
    tileScale: 16
  });
  var maxDict = image.reduceRegion({
    reducer: ee.Reducer.max(),
    geometry: kumaTSR_ROI,
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

// display the image to classify
Map.addLayer(composite, {bands:['b3', 'b2', 'b1'], min:0, max:0.3}, 'image2classify')

// create a variable that holds the relevant model features
var bands= ['Blue', 'Green', 'Red', 'NIR', 'NDVI', 'NDYI', 'RBNI', 'MYI', 'HRFI', 'GLCM', 'Clusters']

// rename bands
var composite =composite.rename(bands)
print(composite, 'imageComposite')

// sample for training areas
var landcover2 = composite.sampleRegions({
  collection: landcover,
  properties: ['class_1'], 
  scale: 0.5
  //tileScale: 16
});

print(landcover, 'landcover2')

// filter by cover class
var landcoverClass1 = landcover2.filter(ee.Filter.eq('class_1', 0))
var landcoverClass2 = landcover2.filter(ee.Filter.eq('class_1', 1))

// print sample size
print(landcoverClass1.size(), 'landcover1'); 
print(landcoverClass2.size(), 'landcover2');

// limit class size to that of the class 1
var landcoverClass2 = landcoverClass2.limit(78)

// merge the feature collection
var landcover3 = landcoverClass2 .merge(landcoverClass1)
print(landcover3, 'landcover3')

//partition label data into training and test samples
landcover3 = landcover3.randomColumn()
var trainingSample = landcover3.filter('random <= 0.8')
var testSample = landcover3.filter('random > 0.8')
print(testSample, 'testSample')

//************************************************************************** 
// Hyperparameter Tuning
//************************************************************************** 
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
        inputProperties:  composite.bandNames()
      });

    // error matrix assessment 
    var accuracy = testSample
      .classify(classifier)
      .errorMatrix('class_1', 'classification')
      .accuracy();
    return ee.Feature(null, {'accuracy': accuracy,
      'numberOfTrees': numTrees,
      'bagFraction': bagFraction})
  })
}).flatten()

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

print(highestAccuracy,'highestAccuracy')
print(optimalNumTrees,'optimalNumTrees')
print(optimalBagFraction,'optimalBagFraction')

// Use the optimal parameters in a model and perform final classification
var optimalModel = ee.Classifier.smileRandomForest({
  numberOfTrees: optimalNumTrees,
  bagFraction: optimalBagFraction
}).train({
  features: trainingSample,  
  classProperty: 'class_1',
  inputProperties: composite.bandNames()
});

//export RF classifier to use later
Export.classifier.toAsset({
  classifier: optimalModel,
  description: 'RFclassifier_KumaAndTSR',
  assetId: assetId
});

// classify the image
var finalClassification = composite.classify(optimalModel);

// display the classified image
Map.addLayer(finalClassification, {min: 0, max: 1, palette: ['green', 'red']}, 'Random Forest Classification of ALG'); // just to reiterate that 0 = no-infestation 1 = high-infestation

// display the reference features with each class coloured differently- 
// function to conditionally set properties of feature collection
var setFeatureProperties = function(feature){ 
  var label = feature.get("class_1") 
  var mapColours2Use = ee.List(['yellow', 'black']); 
  
  // use the class as index to lookup the corresponding display color
  return feature.set({style: {color: mapColours2Use.get(label),fillColor: '00000000'}})
}

// apply the function and view the results on map
var styled = landcover.map(setFeatureProperties)
Map.setCenter(149.164,-36.267,16)
Map.addLayer(styled.style({styleProperty: "style"}), {}, 'Reference Areas')

// assesss performance of the model 
var accuracy2 = testSample
      .classify(optimalModel)
      .errorMatrix('class_1', 'classification')
print(accuracy2.array(), 'accuracy2')

// print the user's accuracy to the console
print('Validation Overall accuracy: ', accuracy2.accuracy())
print('Validation Consumer accuracy: ', accuracy2.consumersAccuracy())
print('Validation Producer accuracy: ', accuracy2.producersAccuracy())
print('Validation Kappa: ', accuracy2.kappa())
print('Validation fscore: ', accuracy2.fscore(1))

// variable importance; run .explain() to see what the classifer looks like
print(optimalModel.explain())

// calculate variable importance
var importance = ee.Dictionary(optimalModel.explain().get('importance'))

// calculate relative importance
var sum = importance.values().reduce(ee.Reducer.sum())

var relativeImportance = importance.map(function(key, val) {
   return (ee.Number(val).multiply(100)).divide(sum)
  })
print(relativeImportance)

// create a FeatureCollection so we can chart it
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
