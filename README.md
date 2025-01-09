## Land use and Land cover Analysis in GEE - Random Forest
This repository contains two [Google Earth Engine](https://code.earthengine.google.com/) javascript scripts to perform a basic land 
use and land cover classification using a random forest classifier. The scripts are parts of a [publication](https://bnrc.springeropen.com/articles/10.1186/s42269-024-01286-z).

| Natural Color                                                                        | Classified LULC Map                                         |
|--------------------------------------------------------------------------------------|-------------------------------------------------------------|
|![](https://github.com/Okikiola-Michael/random_forest_lulc/blob/main/sentinel_nc.png) | ![](https://github.com/Okikiola-Michael/random_forest_lulc/blob/main/sentinel_classified.png)|




Using publicly available resources, this script was developed to perform a basic
random forest classification using [Landsat 9](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC09_C02_T1_L2)  and [Sentinel 2](https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR_HARMONIZED) imageries for a region of interest.

  1. The [`RF_Landsat`](https://github.com/Okikiola-Michael/random_forest_lulc/blob/main/RF_Landsat.md) script classifies a region of interest using a random forest classifier on Landsat 9 surface reflectance imageries
  2. The [`RF_Sentinel`](https://github.com/Okikiola-Michael/random_forest_lulc/blob/main/RF_Sentinel.md) script classifies a region of interest using a random forest classifier on Sentinel 2 surface reflectance imageries


The script perfroms the following:
  1. Selects Sentinel 2/Landsat 9 images based on a desired time interval
  2. Applies the necessary scaling factors and filters based on a desired cloud cover percentage
  3. Clips the study area and loads the LULC training/testing data
  4. Splits the LULC data into 75% for training and 25% for testing data
  5. Trains a Random Forest (RF) classifier
  6. Validate the RF model 
  7. Display the variable information
  
## INPUTS:
  1. Sentinel 2/Landsat 9 multispectral image
  2. Dates - Start and End dates
  3. Region of Interest - Study Area Geometry
  4. LULC data - Training and Testing data 

## OUTPUTS:
  1. Composite Image of the study location
  2. Classified Image of the study location
  3. Accuracy assessment results 
  4. Varaible Importance Chart

This script is free and by using/adapting it or any data derived with it, 
you agree to cite the following reference: 

## Publication: 

> Alegbeleye, O.M., Rotimi, Y.O., Shomide, P. et al. Land use land cover (LULC) analysis in Nigeria: a systematic review of data, methods, and platforms with future prospects. Bull Natl Res Cent 48, 127 (2024). https://doi.org/10.1186/s42269-024-01286-z

```
@article{alegbeleye2024land,
  title={Land use land cover (LULC) analysis in Nigeria: a systematic review of data, methods, and platforms with future prospects},
  author={Alegbeleye, Okikiola Michael and Rotimi, Yetunde Oladepe and Shomide, Patricia and Oyediran, Abiodun and Ogundipe, Oluwadamilola and Akintunde-Alo, Abiodun},
  journal={Bulletin of the National Research Centre},
  volume={48},
  number={1},
  pages={1--13},
  year={2024},
  publisher={SpringerOpen}
}
```
