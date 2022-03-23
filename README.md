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
Best case scenario is that you have a point data set with geographic coordinates. Geographic coordinates are in the form of a longitude and latitude, where longitude is your X coordinate and spans East/West and latitude is your Y coordinate and spans North/South.

Let’s bring in a csv data set of homeless encampments in Los Angeles City, which was downloaded from the <a href="https://data.lacity.org/">Los Angeles City Open Data portal</a>. I uploaded the data set on GitHub so you can directly read it in using read_csv()

```R
homeless311.df <- read_csv("https://raw.githubusercontent.com/crd230/data/master/homeless311_la_2019.csv")
```

The data represent homeless encampment locations in 2019 as reported through the City’s 311 system. 

Viewing the file and checking its class you’ll find that homeless311.df is a regular tibble, not a spatial <b>sf</b> points object.

We will use the function ```st_as_sf()``` to create a point <b>sf</b> object of <i>homeless311.df</i>. The function requires you to specify the longitude and latitude of each point using the ```coords =``` argument, which are conveniently stored in the variables <i>Longitude</i> and <i>Latitude</i>.

```R
homeless311.sf <- st_as_sf(homeless311.df, coords = c("Longitude", "Latitude"))
```

# Street Addresses
Often you will get point data that won’t have longitude/X and latitude/Y coordinates but instead have street addresses. The process of going from address to X/Y coordinates is known as geocoding.  

To demonstrate geocoding, type in your street address, city and state inside the quotes below.

```R
myaddress.df  <- tibble(street = "", city = "", state = "")
```
This creates a tibble with your street, city and state saved in three variables. To geocode addresses to longitude and latitude, use the function ```geocode()``` which is a part of the <b>tidygeocoder</b> package. Use ```geocode()``` as follows

```R
myaddress.df <- geocode(myaddress.df, street = street, city = city, state = state, method = "osm")
```

Here, we specify street, city and state variables. The argument ```method = 'osm' ``` specifies the geocoder used to map addresses to longitude/latitude locations, in the above case ``` 'osm' ``` stands for <a href="https://www.openstreetmap.org">OpenStreetMaps</a>. Think of R going to the OpenStreetMaps website, searching for each address, plucking the latitude and longitude of your address, and saving it in a tibble named <i>myaddress.df</i>

If you view this object, you’ll find the latitude <i>lat</i> and longitude <i>long</i> attached as columns. Convert this point to an <b>sf</b> object using the function ```st_as_sf()```.

```R
myaddress.sf <- st_as_sf(myaddress.df, coords = c("long", "lat"))
```

Type in ```tmap_mode("view")``` and then map <i>myaddress.sf</i> (Hint: Refer to what we did in class/lab for referesher). Zoom into the point. Did it get your home address correct?

```R
shelters.df <- read_csv("https://raw.githubusercontent.com/crd230/data/master/Homeless_Shelters_and_Services.csv")

glimpse(shelters.df)
```

```{r}
## Rows: 182
## Columns: 23
## $ source       <chr> "211", "211", "211", "211", "211", "211", "211", "211", …
## $ ext_id       <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
## $ cat1         <chr> "Social Services", "Social Services", "Social Services",…
## $ cat2         <chr> "Homeless Shelters and Services", "Homeless Shelters and…
## $ cat3         <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
## $ org_name     <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, "www.catalystfdn.org…
## $ Name         <chr> "Special Service For Groups -  Project 180", "1736 Famil…
## $ addrln1      <chr> "420 S. San Pedro", "2116 Arlington Ave", "1736 Monterey…
## $ addrln2      <chr> NA, "Suite 200", NA, NA, NA, NA, "4th Fl.", NA, NA, NA, …
## $ city         <chr> "Los Angeles", "Los Angeles", "Hermosa Beach", "Monrovia…
## $ state        <chr> "CA", "CA", "CA", "CA", "CA", "CA", "CA", "CA", "CA", "C…
## $ hours        <chr> "SITE HOURS:  Monday through Friday, 8:30am to 4:30pm.",…
## $ email        <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
## $ url          <chr> "ssgmain.org/", "www.1736fcc.org", "www.1736fcc.org", "w…
## $ info1        <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
## $ info2        <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
## $ post_id      <dbl> 772, 786, 788, 794, 795, 946, 947, 1073, 1133, 1283, 135…
## $ description  <chr> "The agency provides advocacy, child care, HIV/AIDS serv…
## $ zip          <dbl> 90013, 90018, 90254, 91016, 91776, 90028, 90027, 90019, …
## $ link         <chr> "http://egis3.lacounty.gov/lms/?p=772", "http://egis3.la…
## $ use_type     <chr> "publish", "publish", "publish", "publish", "publish", "…
## $ date_updated <chr> "2017/10/30 14:43:13+00", "2017/10/06 16:28:29+00", "201…
## $ dis_status   <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
```

The file contains no latitude and longitude data, so we need to convert the street addresses contained in the variables <i>addrln1</i>, <i>city</i> and <i>state</i>. Use the function ```geocode()```. The process will take a few minutes so be patient.

```R
shelters.geo <- geocode(shelters.df, street = addrln1, city = city, state = state, method = 'osm')
```

Look at the column names.


```R
names(shelters.geo)
```

```{r}
##  [1] "source"       "ext_id"       "cat1"         "cat2"         "cat3"        
##  [6] "org_name"     "Name"         "addrln1"      "addrln2"      "city"        
## [11] "state"        "hours"        "email"        "url"          "info1"       
## [16] "info2"        "post_id"      "description"  "zip"          "link"        
## [21] "use_type"     "date_updated" "dis_status"   "lat"          "long"
```

We see the latitudes and longitudes are attached to the variables <i>lat</i> and <i>long</i>, respectively. Notice that not all the addresses were successfully geocoded.

```R
summary(shelters.geo$lat)
```

```{r}
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max.    NA's 
##   33.74   33.98   34.05   34.06   34.10   34.70       8
```

Eight shelters received an ```NA```. This is likely because the addresses are not correct, has errors, or are not fully specified. For example, the address <i>11046 Vly</i> Mall should be written out as <i>11046 Valley Mall</i>. You’ll have to manually fix these issues, which becomes time consuming if you have a really large data set. For the purposes of this lab, let’s just discard these, but in practice, make sure to double check your address data (See the document Geocoding_Best_Practices.pdf in the Other Resources folder on Canvas for best practices for cleaning address data).

```R
shelters.geo <- shelters.geo %>%
                filter(is.na(lat) == FALSE & is.na(long) == FALSE)
```

Convert latitude and longitude data into spatial points using the function ```st_as_sf()```.

```R
shelters.sf <- st_as_sf(shelters.geo, coords = c("long", "lat"))
```

