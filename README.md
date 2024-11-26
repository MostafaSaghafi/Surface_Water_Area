# Surface Water Area
Calculate the Surface Water Area using Sentinel-2 by MNDWI index in GEE

------

### **1. Center the Map on the Geometry**
```javascript
Map.centerObject(geometry);
```
- **Purpose**: Centers the map view on the `geometry` object (a region of interest, such as a polygon or point).
- **`geometry`**: A variable that represents the area of interest. It must be defined elsewhere in the script.

---

### **2. Define a Date Range**
```javascript
var startDate = '2013-01-01';
var endDate = '2023-12-29';
```
- **Purpose**: Sets the start and end dates for filtering satellite images.
- **`startDate` and `endDate`**: Strings representing the time range for the analysis.

---

### **3. Load Landsat 8 Surface Reflectance Data**
```javascript
var collection = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
                  .filterBounds(geometry)
                  .filterDate(startDate, endDate)
                  .filter(ee.Filter.lessThan('CLOUD_COVER', 20));
```
- **`ee.ImageCollection`**: Loads a dataset of Landsat 8 surface reflectance images.
- **`.filterBounds(geometry)`**: Filters the images to include only those that intersect with the `geometry` region.
- **`.filterDate(startDate, endDate)`**: Filters the images to include only those captured within the specified date range.
- **`.filter(ee.Filter.lessThan('CLOUD_COVER', 20))`**: Filters the images to include only those with less than 20% cloud cover (to ensure clearer images).

---

### **4. Define a Function to Calculate MNDWI**
```javascript
var calculateMNDWI = function(image) {
  var nir = image.select('SR_B5'); // NIR band
  var swir = image.select('SR_B6'); // SWIR band
  var mndwi = nir.subtract(swir).divide(nir.add(swir)).rename('MNDWI');
  return image.addBands(mndwi);
};
```
- **Purpose**: Calculates the Modified Normalized Difference Water Index (MNDWI) for each image.
1. **`image.select('SR_B5')`**: Selects the Near-Infrared (NIR) band from the image.
2. **`image.select('SR_B6')`**: Selects the Shortwave Infrared (SWIR) band from the image.
3. **`nir.subtract(swir).divide(nir.add(swir))`**: Computes the MNDWI using the formula:

```javascript
   MNDWI = {NIR} - {SWIR} / {NIR} + {SWIR}
```
   
4. **`.rename('MNDWI')`**: Renames the resulting band as `MNDWI`.
5. **`image.addBands(mndwi)`**: Adds the MNDWI band to the original image.

---

### **5. Apply the MNDWI Calculation to the Image Collection**
```javascript
var mndwiCollection = collection.map(calculateMNDWI);
```
- **Purpose**: Applies the `calculateMNDWI` function to every image in the `collection`, creating a new collection with an added `MNDWI` band.

---

### **6. Create a Time Series Chart of MNDWI**
```javascript
var chart = ui.Chart.image.series({
  imageCollection: mndwiCollection.select('MNDWI'),
  region: geometry,
  reducer: ee.Reducer.max(),
  scale: 30
}).setOptions({
  title: 'MNDWI Time Series',
  hAxis: {title: 'Date'},
  vAxis: {title: 'MNDWI'},
  lineWidth: 1,
  pointSize: 3
});
```
- **Purpose**: Creates a time series chart to visualize MNDWI changes over time.
1. **`ui.Chart.image.series`**: Generates a time series chart from an image collection.
2. **`imageCollection: mndwiCollection.select('MNDWI')`**: Uses only the `MNDWI` band from the `mndwiCollection`.
3. **`region: geometry`**: Restricts the analysis to the `geometry` region.
4. **`reducer: ee.Reducer.max()`**: Aggregates the maximum MNDWI value within the region for each image.
5. **`scale: 30`**: Specifies the spatial resolution (30 meters per pixel, the resolution of Landsat 8).
6. **`.setOptions({...})`**: Configures the chart's appearance (e.g., title, axis labels, line width, and point size).

---

### **7. Display the Chart**
```javascript
print(chart);
```
- **Purpose**: Prints the MNDWI time series chart to the GEE console for visualization.

---

### **8. Define a Function to Calculate Water Area**
```javascript
var calculateWaterArea = function(image) {
  var timestamp = ee.Date(image.get('system:time_start'));
  var waterMask = image.select('MNDWI').gt(0);
  var waterArea = waterMask.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: geometry,
    scale: 30,
    maxPixels: 1e9
  }).get('MNDWI');
  var feature = ee.Feature(null, {
    'timestamp': timestamp,
    'water_area': waterArea
  });
  return feature;
};
```
1. **`ee.Date(image.get('system:time_start'))`**: Extracts the timestamp of the image.
2. **`image.select('MNDWI').gt(0)`**: Creates a binary water mask where pixels with `MNDWI > 0` are classified as water.
3. **`waterMask.reduceRegion({...})`**: Calculates the total water area:
   - **`reducer: ee.Reducer.sum()`**: Sums all pixels classified as water.
   - **`geometry: geometry`**: Restricts the calculation to the `geometry` region.
   - **`scale: 30`**: Specifies the spatial resolution.
   - **`maxPixels: 1e9`**: Sets a limit on the number of pixels processed.
4. **`.get('MNDWI')`**: Retrieves the computed water area.
5. **`ee.Feature(null, {...})`**: Creates a feature with properties for the timestamp and water area.

---

### **9. Apply the Water Area Calculation**
```javascript
var waterAreas = mndwiCollection.map(calculateWaterArea);
```
- **Purpose**: Applies the `calculateWaterArea` function to every image in the `mndwiCollection`, creating a collection of features with water area and timestamp.

---

### **10. Print the Water Areas**
```javascript
print('Water areas for each image:', waterAreas);
```
- **Purpose**: Prints the computed water areas for each image to the GEE console.

---

### **11. Export the Water Areas to a CSV File**
```javascript
Export.table.toDrive({
  collection: waterAreas,
  description: 'water_areas_by_time',
  fileFormat: 'CSV'
});
```
- **Purpose**: Exports the `waterAreas` feature collection as a CSV file to Google Drive.
1. **`collection: waterAreas`**: Specifies the feature collection to export.
2. **`description: 'water_areas_by_time'`**: Sets the export task name.
3. **`fileFormat: 'CSV'`**: Specifies the output format as CSV.

---

### **Summary**
- The script calculates the MNDWI for Landsat 8 images over a specified region and time range.
- It generates a time series chart of MNDWI values.
- It computes the water area for each image using a threshold on MNDWI and exports the results as a CSV file.
