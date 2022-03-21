# geospatial-open-data

<a href="http://creativecommons.org/licenses/by-nc/4.0/" rel="license"><img style="border-width: 0;" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" alt="Creative Commons License" /></a>
This tutorial is licensed under a <a href="http://creativecommons.org/licenses/by-nc/4.0/" rel="license">Creative Commons Attribution-NonCommercial 4.0 International License</a>.

# Acknowledgements
This lab is sourced from and adapted from Professor Noli Brazil's course on <a href="https://cougrstats.wordpress.com/2018/10/12/webscraping-in-r/"> "Spatial Methods in Community Research" </a> and Manuel Giomond's <a href="https://mgimond.github.io/Spatial/index.html"> "Intro to GIS and Spatial Analysis" </a> curriculum.

# Introduction

During class, we learned to process spatial data in R--particularly, areal/polygon data. This lab will talk more broadly about how to access and clean data from Open Data portals while also incorporating point data. The learning objectives are as follows:

1. Learn how to read in point data
2. Gain better understanding of Coordinate Reference Systems
3. Learn how to reproject spatial data
4. Learn how to bring in data from OpenStreetMap
5. Learn how to map point data

To achieve these objectives, we will first examine the spatial distribution of homeless encampments in the City of Los Angeles using 311 data downloaded from the city’s <a href="https://data.lacity.org/"> open data portal</a>. This lab guide follows closely and supplements the material presented in the textbook <a href="https://geocompr.robinlovelace.net/"> Geocomputation with R</a>

# Packages

install.packages("sf")
install.packages("tidyverse")
install.packages("units")
install.packages("tmap")
install.packages("tidycensus")
install.packages("tigris")
install.packages("rmapshaper")
install.packages("tidygeocoder")
install.packages("leaflet")
install.packages("osmdata")

# Reading in census tract data

We will need to bring in census tract polygon features and racial composition data from the 2015-2019 American Community Survey using the Census API and keep tracts within Los Angeles city boundaries using a clip. The code for accomplishing these tasks is below. There are  comments embedded within the code that briefly explain what each chunk is doing. The code ```rename_with(~ sub("E$", "", .x), everything())``` eliminates the E at the end of the variable names for the estimates. Remember in a prior lab how we brought in a long dataset and used ```spread()``` to go from long to wide?

```R
# Bring in census tract data. 
ca.tracts <- get_acs(geography = "tract", 
              year = 2017,
              variables = c(tpop = "B01003_001", tpopr = "B03002_001", 
                            nhwhite = "B03002_003", nhblk = "B03002_004",
                             nhasn = "B03002_006", hisp = "B03002_012"),
              state = "CA",
              survey = "acs5",
              output = "wide",
              geometry = TRUE)
# Make the data tidy, calculate percent race/ethnicity, and keep essential vars.
ca.tracts <- ca.tracts %>% 
  rename_with(~ sub("E$", "", .x), everything()) %>%
  mutate(pnhwhite = nhwhite/tpopr, pnhasn = nhasn/tpopr, 
              pnhblk = nhblk/tpopr, phisp = hisp/tpopr) %>%
  dplyr::select(c(GEOID,tpop, pnhwhite, pnhasn, pnhblk, phisp))  

# Bring in city boundary data
pl <- places(state = "CA", cb = TRUE)

# Keep LA city
la.city <- filter(pl, NAME == "Los Angeles")

#Clip tracts using LA boundary
la.city.tracts <- ms_clip(target = ca.tracts, clip = la.city, remove_slivers = TRUE)
```

# Reading in point data
Point data give us the locations of objects or events within an area. Events can be things like crimes and car accidents. Objects can be things like trees, houses, jump bikes or even people, such as the locations of where people were standing during a protest.

Often you will receive point data in tabular (non-spatial) form. These data can be in one of two formats

1. Point longitudes and latitudes (or X and Y coordinates)
2. Street addresses

If you have longitudes and latitudes, you have all the information you need to make the data spatial. This process involves using geographic coordinates (longitude and latitude) to place points on a map. In some cases, you won’t have coordinates but street addresses. Here, you’ll need to geocode your data, which involves converting street addresses to geographic coordinates. These tasks are intimately related to the concept of projection and reprojection, and underlying all of these concepts is the Coordinate Reference System.

# Longitude/Latitude
