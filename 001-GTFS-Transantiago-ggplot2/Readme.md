


Days ago a study says that Santiago, city where I live, has one of the best public transport system in LATAM (WAT?! define *best* please!). So I've search for some information and I found [this](http://www.siemens.com/press/pool/de/feature/2014/infrastructure-cities/2014-06-mobility-opportunity/slide-credo.pdf#page=6). Anyway I tried to find some related data/information to work/play and I found the *Transantiago GTFS*. GTFS means *General Transit Feed Specification* and is a format for public transportation schedules and geographic data.

This information comes in a zip file with information about routes, stations (name, location), shapes (route, path) and other elements in the system. For example the `shape.txt` file have the geographic path of each route.


```r
shapes <- read.csv("data/shapes.txt")
```

|shape_id   | shape_pt_lat| shape_pt_lon| shape_pt_sequence|
|:----------|------------:|------------:|-----------------:|
|225-I-BASE |       -33.39|       -70.55|                 0|
|225-I-BASE |       -33.39|       -70.55|                 1|
|225-I-BASE |       -33.39|       -70.54|                 2|
|225-I-BASE |       -33.39|       -70.54|                 3|
|225-I-BASE |       -33.39|       -70.54|                 4|
|225-I-BASE |       -33.39|       -70.54|                 5|

It's simple plot this data with ggplot.


```r
library(ggplot2)

p <- ggplot(shapes) +
  geom_path(aes(shape_pt_lon, shape_pt_lat, group = shape_id), size = .2, alpha = .1) +
  coord_equal()
p
```

<img src="plot/gtfs-plot_simple-1.png" title="" alt="" style="display: block; margin: auto;" />

Is a good plot with a one line of code to start. But let's get the things more fun. Transantiago have a subway called *Metro*, so let's plot with more detail showing the stations and the routes (lines) over this plot.

First, we need the `devtools` package for load a script from an url. This file contains a null_theme for `ggplot2`:


```r
library(devtools)
library(plyr)
library(dplyr)
library(ggthemes)
```

We need obtain the stops and routes which belong to *Metro*. In this case, the *stop_id* don't contain a number so we filter the metro's stations with `!grepl("\\d", stop_id)`. Then we need filter the shapes and routes for the *metro*. At the beggining is a bit complicated, in fact I needed some time to see the association between all this tables.


```r
routes <- read.csv("data/routes.txt")
trips <- read.csv("data/trips.txt")
stops <- read.csv("data/stops.txt")

stops_metro <- stops %>% filter(!grepl("\\d", stop_id))

routes_metro <- routes %>% filter(grepl("^L\\d", route_id))

shapes_metro <- shapes %>% filter(shape_id %in% trips$shape_id[trips$route_id %in% routes_metro$route_id]) %>%
   arrange(shape_id, shape_pt_sequence)
```

Now, get the color for each Metro line.


```r
shapes_colors <- left_join(left_join(shapes %>% select(shape_id) %>% unique(),
                                     trips %>% select(shape_id, route_id) %>% unique(),
                                     by = "shape_id"),
                           routes %>% select(route_id, route_color) %>% unique(),
                           by = "route_id") %>%
  mutate(route_color = paste0("#", route_color))
```

```
## Warning in left_join_impl(x, y, by$x, by$y): joining factors with
## different levels, coercing to character vector
```

```r
shapes_colors_metro <- shapes_colors %>%
  filter(shape_id %in% trips$shape_id[trips$route_id %in% routes_metro$route_id]) %>% unique() %>%
  arrange(shape_id)
```

The data is ready. So it's time to make another plot.


```r
p2 <- ggplot() +
  geom_path(data=shapes, aes(shape_pt_lon, shape_pt_lat, group=shape_id), color="white", size=.2, alpha=.05) +
  geom_path(data=shapes_metro, aes(shape_pt_lon, shape_pt_lat, group=shape_id, colour=shape_id), size = 2, alpha=.7) +
  scale_color_manual(values=shapes_colors_metro$route_color) +
  geom_point(data=stops_metro, aes(stop_lon, stop_lat), shape=21, colour="white", alpha =.8) +
  coord_equal() +
  theme_null() +
  theme(plot.background = element_rect(fill = "black", colour = "black"),
        title = element_text(hjust=1, colour="white", size = 8),
        axis.title.x = element_text(hjust=0, colour="white", size = 7)) +
  xlab(sprintf("Joshua Kunst | Jkunst.com %s",format(Sys.Date(), "%Y"))) +
  ggtitle("TRANSANTIAGO\nSantiago's public transport system")
p2
```

<img src="plot/gtfs-plot_complex.png" title="plot of chunk plot_complex" alt="plot of chunk plot_complex" style="display: block; margin: auto;" />

Or we can just plot only te metro routes with the follow code:


```r
p3 <- ggplot() +
  geom_path(data=shapes_metro, aes(shape_pt_lon, shape_pt_lat, group=shape_id, colour=shape_id), size = 2, alpha=.8) +
  scale_color_manual(values=shapes_colors_metro$route_color) +
  geom_point(data=stops_metro, aes(stop_lon, stop_lat), shape=21, colour="white", alpha =.8) +
  coord_equal() +
  theme_null() +
  theme(plot.background = element_rect(fill = "black", colour = "black"),
        title = element_text(hjust=1, colour="white", size = 8),
        axis.title.x = element_text(hjust=0, colour="white", size = 7)) +
  xlab(sprintf("Joshua Kunst | Jkunst.com %s",format(Sys.Date(), "%Y"))) +
  ggtitle("Santiago's METRO") 
p3
```

<img src="plot/gtfs-plot_complex_2.png" title="plot of chunk plot_complex_2" alt="plot of chunk plot_complex_2" style="display: block; margin: auto;" />

You can see the original image on wikipedia [here](http://upload.wikimedia.org/wikipedia/commons/archive/4/49/20091229144454%21Metro_de_Santiago.svg).

Finally we can also export the plota in a pdf file like [this](https://github.com/jbkunst/r-posts/blob/master/GTFS-Transantiago-ggplot2/plot/gtfs-metro.pdf?raw=true) or [this](https://github.com/jbkunst/r-posts/blob/master/GTFS-Transantiago-ggplot2/plot/gtfs-transantiago.pdf?raw=true) with:


```r
ggsave(filename = "plot/gtfs-transantiago.pdf", plot = p2, bg = "black")
ggsave(filename = "plot/gtfs-metro.pdf", plot = p3, bg = "black")
```

As you can see, it's simply make a good graphic with a few lines of code. And better, *GTFS* is a *standard*, so you can reuse a big part of this code (and make it a better code!) to plot transport systems from other cities. If you do it, let me know.
