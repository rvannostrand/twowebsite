---
authors:
- admin
categories: []
date: "2021-02-05T00:00:00Z"
image:
  caption: ""
  focal_point: ""
lastMod: "2021-09-05T00:00:00Z"
projects: []
subtitle: ''
summary: ''
tags: []
title: Funtion to Extract Planet Labs Data by Parcel in R
weight: 2
---

The follow code can be copied into R and ran as the function "PlanetToNDVI". You can change the band math and function name to produce any index you wish, or a single band. The output is a data frame representing administrative boundaries and a column containing the average index values from Planet Labs data. Note that this requires 2 files in your root directory: 1.) 1 or many Planet labs products and a shapefile representing some sort of administrative boundary (Parcels, Census Blocks, Counties etc.).




##Prelims

```r
library(tmap) 
library(mapview)
library(sf)
```

```
## Linking to GEOS 3.9.1, GDAL 3.2.1, PROJ 7.2.1; sf_use_s2() is TRUE
```

```r
library(raster)
```

```
## Loading required package: sp
```

```r
library(stringr)
library(questionr)
```

```
## 
## Attaching package: 'questionr'
```

```
## The following object is masked from 'package:raster':
## 
##     freq
```

```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:raster':
## 
##     intersect, select, union
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
library(rgdal)
```

```
## Please note that rgdal will be retired by the end of 2023,
## plan transition to sf/stars/terra functions using GDAL and PROJ
## at your earliest convenience.
## 
## rgdal: version: 1.5-28, (SVN revision 1158)
## Geospatial Data Abstraction Library extensions to R successfully loaded
## Loaded GDAL runtime: GDAL 3.2.1, released 2020/12/29
## Path to GDAL shared files: C:/Users/blair/OneDrive/Documents/R/win-library/4.1/rgdal/gdal
## GDAL binary built with GEOS: TRUE 
## Loaded PROJ runtime: Rel. 7.2.1, January 1st, 2021, [PJ_VERSION: 721]
## Path to PROJ shared files: C:/Users/blair/OneDrive/Documents/R/win-library/4.1/rgdal/proj
## PROJ CDN enabled: FALSE
## Linking to sp version:1.4-6
## To mute warnings of possible GDAL/OSR exportToProj4() degradation,
## use options("rgdal_show_exportToProj4_warnings"="none") before loading sp or rgdal.
## Overwritten PROJ_LIB was C:/Users/blair/OneDrive/Documents/R/win-library/4.1/rgdal/proj
```

```r
library(rgeos)
```

```
## rgeos version: 0.5-9, (SVN revision 684)
##  GEOS runtime version: 3.9.1-CAPI-1.14.2 
##  Please note that rgeos will be retired by the end of 2023,
## plan transition to sf functions using GEOS at your earliest convenience.
##  GEOS using OverlayNG
##  Linking to sp version: 1.4-6 
##  Polygon checking: TRUE
```

```r
memory.limit(size=100000) #forget everything else you have running
```

```
## [1] 1e+05
```

##Single image example


```r
PlanetImage<- stack("Data/10_12_2018_PSScene4Band_Explorer/files/20181012_154650_1_104a_3B_AnalyticMS_SR_clip.tif") #Upload 4band Image
PlanetImage

ndvi <- (PlanetImage[[4]] - PlanetImage[[1]])/(PlanetImage[[4]] + PlanetImage[[1]]) #make NDVI

tmap_mode("view")

tm_shape(ndvi) + 
  tm_raster() #Plot simple NDVI
```


##Extract Average Index value from Multiple Images by Parcel as Function

```r
PlanetToNDVI<-function(ImageLocation,FileOutput,ImageDate,AssetType,UOA){
  
  current.list <- list.files(path=paste(ImageLocation,ImageDate,"_",AssetType,"_Explorer/files/",  sep =""),
     pattern ="SR_clip.tif$", 
     full.names=TRUE)

images<- lapply(current.list, stack)

imageMosaic<-mosaic(images[[1]],images[[2]], fun=mean) # unhash for n number of images in study area. Alternatively write a loop...
#imageMosaic<-mosaic(imageMosaic,images[[3]], fun=mean)
#imageMosaic<-mosaic(imageMosaic,images[[4]], fun=mean)
#imageMosaic<-mosaic(imageMosaic,images[[5]], fun=mean)
#imageMosaic<-mosaic(imageMosaic,images[[6]], fun=mean)
#imageMosaic<-mosaic(imageMosaic,images[[7]], fun=mean)
#imageMosaic<-mosaic(imageMosaic,images[[8]], fun=mean)
  
ndvi <- (imageMosaic[[4]] - imageMosaic[[3]])/(imageMosaic[[4]] + imageMosaic[[3]])

Parcels = read_sf(UOA) #Load in its of analysis shapefile and reproject image coordinate system
Parcels<-st_transform(Parcels, crs(imageMosaic))


ndvi[ndvi < 0.2] <- NA #mask off index below threshold to try and omit roof, driveway etc. Consider deriving threshold from ML algo at some point...

ex <- extract(ndvi, 
    Parcels, 
    fun=mean,
    na.rm=TRUE,
    df=TRUE)

ex$GlobalID <-Parcels$GlobalID

names(ex)[2] <- ImageDate

write.csv(ex, paste(FileOutput,ImageDate,".csv", sep=""))

}
```

##Run Function

```r
PlanetToNDVI("Data/",
             "Ouputs/",
             "10_12_2018", 
             "PSScene4Band",
             "ParcelAOI.shp")
```

