## Author: Okikiola Michael Alegbeleye (alegbeleyeokiki@gmail.com)

Using publicly available resources, this script was developed to perform a 
random forest classification using Landsat 9 images for a given study location 
(Here, we used Omo Forest in Nigeria).

This script is free and by using/adapting it or any data derived with it, 
you agree to cite the following reference: 
Publication:


This script perfroms the following:
  1. Selects Landsat 9 images based on a desired time interval
  2. Applies the necessary scaling factors and filters based on a desired cloud cover percentage
  3. Clips the study area and loads the LULC training/testing data
  4. Splits the LULC data into 75% for training and 25% for testing data
  5. Trains a Random Forest (RF) classifier
  6. Validates the RF model 
  7. Displays the variable information
  
#### INPUTS:
  1. Landsat 9 image
  2. Dates - Start and End dates
  3. Region of Interest - Study Area Geometry
  4. LULC data - Training and Testing data 

#### OUTPUTS:
  1. Composite Image of the study location
  2. Classified Image of the study location
  3. Accuracy assessment results 
  4. Varaible Importance Chart

```javascript
// Dates
var start_date = "2022-11-01";
var end_date = "2022-12-30";

// Study Location - Omo Forest in Nigeria saved as an asset
var omo = ee.FeatureCollection("users/alegbeleye_okikiola_michael/omo_forest");


//  Selecting Landsat data
var dataset = ee.ImageCollection('LANDSAT/LC09/C02/T1_L2')
    .filterDate(start_date, end_date);

// This Function applies scaling factors to Landsat bands
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBands, null, true);
}

// Apply the function to the selected landsat data
dataset = dataset.map(applyScaleFactors);
```

This line of code performs the following:
  1. selects the needed bands, 
  2. filters the images' cloud cover (Here, cloud cover < 1)
  3. applies the median operator to reduce the overlap, and 
  4. clips the image with the Omo forest boundary feature  

```javascript
var omo_landsat = dataset.select(['SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6','SR_B7'])
                  .filterMetadata('CLOUD_COVER', 'less_than', 1)
                  .median().clip(omo);
                
// Print the clipped image properties
print(omo_landsat);
```

## Band combinations for different features:
  1.  Bands 7, 6, and 4 for urban and waterways, 
  2.  Bands 4, 3, and 2 for Natural Color; 
  3.  Bands 6, 5, and 4 for Vegetation Analysis; 
  4.  Band 5, 4, and 3 for color infrared.

```javascript
// Visualization parameter using the natural color band combination
var vis = {
  bands: ['SR_B4', 'SR_B3', 'SR_B2'],
  min: 0.0,
  max: 0.2,
};

// Set map center
Map.centerObject(omo);

// Display the the map layer (Add false to not display map: 
  // Map.addLayer(omoclip, vis, "Omo Forest NC", false);
Map.addLayer(omo_landsat, vis, "Omo Forest NC");
```

## Training/Validation Data 


 | Land Cover Types        |  Codes    | Number of Points|
 |-------------------------|-----------|-----------------|
 | High vegetation         |     1     |     241         |
 | Built-up                |     2     |     147         |
 | Low vegetation          |     3     |     113         |
 | Waterbodies             |     4     |     53          |


```javascript
// Load the training/testing data from the assets
var lulc_data = ee.FeatureCollection("users/alegbeleyeokiki/lulc_traintest_data_new");

// Print the point data properties
print(lulc_data, 'LULC Point data');

// Add the points to the map
Map.addLayer(lulc_data, {color : "brown"}, "LULC Point data");
```

For random classification, you can create a random points and split them into _training_
and _validation_ points or you can generate seperate points for `training` and `validation`. 

```javascript
Split the points into training and validation points
var sample = lulc_data.randomColumn();

// 75% training sample
var trainingsample = sample.filter('random <= 0.75'); 
// Print the training data
print(trainingsample, 'Training sample');

// 25% validation data
var validationsample = sample.filter('random > 0.75');
// Print the validation data
print(validationsample, 'Validation sample');
```

### Predictor variables to use for classification

_Several bands, topographic data, climatic data and indices can be used as predictors
when classifying LULC. Here, we only used selected Sentinel bands.
The spectral values of the bands are extracted using the training and validation points.
The "level" is unique to different land covers._

```javascript
// Training sample extraction
var training = omo_landsat.sampleRegions({
    collection: trainingsample,
    properties: ['level'],
    scale: 30,
});

print(training, 'Training Band values');

// Validation sample extraction
var validation = omo_landsat.sampleRegions({
    collection: validationsample,
    properties: ['level'],
    scale:30,
});

print(validation, 'Validation Band values');
```

### RF Classifier Model Building
_The number of decision tree used was 50 and this is flexible._

```javascript
// Training a Random Forest model
var randomforest = ee.Classifier.smileRandomForest(50).train({
  features: training,
  classProperty: 'level'});

// Running the Random Forest classification
var classified = omo_landsat.classify(randomforest);
print(classified, 'Classified');

// Land use and land cover palette
var lulc_palette = [
  'green',  //  Forest
  'grey',   //  Built-up/Bare land
  'yellow', //  LoW_vegetation 
  'blue',   //  Waterbody
];

// Display the classified Omo forest
Map.addLayer(classified, {palette: lulc_palette, min: 1, max: 4}, 'Classified map');
```


###  Accuracy Assessment

```javascript
var model_validation = validation.classify(randomforest);

/* Confusion matrix and the overall accuracy for the validation sample */
var validation_accuracy = model_validation.errorMatrix('level', 'classification');

/* Print the results*/
print(validation_accuracy, 'Validation Error matrix');
print(validation_accuracy.accuracy(), 'Overall Validation Accuracy');
```

###  Variable Importance 

```javascript
var model_explain = randomforest.explain();
print(model_explain, 'RF Model Properties');

/*  Variable Importance of RF Classifier  */
var variable_importance = ee.Feature(null, ee.Dictionary(model_explain).get('importance'));

/*  Chart of Variable Importance of RF Classifier    */ 
var options = {
        title: 'Random Forest Variable Importance',
        legend: {position: 'none'},
        hAxis: {title: 'Landsat Bands'},
        vAxis: {title: 'Variable Importance'}
      };
      
var chart =
    ui.Chart.feature.byProperty(variable_importance)
      .setChartType('ColumnChart')
      .setOptions(options);
      
      
/*   Chart: Location and Plot     */ 
chart.style().set({
  position: 'bottom-right',
  width: '350px',
  height: '350px'
});

// Display the variable importance chart
Map.add(chart);
```

