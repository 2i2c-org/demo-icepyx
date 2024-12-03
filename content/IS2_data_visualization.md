---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.4
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Visualizing ICESat-2 Elevations

This notebook ({download}`download <IS2_data_visualization.ipynb>`) demonstrates interactive ICESat-2 elevation visualization by requesting data from [OpenAltimetry](https://www.openaltimetry.org/) based on metadata provided by [icepyx](https://icepyx.readthedocs.io/en/latest/). We will show how to plot spatial extent and elevation interactively.

> ⚠️ **Some of this notebook is currently non-functional**
>
> Visualizations requiring the
> [OpenAltimetry API](https://openaltimetry.earthdatacloud.nasa.gov/data/openapi/swagger-ui/index.html)
> are currently unavailable (since ~October 2023).
> The API changed and we haven't yet updated this notebook correspondingly.

+++

Import packages

```{code-cell} ipython3
import icepyx as ipx
```

## Create an ICESat-2 query object

Set the desired parameters for your data visualization.

For details on minimum required inputs, please refer to [IS2_data_access](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access.html). If you are using a spatial extent input other than a bounding box for your search, it will automatically be converted to a bounding box for the purposes of visualization ONLY (your query object will not be affected).

```{code-cell} ipython3
#bounding box
#Larsen C Ice Shelf
short_name = 'ATL06'
date_range = ['2020-7-1', '2020-8-1']
spatial_extent = [-67, -70, -59, -65] 
cycles = ['03']
tracks = ['0948', '0872', '1184', '0186', '1123', '1009', '0445', '0369']
```

```{code-cell} ipython3
# # polygon vertices
# short_name = 'ATL06'
# date_range = ['2019-02-20','2019-02-28']
# spatial_extent = [(-55, 68), (-55, 71), (-48, 71), (-48, 68), (-55, 68)]
```

```{code-cell} ipython3
# # polygon geospatial file
# short_name = 'ATL06'
# date_range = ['2019-10-01','2019-10-05']
# spatial_extent = './supporting_files/data-access_PineIsland/glims_polygons.shp'
```

```{code-cell} ipython3
region = ipx.Query(short_name, spatial_extent, date_range)
```

```{code-cell} ipython3
print(region.product)
print(region.dates)
print(region.start_time)
print(region.end_time)
print(region.product_version)
print(list(set(region.avail_granules(cycles=True)[0]))) #region.cycles
print(list(set(region.avail_granules(tracks=True)[0]))) #region.tracks
```

## Visualize spatial extent 
By calling function `visualize_spatial_extent`, it will plot the spatial extent in red outline overlaid on a basemap, try zoom-in/zoom-out to see where is your interested region and what the geographic features look like in this region.

```{code-cell} ipython3
region.visualize_spatial_extent()
```

## Visualize ICESat-2 elevation using OpenAltimetry API

**Note: this function currently only supports products `ATL06, ATL07, ATL08, ATL10, ATL12, ATL13`**

Now that we have produced an interactive map showing the spatial extent of ICESat-2 data to be requested from NSIDC using icepyx, what if we want to have a quick check on the ICESat-2 elevations we plan to download from NSIDC? [OpenAltimetry API](https://openaltimetry.org/data/swagger-ui/#/) provides a nice way to achieve this. By sending metadata (product, date, bounding box, trackId) of each ICESat-2 file to the API, it can return elevation data almost instantaneously. The major drawback is requests are limited to 5x5 degree spatial bounding box selection for most of the ICESat-2 L3A products [ATL06, ATL07, ATL08, ATL10, ATL12, ATL13](https://icesat-2.gsfc.nasa.gov/science/data-products). To solve this issue, if you input spatial extent exceeds the 5 degree maximum in either horizontal dimension, your input spatial extent will be split into 5x5 degree lat/lon grids first, use icepyx to query the metadata of ICESat-2 files located in each grid, and send each request to OpenAltimetry. Data sampling rates are 1/50 for ATL06 and 1/20 for other products.

There are multiple ways to access icepyx's visualization module. This option assumes you are visualizing the data as part of a workflow that will result in a data download. Alternative options for accessing the OpenAltimetry-based visualization module directly are provided at the end of this example.

```{code-cell} ipython3
cyclemap, rgtmap = region.visualize_elevation()
cyclemap
```

#### Plot elevation for individual RGT

The visualization tool also provides the option to view elevation data by latitude for each ground track.

```{code-cell} ipython3
rgtmap
```

### Move on to data downloading from NSIDC if these are the products of interest

For more details on the data ordering and downloading process, see [ICESat-2_DAAC_DataAccess_Example](https://github.com/icesat2py/icepyx/blob/main/examples/ICESat-2_DAAC_DataAccess_Example.ipynb)

```{code-cell} ipython3
region.order_granules()

#view a short list of order IDs
region.granules.orderIDs

path = 'your data directory'
region.download_granules(path)
```

+++ {"user_expressions": []}

```{admonition} Important Authentication Update
Previously, icepyx required you to explicitly use the `.earthdata_login()` function to login. Running this function is deprecated and will result in an error, as icepyx will call the login function as needed. The user will still need to provide their credentials.
```

+++

### Alternative Access Options to Visualize ICESat-2 elevation using OpenAltimetry API

You can also view elevation data by importing the visualization module directly and initializing it with your query object or a list of parameters:
 ```python
 from icepyx.core.visualization import Visualize
 ```
 - passing your query object directly to the visualization module
 ```python
 region2 = ipx.Query(short_name, spatial_extent, date_range)
 vis = Visualize(region2)
 ```
 - creating a visualization object directly without first creating a query object
 ```python
 vis = Visualize(product=short_name, spatial_extent=spatial_extent, date_range=date_range)
 ```

+++

#### Credits
* Notebook by: [Tian Li](https://github.com/icetianli), [Jessica Scheick](https://github.com/JessicaS11) and 
[Wei Ji](https://github.com/weiji14)
* Source material: [READ_ATL06_DEM Notebook](https://github.com/ICESAT-2HackWeek/Assimilation/blob/master/contributors/icetianli/READ_ATL06_DEM.ipynb) by Tian Li and [Friedrich Knuth](https://github.com/friedrichknuth)
