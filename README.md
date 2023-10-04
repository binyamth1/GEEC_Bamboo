//# GEEC_Bamboo
//A Google Earth Engine script that map Highland Bamboo Forest using satellite, landscape and climate variables. 
//Loading the image
var image = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2').filterBounds(Andracha)
    .filterDate('2020-01-01', '2020-03-30').filterMetadata('CLOUD_COVER','Less_Than',05);
// Applies scaling factors.It is a pre-processing steps 
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBands, null, true).clip(Andracha)}
//}
var dataset = image.map(applyScaleFactors);
//Zoom to the Area of Interest (AOI)
Map.setCenter(35.5, 7.55, 11);
//Display of data
Map.addLayer(Andracha,{min:0, max:1},'Andracha');
Map.addLayer(And_Tr, {min:0, max:1}, 'GCPs');
// Compute NDVI
var dataset1 = dataset.median();
var GREEN = dataset1.select('SR_B3');
var NIR = dataset1.select('SR_B5');
var RED = dataset1.select('SR_B4');
var SWIR1 = dataset1.select('SR_B6');
var SWIR2 = dataset1.select('SR_B7');

Map.addLayer(NIR,{min: 0 , max: 1}, 'NIR');
//Create Training Data
var training_an = Forest.merge(Nonforest_vegetation).merge(Non_vegetation);
print(training_an);
//var bands_an = ['GREEN','NIR','RED','SWIR1','SWIR2'];
var bands_an = ['SR_B3','SR_B5','SR_B4','SR_B6','SR_B7'];
//Overlay the points on the imaery to get trainings
var trainImage_an = dataset1.select(bands_an).sampleRegions({
  collection: training_an,
  properties: ['Class'],
  scale: 30
  });
print(trainImage_an);
var trainingData_an = trainImage_an.randomColumn('rand');
var trainSet_an = trainingData_an.filter(ee.Filter.lt('rand', 0.95));
var testSet_an = trainingData_an.filter(ee.Filter.gte('rand', 0.05));
//Map.addLayer(trainSetm, {color: '000000'}, 'ROI Train', true );
//Map.addLayer(testSetm, {color: 'FF0000'}, 'ROI Test', true );
// Classification Model : this knows what LULC classes are based on the training data provided
var classifier_an = ee.Classifier.smileCart().train(trainSet_an,'Class', bands_an);
var classifiertest_an = ee.Classifier.smileCart().train(testSet_an, 'Class', bands_an);
//Classify the image
var classified_an = dataset1.select(bands_an).classify(classifier_an);
var classifiedtest_an = dataset1.select(bands_an).classify(classifiertest_an);

//print(classifiedm.getInfo());
//Define a palette for the classification
var landcoverPalette_an = [
'#20a10c', // Forest(0)
'#e1ff17', // Nonforest_vegetation(1)
'#b2f0b1', // Non_vegetation(2)
];
Map.addLayer(classified_an.clip(Andracha), {palette: landcoverPalette_an, min:0, max:3}, 'Classification CART_M');
//Accuracy Assessment
var confusionMatrix_an = ee.ConfusionMatrix(testSet_an.classify(classifier_an)
.errorMatrix({
  actual: 'Class',
  predicted: 'Classification'}));
print ('Confusion Matrix:', confusionMatrix_an);
print ('Overall Accuracy', confusionMatrix_an.accuracy());
////Export classified map to Google Drive
Export.image.toDrive({
image: classified_an,
description: "Landsat_CART_M_1",
scale: 30,
region: Andracha,
maxPixels:1e13,
});
