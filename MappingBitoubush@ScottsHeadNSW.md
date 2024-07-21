# mapping-aussie-weeds
Detection of invasive plants in New South Wales, Australia, using SkySat satellite imagery and Google Earth Engine (GEE) JavaScript API.

The code snippets can be copied into GEE Code Editor to run.
The image data used for the project is available upon request through nii.azu.crabbe@gmail.com. 
Unless you have the image data, the part of the code that requires loading the datasets might not work for you.

**Acknowledgements**

Google Earth Engine Developers

Google Earth Engine Team

Stacker Overflow

```JavaScript
//define study area
var roi = ee.Geometry({
  'type': 'Polygon',
  'coordinates':
    [[[152.98775357302443,-30.735761398213207],
      [152.98846167620437,-30.735729121930767],
      [152.98807543810622,-30.732727380409955],
      [152.98746925886886,-30.732722769367957],
      [152.98775357302443,-30.735761398213207]]]
});

// load the uav imagery
var uavScottsHead = ee.Image ('projects/ee-richcrabbe/assets/scottsHead_msi_altum_orthomosaic')
print(uavScottsHead, 'uavScottsHead_oneBand')

Map.setCenter(152.9881334183168,-30.734091023014066,20)
Map.addLayer(uavScottsHead, {bands:['b3', 'b2', 'b1'], min:500, max:10000}, 'UAV Image')


// load the skysat imagery
var scottsHead = ee.Image('projects/ee-richcrabbe/assets/BITOU_SCOTTSHEAD_20240417_232426_skysat_SR')
var scottsHead = scottsHead.clip(roi)

Map.setCenter(152.9881334183168,-30.734091023014066,20)
Map.addLayer(scottsHead,{bands:['b3', 'b2', 'b1'], min:100, max:1000}, 'SkySat' )

/*
CLASSIFICATION BEGINS ****************
*/

// ground reference data
var landcover= ee.FeatureCollection('projects/ee-richcrabbe/assets/Bitou_ScottsHead_ReferenceData');
var landcoverClass1 = landcover.filter(ee.Filter.eq('class_1', 1))
var landcoverClass2 = landcover.filter(ee.Filter.eq('class_1', 2))

print(landcoverClass1.size(), 'landcover1'); // 41 cases
print(landcoverClass2.size(), 'landcover2'); // 18 cases

// make the number of cases for each class equal
// randomly partition data to obtain 18 ALG cases 
var landcoverSample = landcoverClass1.randomColumn()
var leaveOut = landcoverSample.filter('random <= 0.54') // this sample is left out 
var  landcoverClassALG = landcoverSample.filter('random > 0.54')  // sample is used
print(landcoverClassALG.size(), 'landcover1'); // 

// merge the feature collection
var landcover = landcoverClass2 .merge(landcoverClassALG)
print(landcover.size(), 'landcover')

Map.addLayer (landcover, {}, 'Reference Features')

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


//remap the 'label'property to align with default JavaScript format
var landcover = landcover.remap([1, 2], [1, 0], 'class_1'); // zero is no-bitou and one = bitou
print(landcover, 'landcover');


// display this ground reference data
var symbology = {color: 'red', fillColor: '00000000'};
Map.addLayer(landcover.style(symbology), {}, 'Ground Reference Features');

// feature engineering
var engineer_model_features = vegetation_indices(scottsHead);
print (engineer_model_features, 'engineer_model_features');


// extract the NDVI band
var engineer_model_features_NDVI = engineer_model_features.select('NDVI');
print (engineer_model_features_NDVI, 'engineer_model_features_NDVI');
Map.addLayer(engineer_model_features_NDVI, {}, "engineer_model_features_NDVI");

// Define a neighborhood with a kernel.
// scale the NDVI to 8-bits as this is required for glcm to work
var engineer_model_features_NDVI_8bits = engineer_model_features_NDVI.unitScale(-1, 1).multiply(255).toByte();
var square_munmorah = ee.Kernel.square({radius: 3});

// Compute the gray-level co-occurrence matrix (GLCM), get contrast.
var glcm_munmorah = engineer_model_features_NDVI_8bits.glcmTexture({size: 3});
print(glcm_munmorah, 'glcm_munmorah');
var glcm_munmorah_selectedBands = glcm_munmorah.select(['NDVI_asm', 'NDVI_contrast', 'NDVI_corr', 'NDVI_var', 'NDVI_idm', 'NDVI_savg', 'NDVI_ent']);
print(glcm_munmorah_selectedBands,'glcm_munmorah_selectedBands');

// make the glcm data an image collection to be used for the pca
var glcm_munmorah_selectedBands =ee.ImageCollection(glcm_munmorah_selectedBands);
//print (glcm_munmorah_selectedBands, 'glcm_munmorah_selectedBands ');

//**********************************************


// Principal components analysis- source:https://stackoverflow.com/questions/62436252/performing-pca-per-image-over-imagecollection-in-google-earth-engine

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
print("PCA imagery: ",Preped);

// select pca 1
var glcmBands_plus_pcaBands = Preped.toBands();
print('glcmBands_plus_pcaBands', glcmBands_plus_pcaBands);

// image to classify but not normalised
var engineer_model_features=engineer_model_features.addBands(glcmBands_plus_pcaBands.select('0_pc1'));



//*** snic segmentation*********************
// segmement the image using SNIC
var snic_seeds = ee.Algorithms.Image.Segmentation.seedGrid({
  size:20,
  gridType:"square"
  });

var snic = ee.Algorithms.Image.Segmentation.SNIC({
  image: scottsHead,
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

// add the segmentation layer
var engineer_model_features = engineer_model_features.addBands(snicClusters)

print(engineer_model_features, 'engineer_model_features2'); 

// normalise the image to use. this is a function to do this
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

// image to use for the classification. it is now normalized
var composite = normalize(engineer_model_features);
//print(composite,'Image2Classify' )

// rename bands
var var_args= ['Blue', 'Green', 'Red', 'NIR', 'NDVI', 'NDYI', 'RBNI', 'MYI', 'HRFI', 'GLCM', 'Clusters']
var composite =composite.rename(var_args)

//export image2classify  to use later
Export.image.toAsset({
  image: composite,
  description: 'GEEappScottsHeadImage2Classify',
  assetId: 'projects/ee-richcrabbe/assets/GEEappScottsHeadImage2Classify'
});

// sample training areas
var landcover2 = composite.sampleRegions({
  collection: landcover,
  properties: ['class_1'],
  scale: 0.5
  //tileScale: 16
});

print(landcover2, ' landcover2')

var landcoverClass1 = landcover2.filter(ee.Filter.eq('class_1', 0))
var landcoverClass2 = landcover2.filter(ee.Filter.eq('class_1', 1))

//print results
print(landcoverClass1.size(), 'landcover1'); 
print(landcoverClass2.size(), 'landcover2');

//limit the size to that of the class 1
var landcoverClass2 = landcoverClass2.limit(65)

// merge the feature collection
var landcover2 = landcoverClass2 .merge(landcoverClass1)


// partition training areas into training and test sets: apply 80-20 rule
var landcover3 = landcover2.randomColumn()
var trainingSample = landcover3.filter('random <= 0.8') //80% of the data would be for model training
var testSample = landcover3.filter('random > 0.8')  //20% of data for model testing

//************************************************************************** 
// Hyperparameter Tuning
//************************************************************************** 

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
        inputProperties:  composite.bandNames()
      });

    // compute error matrix 
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
  numberOfTrees: optimalNumTrees, //used the one from Birdies Beach
  bagFraction: optimalBagFraction  //used the one from Birdies Beach
}).train({
  features: trainingSample,  
  classProperty: 'class_1',
  inputProperties: composite.bandNames()
});


// classify the image
var finalClassification = composite.classify(optimalModel);

// apply the model to classify the image
var finalClassification = composite.classify(optimalModel);

// display the classified image
Map.setCenter(152.9881334183168,-30.734091023014066, 22)
Map.addLayer(finalClassification, {min: 0, max: 1, palette: ['green', 'red']}, 'Random Forest Classification of ALG'); 
//Map.addLayer(landcover, {}, 'reference data')

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
Map.setCenter(152.988,-30.732, 16)
Map.addLayer(styled.style({styleProperty: "style"}), {}, 'Reference Areas')

// appraise performance of the model 
var accuracy2 = testSample
      .classify(optimalModel)
      .errorMatrix('class_1', 'classification')


//Print the user's accuracy to the console
print('Validation Overall accuracy: ', accuracy2.accuracy())
print('Validation Consumer accuracy: ', accuracy2.consumersAccuracy())
print('Validation Producer accuracy: ', accuracy2.producersAccuracy())
print('Validation Kappa: ', accuracy2.kappa())
print('Validation fscore: ', accuracy2.fscore(1))

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

// end of code
print('End of Code')


```
