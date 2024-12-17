---
layout: post
author: simon

title: Silhouette Generation for Minimizing Scanning Variation
caption: Search algorithms for synthesizing a light-adjusting filter shape
---

*This is part of a series exploring the design of a novel scanning photo-exposure technique for screen printing and other light-sensitive media (cyanotypes, platinum and palladium prints, photo-chemical etching, etc). This exposure technique scans a light strip across the screen and stencil, using focusing lenses to achieve the excellent exposure characteristics of a room-sized projection lamp in the compact space of a typical light table.*

*[Previous work first explained projection issues](https://simonsbench.net/flatbed-exposure-sketch) and sketched out the idea, and a [later project](https://simonsbench.net/light-shapes-and-lens-spacings) demonstrated that exposure variation could be minimized sufficiently for many processes through measurement and simulation of different lenses.*

A follow-on project for prototyping a scanning photo-exposure unit is exploring generating silhouettes. These are flat shapes placed in profile top of an LED and its lens, and selectively trim specific areas of the light shape. Ideally, this minimizes the cumulative exposure variation across the strip.

The 3-5% variation achieved through careful choice of spacing alone is excellent for many processes using an emulsion layer; these have variation tolerances upwards of 20%. But this level of variation does not work for more sensitive processes, like platinum print photography and cyanotypes; the artifact of the lights combining, in the form of a big-hump, little-hump pattern, will image. These processes need less than 1% variation.

This post examines a search algorithm for synthesizing the shape of this filter, finding that a greedy algorithm and an heuristic that maintains edge smoothness (and consequently manufacturability) can synthesize a mask that lower the variation to under 0.5%.

## State
The measurement data used to find ideal spacings is reused; this is a 2D matrix of intensity values. This experiment introduces another 2D matrix of the same size representing the silhouette mask; the mask uses 0/1 values to represent selecting (1) or removing (0) the light from each square cell in the measurement grid. The intensity matrix multiplied by the mask matrix produces a new matrix that simulates the light, lens, and mask combined. The same spacing and addition code translates this multiplied matrix into a simulation of the entire strip.

## Problem Structure and Difficulty of a Directly Computed Solution
In a search, any decision to include an area in a filtered or masked-out area has computational consequences that cascade across the strip. This makes a closed-form or direct solution difficult.

A cell that is blocked out the filter reduces the exposure level of its column as the strip scans. This changes the filter characteristics of two other groups of columns.

First, whenever a cell in column *i* is filtered, several others columns are in a balance group. Several columns contribute light to this spot in a scan, and these can be filtered less; in the cumulative exposure, the error of this column is reduced by the cell being filtered. Given a spacing period *p*, these balance columns are defined by the cyclic group *i + |p|*.

Second, the columns in a symmetric group with the filtered column *i* are also impacted. The same filter design is (ideally) placed over every light, so a blocked cell will show up in the same place on each one.

The balance and symmetry groups cascade; starting from a column *i*, there is a balance group that contributes to the same spot and a symmetric group that mirrors this spot on the filter. But the balance group also has a symmetry group, and the symmetry group also has a balance group. And so on.

The cycle will converge, but it is not useful to try and solve for these groups directly; if the spacing has a period of 1, then the columns impacted by any change to the filter are... every column!

## An Initial Heuristic (That Didn't Work)
An initial search used a greedy search, with a step heuristic that picked the column with the highest error and masked out cells in that column until the error was gone. The cumulative exposure graph looked wonderfully flat; computationally, the filter worked very well, with an error of 0.1%.

But the filter shape was impractical. Some parts of the mask featured pillars that stuck out far over the lens, with jagged, discontinuous edges as the mask jogged over to a neighboring cell with a short filter.

Some effort was put into a human-assisted search; with the cumulative exposure and an individual light profile graphed simultaneously, it is easy to see where and why the exposure becomes uneven.

## A Smoothness Heuristic
A heuristic that did work is one that ensures the mask edge is relatively smooth. A slope-delta calculation compares the slope between pairs of cells at the forefront of the filter. Given a column *i*, the filter fronts for *i-1* and *i* form the left slope, and the fronts for *i* and *i* form the right slope. The slope delta is the left slope minus the right slope. Cells at either end mirror their inner neighbor to the outside.

The heuristic used in the search has two conditions:
* the column must have some error
* the column's heuristic value is slope delta, plus 2

Cells that are notched into the filter edge are prioritized for being filled in; cells that are smooth have a slope delta of 2; cells that are a spikey away from the filter edge are deprioritized. This enforces smoothness of the filter shape to make it manufacturable, while allowing some local conformance to error peaks.

This heuristic reminds me of surface tension; the edge creeps outwards, until it hits the error limit, and stops.

With this heuristic, the variation in the cumulative exposure output was between 0.2 and 0.3%.

One additional detail is that the search worked better when the trim value was lowered past the minimum exposure level in the unfiltered strip. This turned out to be because these minimum points prevented the mask from reaching deep enough into some other areas; they were outside the achievable slope rate of the filter.
