# Surface Water Area
Calculate the Surface Water Area using Sentinel-2 by MNDWI index in GEE

------

**Part 1: Calculating and Charting MNDWI**

1. **`Map.centerObject(geometry);`**: This line centers the map view on the region of interest defined by the `geometry` variable.  `geometry` is assumed to be a previously defined object representing the area of interest (e.g., a polygon or point).

2. **`var startDate = '2013-01-01';`** and **`var endDate = '2023-12-29';`**: These lines define the start and end dates for filtering the Landsat imagery.

3. **`var collection = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")...`**: This section loads Landsat 8 Collection 2 Level 2 surface reflectance data. It filters the data based on several criteria:
    - `.filterBounds(geometry)`: Filters the collection to include only images that intersect the specified `geometry`.
    - `.filterDate(startDate, endDate)`: Filters the collection to include only images within the specified date range.
    - `.filter(ee.Filter.lessThan('CLOUD_COVER', 20))`: Filters the collection further, keeping only images with less than 20% cloud cover.

4. **`var calculateMNDWI = function(image) { ... };`**: This defines a function called `calculateMNDWI` that takes a single Landsat image as input and calculates the Modified Normalized Difference Water Index (MNDWI).
    - `var nir = image.select('SR_B5');`: Selects the Near-Infrared (NIR) band (Band 5).
    - `var swir = image.select('SR_B6');`: Selects the Shortwave Infrared (SWIR) band (Band 6).
    - `var mndwi = nir.subtract(swir).divide(nir.add(swir)).rename('MNDWI');`: Calculates the MNDWI using the formula: (NIR - SWIR) / (NIR + SWIR) and renames the resulting band to 'MNDWI'.
    - `return image.addBands(mndwi);`: Adds the calculated MNDWI band to the original image and returns the modified image.

5. **`var mndwiCollection = collection.map(calculateMNDWI);`**: This line applies the `calculateMNDWI` function to every image in the `collection`.  The result is a new collection, `mndwiCollection`, where each image now has an additional band containing the MNDWI.

6. **`var chart = ui.Chart.image.series({ ... });`**: This creates a time series chart of MNDWI values.
    - `imageCollection: mndwiCollection.select('MNDWI')`: Specifies the image collection and the band ('MNDWI') to use for the chart.
    - `region: geometry`: Defines the region over which to calculate the MNDWI values.
    - `reducer: ee.Reducer.max()`: Uses the maximum MNDWI value within the region for each image.
    - `scale: 30`: Sets the spatial resolution (in meters) for the calculations.

7. **`setOptions({ ... });`**: Customizes the chart's appearance, including title, axis labels, line width, and point size.

8. **`print(chart);`**: Displays the generated chart in the Earth Engine console.


**Part 2: Calculating and Exporting Water Area**

1. **`var calculateWaterArea = function(image) { ... };`**:  This function calculates the water area for each image.
    - `var timestamp = ee.Date(image.get('system:time_start'));`: Gets the timestamp of the image.
    - `var waterMask = image.select('MNDWI').gt(0);`: Creates a binary water mask by thresholding the MNDWI band. Pixels with MNDWI > 0 are considered water (value 1), others are not water (value 0).
    - `var waterArea = waterMask.reduceRegion({ ... }).get('MNDWI');`: Calculates the sum of the pixels in the `waterMask` within the given region.  This effectively counts the water pixels. The scale is set to 30 meters.
    - `var feature = ee.Feature(null, { ... });`: Creates a feature (a table row) with the timestamp and calculated water area.
    - `return feature;`: Returns the created feature.

2. **`var waterAreas = mndwiCollection.map(calculateWaterArea);`**: Applies the `calculateWaterArea` function to each image in the `mndwiCollection`, creating a collection of features, where each feature represents the water area at a specific time.

3. **`print('Water areas for each image:', waterAreas);`**: Prints the `waterAreas` feature collection to the console.

4. **`Export.table.toDrive({ ... });`**: Exports the `waterAreas` feature collection to a CSV file in Google Drive.
    - `collection: waterAreas`: Specifies the collection to export.
    - `description: 'water_areas_by_time'`: Sets the description for the export task.
    - `fileFormat: 'CSV'`: Specifies the output file format as CSV.


