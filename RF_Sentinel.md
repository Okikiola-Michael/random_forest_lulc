## Author: Okikiola Michael Alegbeleye (alegbeleyeokiki@gmail.com)

Using publicly available resources, this script was developed to perform a 
random forest classification using Landsat 9 images for a given study location 
(Here, we used Omo Forest in Nigeria).

This script is free and by using/adapting it or any data derived with it, 
you agree to cite the following reference: 
Publication:


This script perfroms the following:
  1. Selects Sentinel 2 images based on a desired time interval
  2. Applies the necessary scaling factors and filters based on a desired cloud cover percentage
  3. Clips the study area and loads the LULC training/testing data
  4. Splits the LULC data into 75% for training and 25% for testing data
  5. Trains a Random Forest (RF) classifier
  6. Validate the RF model 
  7. Display the variable information
  
#### INPUTS:
  1. Sentinel 2 multispectral image
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


// Function to mask clouds using the Sentinel-2 QA band
    // param {ee.Image} image Sentinel-2 image
    // return {ee.Image} cloud masked Sentinel-2 image
 
 
function maskS2clouds(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}
```

This line of code performs the following:
  1. selects the needed bands, 
  2. filters the images' cloud cover (Here, cloud cover < 1)
  3. applies the median operator to reduce the overlap, and 
  4. clips the image with the Omo forest boundary feature  


```javascript
// Select the Harmonized Sentinel 2 Multispectral Images
var omo_image = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
                  // Selects dates 
                  .filterDate(start_date, end_date)       
                  // Pre-filter to get less cloudy granules.
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',10))
                  // Applies the cloud masking function
                  .map(maskS2clouds)
                  // Applies the median operator to reduce the collection and clips the study area
                  .median().clip(omo);


var visualization = {
  min: 0.0,
  max: 0.5,
  bands: ['B8', 'B4', 'B3'],
};
```

_The clipped image has several bands, however, to perform a basic LULC,
we are only selecting the few (Blue, Green, Red, Red Edge 1, Red Edge 2
Red Edge 3, Near Infrared, RedEdge 4, SWIR 1 and 2)_

 ```javascript
var omo_sentinel = omo_image.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7', 'B8', 'B8A', 'B11', 'B12']);

// Print the image properties
print(omo_sentinel);
```

## Band combinations for different features:
  1.  Bands 12, 11, and 4 for urban, 
  2.  Bands 4, 3, and 2 for Natural Color; 
  3.  Bands 11, 8, and 4 for Vegetation Analysis; 
  4.  Band 8, 4, and 3 for color infrared.

```javascript
// Visualization parameter using the natural color band combination
var vis = {
  bands: ['B4', 'B3', 'B2'],
  min: 0.0,
  max: 0.2,
};

// Set map center
Map.centerObject(omo);

// Display the the map layer (Add false to not display map): 
  // Map.addLayer(omo_sentinel, vis, "Omo Sentinel NC", false);
Map.addLayer(omo_sentinel, vis, "Omo Sentinel NC");
```

## Training/Validation Data 
_The total number of samples is 554 and were randomly generated._ _Users can upload their data as an asset_ 

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
var training = omo_sentinel.sampleRegions({
    collection: trainingsample,
    properties: ['level'],
    scale: 10,
});

print(training, 'Training Band values');

// Validation sample extraction
var validation = omo_sentinel.sampleRegions({
    collection: validationsample,
    properties: ['level'],
    scale:10,
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
var classified = omo_sentinel.classify(randomforest);
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
/*  Variable Importance */
var model_explain = randomforest.explain();
print(model_explain, 'RF Model Properties');

/*  Variable Importance of RF Classifier  */
var variable_importance = ee.Feature(null, ee.Dictionary(model_explain).get('importance'));

/*  Chart of Variable Importance of RF Classifier    */ 
var options = {
        title: 'Random Forest: Bands Variable Importance',
        legend: {position: 'none'},
        hAxis: {title: 'Sentinel Bands'},
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
