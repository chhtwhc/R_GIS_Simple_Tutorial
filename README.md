# Simple Tutorial for GIS in R
 
This is my project for introducing the basic fundamentals of GIS and how to operate it in R (with R code examples).

# Data Download

- Species data are download from TBN:<br> https://www.tbn.org.tw/ <br>
- Annual temperature data are download from CHELSA:<br> https://chelsa-climate.org/ <br>
- Taiwan map is download from Taiwan Government Open Data Platform:<br> https://data.nat.gov.tw/

# R Code Examples

### Setting environment

```r
# Set working directory
setwd(dirname(rstudioapi::getSourceEditorContext()$path))

# Load libraries
library(tidyverse)  # Process attribute data
library(sf)  # Process vector data
library(terra)  # Process raster data
library(mapview)  # Data visualization
```

### Generate, load, and save simple feature (sf)

```r
# Load raw CSV data
Qg_raw = read.csv("./Data/Quercus_gilva.csv")

# Check and clean data
head(Qg_raw)
summary(Qg_raw)

Qg = Qg_raw %>%
  select(經度.十進位., 緯度.十進位.) %>%
  rename(Longitude = 經度.十進位., Latitude = 緯度.十進位.) %>%
  na.omit()

# Convert to simple feature (sf)
Qg_sf = Qg %>%
  st_as_sf(coords = c("Longitude", "Latitude")) %>%
  st_set_crs(4326)

head(Qg_sf)

# Save simple feature
save(Qg_sf, file = "./Data/Quercus_gilva.RData")  # Save as RData file
st_write(Qg_sf, dsn = "./Data/Quercus_gilva.shp")  # Save as Shapefile
```

### Visualize the data

```r
# Load and process the data
Taiwan = read_sf(dsn = "./Data/Taiwan_county.shp", options = "ENCODING=BIG5") %>%
  select(COUNTY, geometry)

# Transform the CRS if the CRS are different
st_crs(Qg_sf)  # EPSG: 4326
st_crs(Taiwan)  # EPSG: 3826

st_crs(Qg_sf) == st_crs(Taiwan)  # FALSE

Taiwan_4326 = st_transform(Taiwan, crs = st_crs(Qg_sf)) %>% 
  st_make_valid()

# Visualize the location and basic information
plot(Taiwan_4326$geometry)
plot(Qg_sf, add = T, pch = 16, col = "tomato")

# (What happen if you don't transform)
plot(Taiwan$geometry)
plot(Qg_sf, add = T, pch = 16, col = "tomato")

# Useful package and function: mapview()
mapview(Qg_sf)
```

### Extract information from raster data

```r
# Load the temperature raster data
Chelsa_files = "./Data/CHELSA_bio1_1981-2010_V.2.1.tif"  # Set the file route to access the .tif file
Chelsa_tif = rast(Chelsa_files)  # Read .tif files
plot(Chelsa_tif)

# Check and set the CRS of the data
# (You can also use crs() function from the terra package)
st_crs(Qg_sf)  # EPSG: 4326
st_crs(Chelsa_tif)  # EPSG: 4326
st_crs(Qg_sf) == st_crs(Chelsa_tif)  # TRUE

# Extract environmental condition for each point
Qg_temp = extract(Chelsa_tif, Qg_sf) %>% 
  rename(AnnualTemp = `CHELSA_bio1_1981-2010_V.2.1`)
class(Qg_temp)  # Dataframe

# Add the attribute to sf
Qg_sf_temp = Qg_sf %>% 
  mutate(AnnualTemp = Qg_temp$AnnualTemp)

class(Qg_sf_temp)  # sf, data.frame

mapview(Qg_sf_temp)
```

### Extract information from polygon data

```r
Qg_sf_county = st_join(Qg_sf, Taiwan_4326, left = TRUE)
mapview(Qg_sf_county)
```

### Filter data by geometry

```r
# Select specific region (e.g., New Taipei City)
NewTaipei = Taiwan_4326 %>% filter(COUNTY == "新北市")

# Filter out the data points within specific region by intersection
Qg_inter = st_intersection(Qg_sf, NewTaipei)

# Visualize
mapview(Qg_sf, col.regions = "lightblue", alpha = 0.7) +
  mapview(Qg_inter, col.regions = "pink", alpha = 0.7) +
  mapview(NewTaipei)
```

### Filter data by attribute values

```r
# Filter out the data by category
Qg_sf_newTaipei = Qg_sf_county %>% filter(COUNTY == "新北市")

# Visualize
mapview(Qg_sf, col.regions = "lightblue", alpha = 0.7) +
  mapview(Qg_inter, col.regions = "pink", alpha = 0.7) +
  mapview(Qg_sf_newTaipei, col.regions = "orange", alpha = 0.7) +
  mapview(NewTaipei)

# Filter out the data by values
Qg_sf_temp20 = Qg_sf_temp %>% filter(AnnualTemp > 20)

# Visualize
mapview(Qg_sf_temp, col.region = rev(colorRampPalette(RColorBrewer::brewer.pal(11, "RdBu"))(100))) +
  mapview(Qg_sf_temp20, col.regions = "green", alpha = 0.7, legend = F)
```

### Buffer and clip raster data

```r
# Merge all features into one single feature
Taiwan_merge = st_union(Taiwan_4326) %>% st_as_sf()

# Buffer
Taiwan_merge_500 = st_buffer(Taiwan_merge, 500) %>% st_as_sf()

# Clip the raster by polygon
Chelsa_tif_clip = Chelsa_tif %>% 
  crop(Taiwan_merge_500) %>% 
  mask(Taiwan_merge_500)

# Visualization
mapview(Chelsa_tif_clip, col.region = rev(colorRampPalette(RColorBrewer::brewer.pal(11, "RdBu"))(100))) +
  mapview(Taiwan_merge, col.region = "brown4", legend = F) +
  mapview(Taiwan_merge_500, col.region = "grey", legend = F)
```
