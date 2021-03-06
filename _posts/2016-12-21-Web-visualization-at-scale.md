---
layout: post
title: Web Visualization at Scale
author: Egil Moeller
tags: [Data, Visualization]
category: [Code]
comments: False
redirect_to: http://globalfishingwatch.org/data-blog/web-visualization-at-scale/
---

# Background

One of the main outputs of the Global Fishing Watch project is the interactive [map of global fishing](http://globalfishingwatch.org/map). This map presented some particular challenges to implement. This blog post is about these challenges and our solutions to them.

Vessel tracks are derived from time series data, that is, ordered points, each of which each have a latitude, longitude and timestamp. The data is generally sparse, often with [long gaps due to satellite coverage]({{ site.url }}{{site.baseurl}}//vessel%20tracks/2016/11/01/AIS-Data-Example.html), meaning that for most positions along the track, most of the time, there is no vessel.

There are two main approaches to rendering map data on the web in use today: Server side rasterization together with tiling, and client side rendering using javascript and either dom / SVG dom objects or the 2D canvas.

The advantage of the javascript approach is that data can be manipulated, styled, and interacted with client side. However, it comes at a performance price, and rendering larger datasets this way is not possible.

Tiled server side rasterization on the other hand does not have any size limitations beyond your server farms capacity. And even that  is not a hard limit as you can either pre-generate all tiles, spending whatever time necessary to do so, or generate tiles and cache them on the fly depending on your specific data and usage patterns.

Neither of these two approaches, however, fit our data. The data size is too big for client side javascript rendering, and rasterizing in three dimensions (lat, lon, time) generates too big tiles that are too large. There has been some preliminary work by others to solve this using video compression, but such a solution would limit some of the interactivity and our ability to style the visualization. 


# Sketching Our Solution

We chose to do a hybrid approach -- rendering client side, but using tiling. That is, our tiles contain vector data. At lower zoom levels the tiles can not contain individual track points, but rather clusters of points. We used two different clustering methods: First we cluster in time inside each track, resampling the tracks to lower precision in time. If that is not enough to get the tile content below a predefined size limit, we then cluster spatially for each time point, across vessels.

Unfortunately, given the number of tracks we had (150,000 vessels) and the time window of our visualization (multiple years) this approach was not enough to reduce the data size below what the javascript renderers can handle without degrading the user experience severely. To visualize tracks so that a user can manually detect fishing, you need to draw several individual points per hour.

To get around this, we split the tiles temporally, generating a (very flat, see below) tile cube. In addition, we switched to rendering the vector data using WebGL rather than javascript.

# WebGL Rendering

WebGL is intended to do 3D visualizations. However, it is not a 3D API -- it deals with 2D rendering of 3D objects by user supplied shader code, providing primitives to do coordinate transformation, projection, Z-ordering, etc. The shader code runs directly on the graphics card. They are written in a statically typed compiled language and run in parallel on multiple data. All this makes WebGL vastly faster than Javascript at processing large number of points -- somewhere between 1000 and 10000 times faster depending on exactly how you visualize the points. It mostly depends on how large each point is in pixels.

Together with WebGL, modern browsers also support a very basic API for handling large arrays of typed data. Given the correct structuring of the tile files this can be leveraged to load data from tiles directly into WebGL without javascript code parsing or manipulating each individual row of data. We developed a [self-documenting format](https://github.com/GlobalFishingWatch/pelagos-client/blob/master/js/app/Data/TypedMatrixParser.js) consisting of a JSON encoded header specifying what columns a tile contains together with column and tile metadata, followed by each column as a packed array of IEEE 32bit floats.

This approach does require that all processing be done in WebGL. This includes both styling, mapping from tile data to visualization parameters, and point filtering (e.g. handling the currently selected time window or user defined filters like vessel flag states or gear type).

## Mouse Interactivity

We used a variant of a technique called [WebGL picking](http://www.opengl-tutorial.org/miscellaneous/clicking-on-objects/picking-with-an-opengl-hack/) to handle mouse interactivity. In this technique, objects are rendered in an offscreen buffer using a unique color for each object, so that the object under the mouse pointer can be deduced by reading back the pixel color. Our variant doesn't render the whole scene, but rather only a 100x100 pixel square around the mouse pointer (using WebGL clipping), caching the results between mouse moves unless the visualization parameters have changed. This minimizes the sum of the draw time and the time to read the pixel values back from the graphics card.

 # Temporal Tiling

A spatial tiling system consists of a pyramid of tile grids at all available zoom levels. A full temporal tiling system using the same approach would require a very large set of tiles. The usage patterns of such a tile cube would however be very sparse because most tiles would only be requested by a very small number of users.

Our [temporal tiling system](http://globalfishingwatch.github.io/pelagos-client/tiling.html):

* Loads more sparse data / clustered data when
  * a large temporal window is selected
  * a large spatial window is selected
* Loads more dense data / raw data when
  * a small temporal window is selected
  * a small spatial window is selected
* Does not have to produce all combinations of all time ranges and all spatial extents


We used a normal spatial tiling with a pyramid of tiles in sizes ranging from the whole world to very small regions, but only one temporal slice / tile length - one month. We used a dynamic spatial zoom level to emulate the effect of a full temporal pyramid for each spatial zoom level. The zoom level for the spatial tiles depend both on actual spatial zoom and the temporal zoom: The number of spatial tiles to divide the screen into is divided by the number of temporal slices needed to cover the cuorrent time window.

If you have questions about how we made this visualization, or suggestions for how we might improve the technical “under the hood” aspects of the visualization, let us know in the comments below. You can also send an email to egil@skytruth.org
