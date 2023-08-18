# Route Plotting from `.gpx` and `.fit` Files in R

<br>

## Introduction

Smart watches, fitness trackers and various other devices are becoming more and more popular 
for people who want to keep track of their leisure activities such as running, cycling, swimming 
etc.  In most cases these devices have some global positioning functionality that allows users 
to retain a record of where they've been, or where they want to go.
<br>
A lot of these spatial details are stored by default as either `.gpx` or `.fit` files.  So what are they?  
<br>
`.gpx` files usually contain little more that spatial elements and are normally created in advance 
of an activity, providing a route map for a user to follow.  `.fit` files usually also contain similar 
spatial data, but also include additional parameters such as speed, distance, heart rate, step count, 
calories, respiration rate etc., and are more commonly used to analyse the performance aspects of a user's 
activity.
<br>
From a more technical point of view;

## `.gpx` Files

A `.gpx` or GPS eXchange Format file is a common open format XML (extensible markup language) file containing 
geospatial data relating to "waypoints", "tracks" and "routes".

This might seem complicated, but in fact is just a sequence of latitudes and longitudes (often with elevation, 
time, and other data fields) that collectively make a route that a GPS enabled device can follow.  This 
sequence of positions can be overlaid onto a map to provide additional context.

As a minimum, `.gpx` files will always contain latitude and longitude. All other elements are optional.

<br>

## `.fit` Files

`.fit` (Flexible and Interoperable Data Transfer) files were developed by Garmin Ltd. specifically for 
storing data resulting from sporting activities.  A `.fit` file can contain almost 100 different variables 
and the exact file contents can vary depending on the device that created it.  `.fit` files often contain 
data not only relating to GPS position (latitude/longitude/elevation) but also other data often recorded 
from additional sensors such as heart rate monitors, power monitors, step counters etc.  

An added complication of this is that each of these different physical data sources (heart rate monitor, 
step counter, GPS unit etc.) all send their data to the same `.fit` file, with each sensor having it's own 
specific list of measurement values within the file.  But each of them can be activated at slightly different 
times, resulting in one `.fit` file containing multiple lists of data with slightly different start and/or 
stop times!  So, a potentially rich source, but one which might need some careful handling.

<br>

## Handling `.gpx` Files in R with `plotKML()`

Using the `plotKML()` package to interpret a `.gpx` file in R.

```{r, warning = FALSE, message = FALSE}
library(tidyverse)
library(plotKML)
library(leaflet)
library(here)
```


To start, read in the data using `the readGPS()` function from `plotKML`.

```{r}
gpx <- readGPX(here("data/COURSE_149408654.gpx"))
```


The `.gpx` file structure is set up as a series of lists, namely;

* metadata
* bounds
* waypoints
* tracks
* routes

In this example there's only track data, so extract that, save it as a tibble and see what it looks like.

```{r}
track <- as_tibble(gpx$tracks[[1]][[1]])

head(track)
```
![](images/gpx_tracks.png)        

So the data contains latitude and longitude, elevation and time observations.  The time column corresponds 
with the time the `.gpx` track was created (maybe using a mapping website) and doesn't reflect the time an 
activity was carried out.  

Note that the elevation column is stored as a character and should be changed into a numeric before using it 
in any analysis;

```{r}
track_gpx <- track %>% 
  mutate(ele = as.numeric(ele))
```

I should now be able to overlay this onto a map using the `addPolyLines()` function in `leaflet()`.    At this 
stage I don't know what the maximum and minimum latitude and longitude should be, but I can use `min()` and `max()` 
to determine this when I make the plot.

```{r}
track_gpx %>% 
leaflet() %>%
  fitBounds(lng1 = min(track_gpx$lon), lat1 = min(track_gpx$lat),
            lng2 = max(track_gpx$lon), lat2 = max(track_gpx$lat)) %>%
  addTiles() %>%  
  addPolylines(lng = ~lon,
               lat = ~lat)

```

![](images/codeclan_britannia_track.png)        

<br>

## Overlaying Elevation Data from `.gpx` as a Colour Gradient

The `.gpx` file also contains information about the route elevation and it would be nice to have an indication of 
elevation on the track as a colour gradient.

`leaflet()` maps are generated using javascript under the hood, so adding additional layers of information to the
plotted track is not necessarily straightforward within the confines of R.  However, there are packages available 
to help.  I'm going to use a javascript plug-in called `leaflet.hotline`; there's more information about the 
plug-in [here](https://github.com/iosphere/Leaflet.hotline).

For the plugin to work successfully I'll also need the following libraries;

```{r}
library(htmltools)
library(htmlwidgets)
```

The first to do is to download the javascript plug-in to the project environment.

```{r}
# Download the plug-in to the current project folder.

download.file(
  # This is where the plug-in comes from
  "https://raw.githubusercontent.com/iosphere/Leaflet.hotline/master/dist/leaflet.hotline.js",  
  # this is where the plug-in details can be found (relative to the project top level)
  "leaflet.hotline.js", 
  # and I'm using a binary version
  mode="wb") 
```

The plug-in is downloaded, I need to load it into our environment and register it;

```{r}
#load the plugin
hotlinePlugin <- htmltools::htmlDependency(
  name = 'Leaflet.hotline',
  version = "0.4.0",
  # file path in the next line should be the directory containing this .Rmd file
    src = c(file = normalizePath(here())), 
  script = "leaflet.hotline.js"
  )
#register plugin
registerPlugin <- function( map, plugin ) {
  map$dependencies <- c( map$dependencies, list( plugin ) )
  map
}
```
And now it's ready to use in the `leaflet()` plot.  

```{r}
leaflet() %>% 
  addTiles() %>% 
  fitBounds(
    lng1 = min(track_gpx$lon), lat1 = min(track_gpx$lat),
    lng2 = max(track_gpx$lon), lat2 = max(track_gpx$lat)) %>% 
  registerPlugin(hotlinePlugin) %>%
  onRender("function(el, x, data) {
    data = HTMLWidgets.dataframeToD3(data);
    data = data.map(function(val) { return [val.lat, val.lon, val.ele]; });
    L.hotline(data, {renderer: L.Hotline.renderer(), min: 0, max: 100}).addTo(this); }", 
    data = track_gpx)
```

![](images/codeclan_britannia_track_elev.png)        


In addition to the previous code I've added `registerPlugin()` for the javascript plug-in that I want to use
and then added a small block of javascript that we want to run when the map is rendered using `onRender()`.

If all you want is a colour-graded elevation overlay on your map then this is fine and you don't need to know 
any more javascript.  Suffice to say that the values in `{min: __, max: __}` relate to the minimum and maximum 
elevation values that you want to include in the colour grading layer.  High elevations will be red, low 
elevations will be green.

# Handling `.fit` Files in R with `FITfileR()`

Although `.fit` files are an open format data type, there are often some intricacies specific to different 
device manufacturers that mean that not all `.fit` files can be processed in *exactly* the same way.  A brief 
overview of some of these differences can be found [here](https://logiqx.github.io/gps-wizard/fit.html).

For the purposes of this example I'm going to look at data collected on a Garmin device (the `.fit` file 
format was developed by Garmin, so if anyone's doing things properly it should be them!).

`FITfileR` is currently only available from github and can be downloaded as follows:

```{r, eval=FALSE}
if (!require(remotes)) install.packages("remotes")
remotes::install_github("grimbough/FITfileR")
```

**Note:** `FITfileR()` needs the most up to date version of `rlang()` to install successfully

Once installed, call the library:
```{r}
library(tidyverse)
library(FITfileR) # note: needs current version of 'rlang' to install successfully
```

The first thing to do is to read the data using `readFitFile()` from `FITfileR()`.  

**Note:** Be aware that `readFitFile()` needs *forward slash* directory separators - be 
careful if you copy file path names straight from windows (which use backslash)!

```{r}
activity <- readFitFile(here("data/718883059.fit"))
```

If I try to view the activity data directly then I only get a fraction of the total information:

```{r}
activity
```

![](images/activity_fit_file.png)        


The `.fit` file specification allows for almost 100 different types of data "message".  To view all of the 
message types stored within a specific file that are available for analysis you can use the function 
`listMessageTypes()`.

```{r}
listMessageTypes(activity)
```

![](images/activity_messages.png)        

Usually working with `.fit` files we are going to be interested in certain types of data, typically 
location, speed, elevation, etc. This type of data is classed as "records" and can be accessed using the 
`records()` function.

```{r}
activity_records <- records(activity)

activity_records
```

![](images/activity_records.png)        


You will often see that `activity_records` contains a list of several tibbles.  This happens when the file 
contains multiple distinct definitions of the activity record. This happens when multiple data recording 
devices send data simultaneously to the `.fit` file (e.g. GPS + heart rate monitor + step counter) and 
each start acquiring data at different times.

If you look at each tibble individually you will see that all of the column headers are the same, which 
means that you can often merge all of the messages together into a single tibble using `bind_rows()` and 
then arrange the resulting data by timestamp to make sure that it is in chronological order.  Any missing 
values will be filled with NA. 

```{r}
all_activity_records <- records(activity) %>% 
  bind_rows() %>% 
  arrange(timestamp) 

all_activity_records
```

![](images/all_activity_records.png)        


This looks very much like the dataset I got from my `.gpx` file to plot in leaflet, and I can use 
this data in the same way.  For a simple track plot I can use the track data from `all_activity_records` 
as follows;

```{r}
# note that the .fit file refers to 'altitude' rather than 'elevation'
track_fit <- all_activity_records %>%
  rename(ele = altitude,
         lat = position_lat,
         lon = position_long) %>% 
  drop_na() # drop na for plotting to render successfully

```

And then plot the activity using `leaflet()`.

```{r}
track_fit %>% 
  leaflet() %>% 
  fitBounds(lng1 = min(track_fit$lon), lat1 = min(track_fit$lat),
            lng2 = max(track_fit$lon), lat2 = max(track_fit$lat)) %>% 
  addTiles() %>% 
  addPolylines(lng = ~lon,
               lat = ~lat)
 
```

![](images/track_fit.png)        

<br>


## Overlaying Additional Data Layers from `.fit`

<br>
It's also possible to use the inherent versatility of `leaflet()` to add more information to this, such 
as start and stop points, elevation data using `leaflet.hotline` and additional markers as required, e.g. 
we already have `leaflet.hotline` installed and registered from when we added layers to our `.gpx` file 
plot, so there's no need to redo that for this example.  So let's add the following;

* Start point marker
* End point marker
* Marker to show where the user's maximum speed was achieved

<br>

```{r}
# Determine start and end points;
start_lat <- first(track_fit$lat)
start_lon <- first(track_fit$lon)

end_lat <- last(track_fit$lat)
end_lon <- last(track_fit$lon)

# Determine the point at which max speed was reached
speed_peak <- track_fit %>% 
  filter(speed == max(speed))

speed_label <- paste("Maximum Speed: ", speed_peak$speed)

# And plot:
leaflet() %>% 
  addProviderTiles(providers$Esri.WorldImagery) %>% 
  fitBounds(lng1 = min(track_fit$lon), lat1 = min(track_fit$lat),
            lng2 = max(track_fit$lon), lat2 = max(track_fit$lat)) %>% 
  addMarkers(lng = speed_peak$lon, 
                   lat = speed_peak$lat,
                   label = speed_label) %>% 
  addCircleMarkers(lng = start_lon,
                   lat = start_lat,
                   label = "Start",
                   radius = 4,
                   col = "slateblue",
                   opacity = 1) %>% 
  addCircleMarkers(lng = end_lon,
                   lat = end_lat,
                   label = "End",
                   radius = 4,
                   col = "red",
                   opacity = 1) %>%
  registerPlugin(hotlinePlugin) %>%
  onRender(data = track_fit, "function(el, x, data) {
    data = HTMLWidgets.dataframeToD3(data);
    data = data.map(function(val) { return [val.lat, val.lon, val.ele]; });
    L.hotline(data, {renderer: L.Hotline.renderer(), min: 100, max: 400}).addTo(this); }")


```

![](images/track_fit_overlay.png)        

