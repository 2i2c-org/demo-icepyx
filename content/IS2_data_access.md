---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.4
kernelspec:
  display_name: icepyx
  language: python
  name: python3
---

# Accessing ICESat-2 Data
This notebook ({download}`download <IS2_data_access.ipynb>`) illustrates the use of icepyx for programmatic ICESat-2 data query and download from the NASA NSIDC DAAC (NASA National Snow and Ice Data Center Distributed Active Archive Center).
A complimentary notebook demonstrates in greater detail the [subsetting](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access2-subsetting.html) options available when ordering data.

+++

Import packages, including icepyx

```{code-cell} ipython3
import icepyx as ipx
import os
import shutil
%matplotlib inline
```

+++ {"user_expressions": []}

---------------------------------

## Quick-Start Guide

The entire process of getting ICESat-2 data (from query to download) can ultimately be accomplished in two minimal lines of code:

`region_a = ipx.Query(short_name, spatial_extent, date_range)`

`region_a.download_granules(path)`

where the function inputs are described in more detail below.

**The rest of this notebook explains the required inputs used above, optional inputs not available in the minimal example, and the other data search and visualization tools built in to icepyx that make it easier for the user to find, explore, and download ICESat-2 data programmatically from NSIDC.** The detailed steps outlined and the methods showcased below are meant to give the user more control over the data they find and download (including options to order/download only the relevant portions of a data granule), some of which are called using default values behind the scenes if the user simply skips to the `download_granules` step.

+++ {"user_expressions": []}

## Key Steps for Programmatic Data Access

There are several key steps for accessing data from the NSIDC API:
1. Define your parameters (spatial, temporal, dataset, etc.)
2. Query the NSIDC API to find out more information about the dataset
4. Define additional parameters (e.g. subsetting/customization options)
5. Order your data
6. Download your data

icepyx streamlines this process into a minimal number of lines of code.

+++ {"user_expressions": []}

### Create an ICESat-2 data object with the desired search parameters

There are three required inputs, depending on how you want to search for data. Two are required in all cases:
- `short_name` = the data product of interest, known as its "short name".
See https://nsidc.org/data/icesat-2/products for a list of the available data products.
- `spatial extent` = a region of interest to search within. This can be entered as a bounding box, polygon vertex coordinate pairs, or a polygon geospatial file (currently shp, kml, and gpkg are supported).
    - bounding box: Given in decimal degrees for the lower left longitude, lower left latitude, upper right longitude, and upper right latitude
    - polygon vertices: Given as longitude, latitude coordinate pairs of decimal degrees with the last entry a repeat of the first.
    - polygon file: A string containing the full file path and name.
    
*NOTE: The input keyword for `short_name` was updated in the code from `dataset` to `product` to match common usage.
This should not affect users providing positional inputs as demonstrated in this tutorial.*

*NOTE: You can submit at most one bounding box or a list of lonlat polygon coordinates per object instance.
Per NSIDC requirements, geospatial polygon files may only contain one feature (polygon).*

Then, for all non-gridded products (ATL<=13), you must include AT LEAST one of the following inputs (temporal or orbital constraints):
- `date_range` = the date range for which you would like to search for results. The following formats are accepted: 
    - A list of two 'YYYY-MM-DD' strings separated by a comma
    - A list of two 'YYYY-DOY' strings separated by a comma
    - A list of two datetime.date or datetime.datetime objects
    - Dict with the following keys:
        - `start_date`: start date, type can be datetime.datetime, datetime.date, or strings (format 'YYYY-MM-DD' or 'YYYY-DOY')
        - `end_date`: end date, type can be datetime.datetime, datetime.date, or strings (format 'YYYY-MM-DD' or 'YYYY-DOY')
- `cycles` = Which orbital cycle to use, input as a numerical string or a list of strings. If no input is given, this value defaults to all available cycles within the search parameters.  An orbital cycle refers to the 91-day repeat period of the ICESat-2 orbit.
- `tracks` = Which [Reference Ground Track (RGT)](https://icesat-2.gsfc.nasa.gov/science/specs) to use, input as a numerical string or a list of strings. If no input is given, this value defaults to all available RGTs within the spatial and temporal search parameters.

Below are examples of each type of spatial extent and temporal input and an example using orbital parameters. Please choose and run only one of the input option cells to set your spatial and temporal parameters.

```{code-cell} ipython3
# bounding box
short_name = 'ATL06'
spatial_extent = [-55, 68, -48, 71]
date_range = ['2019-02-20','2019-02-28']
```

```{code-cell} ipython3
# polygon vertices (here equivalent to the bounding box, above)
short_name = 'ATL06'
spatial_extent = [(-55, 68), (-55, 71), (-48, 71), (-48, 68), (-55, 68)]
date_range = ['2019-02-20','2019-02-28']
```

```{code-cell} ipython3
# bounding box with 'YYYY-DOY' date range (equivalent to 'YYYY-MM-DD' date ranges above)
short_name = 'ATL06'
spatial_extent = [-55, 68, -48, 71]
date_range = ['2019-051','2019-059']
```

```{code-cell} ipython3
# polygon vertices with datetime.datetime date ranges
import datetime as dt

start_dt = dt.datetime(2019, 2, 20, 0, 10, 0)
end_dt = dt.datetime(2019, 2, 28, 14, 45, 30)
short_name = 'ATL06'
spatial_extent = [(-55, 68), (-55, 71), (-48, 71), (-48, 68), (-55, 68)]
date_range = [start_dt, end_dt]
```

```{code-cell} ipython3
# bounding box with dict containing date ranges
short_name = 'ATL06'
spatial_extent = [-55, 68, -48, 71]
date_range = {"start_date": start_dt, "end_date": '2019-02-28'}
```

```{code-cell} ipython3
# polygon geospatial file (metadata match but no subset match)
# short_name = 'ATL06'
# spatial_extent = './supporting_files/data-access_PineIsland/glims_polygons.kml'
# date_range = ['2019-02-22','2019-02-28']

# #polygon geospatial file (subset and metadata match)
# short_name = 'ATL06'
# spatial_extent = './supporting_files/data-access_PineIsland/glims_polygons.shp'
# date_range = ['2019-10-01','2019-10-05']

#polygon geospatial file (same area as other examples; subset and metadata match)
short_name = 'ATL06'
spatial_extent = './supporting_files/simple_test_poly.gpkg'
date_range = ['2019-10-01','2019-10-05']
```

+++ {"user_expressions": []}

Create the data object using our inputs

```{code-cell} ipython3
region_a = ipx.Query(short_name, spatial_extent, date_range)
```

```{code-cell} ipython3
# using orbital parameters with one of the above data products + spatial parameters
region_a = ipx.Query(short_name, spatial_extent,
   cycles=['03','04','05','06','07'], tracks=['0849','0902'])

print(region_a.product)
print(region_a.product_version)
print(region_a.cycles)
print(region_a.tracks)
```

+++ {"user_expressions": []}

These properties include visualization of the spatial extent on a map. The style of map you will see depends on whether or not you have a certain library, `geoviews`, installed. Under the hood, this is because the `proj` library must be installed with conda (it is not available from PyPI) to support some `geoviews` dependencies. With `geoviews`, this plotting function returns an interactive map. Otherwise, your spatial extent will plot on a static map using `matplotlib`.

```{code-cell} ipython3
# print(region_a.spatial_extent)
region_a.visualize_spatial_extent()
```

+++ {"user_expressions": []}

Formatted parameters and function calls allow us to see the the properties of the data object we have created.

```{code-cell} ipython3
print(region_a.product)
print(region_a.temporal) # .dates, .start_time, .end_time can also be used for a piece of this information
# print(region_a.dates)
# print(region_a.start_time)
# print(region_a.end_time)
print(region_a.cycles)
print(region_a.tracks)
print(region_a.product_version)
region_a.visualize_spatial_extent()
```

+++ {"user_expressions": []}

There are also several optional inputs to allow the user finer control over their search. Start and end time are only valid inputs on a temporally limited search, and they are ignored if your `date_range` input is a datetime.datetime object.
- `start_time` = start time to search for data on the start date. If no input is given, this defaults to 00:00:00.
- `end_time` = end time for the end date of the temporal search parameter. If no input is given, this defaults to 23:59:59. 

Times must be input as 'HH:mm:ss' strings or datetime.time objects.

- `version` = What version of the data product to use, input as a numerical string. If no input is given, this value defaults to the most recent version of the product specified in `short_name`.

*NOTE Version 002 is used as an example in the below cell. However, using it will cause 'no results' errors in granule ordering for some search parameters. These issues have been resolved in later versions of the data products, so it is best to use the most recent version where possible.
Similarly, if you try to order/download too old a version (such that it is no longer hosted by NSIDC), you will get a "no data matched your request" error.
Thus, you will need to update the version associated with `region_a` and rerun the next cell for the rest of this notebook to run.*

```{code-cell} ipython3
region_a = ipx.Query(short_name, spatial_extent, date_range, \
   start_time='03:30:00', end_time='21:30:00', version='002')

print(region_a.product)
print(region_a.dates)
print(region_a.product_version)
print(region_a.spatial)
print(region_a.temporal)
```

+++ {"user_expressions": []}

Alternatively, you can also just create the query object without creating named variables first:

```{code-cell} ipython3
# region_a = ipx.Query('ATL06',[-55, 68, -48, 71],['2019-02-01','2019-02-28'], 
#                            start_time='00:00:00', end_time='23:59:59', version='002')
```

+++ {"user_expressions": []}

### More information about your query object
In addition to viewing the stored object information shown above (e.g. product short name, start and end date and time, version, etc.), we can also request summary information about the data product itself or confirm that we have manually specified the latest version.

```{code-cell} ipython3
region_a.product_summary_info()
print(region_a.latest_version())
```

+++ {"user_expressions": []}

If the summary does not provide all of the information you are looking for, or you would like to see information for previous versions of the data product, all available metadata for the collection product is available in a readable format.

```{code-cell} ipython3
region_a.product_all_info()
```

+++ {"user_expressions": []}

### Querying a data product
In order to search the product collection for available data granules, we need to build our search parameters. This is done automatically behind the scenes when you run `region_a.avail_granules()`, but you can also build and view them by calling `region_a.CMRparams`. These are formatted as a dictionary of key:value pairs according to the [CMR documentation](https://cmr.earthdata.nasa.gov/search/site/docs/search/api.html).

```{code-cell} ipython3
#build and view the parameters that will be submitted in our query
region_a.CMRparams
```

+++ {"user_expressions": []}

Now that our parameter dictionary is constructed, we can search the CMR database for the available granules.
Granules returned by the CMR metadata search are automatically stored within the data object.
The search completed at this level relies completely on the granules' metadata.
As a result, some (and in rare cases all) of the granules returned may not actually contain data in your specified region, particularly if the region is small or located near the boundaries of a given granule. If this is the case, the subsetter will not return any data when you actually place the order.
A warning message will be issued during ordering for each granule to which this applies (but no message is output for successfully subsetted granules, so don't worry!)

```{code-cell} ipython3
#search for available granules and provide basic summary info about them
region_a.avail_granules()
```

```{code-cell} ipython3
#get a list of granule IDs for the available granules
region_a.avail_granules(ids=True)
```

```{code-cell} ipython3
#print detailed information about the returned search results
region_a.granules.avail
```

+++ {"user_expressions": []}

### Log in to NASA Earthdata
When downloading data from NSIDC, all users must login using a valid (free) Earthdata account. The process of authenticating is handled by icepyx by creating and handling the required authentication to interface with the data at the DAAC (including ordering and download). Authentication is completed as login-protected features are accessed. In order to allow icepyx to login for us we still have to make sure that we have made our Earthdata credentials available for icepyx to find.

There are multiple ways to provide your Earthdata credentials via icepyx. Behind the scenes, icepyx is using the [earthaccess library](https://nsidc.github.io/earthaccess/). The [earthaccess documentation](https://earthaccess.readthedocs.io/en/latest/tutorials/getting-started/#auth) automatically tries three primary mechanisms for logging in, all of which are supported by icepyx:
- with `EARTHDATA_USERNAME` and `EARTHDATA_PASSWORD` environment variables (these are the same as the ones you might have set for icepyx previously)
- through an interactive, in-notebook login (used below); passwords are not shown plain text with this option
- with stored credentials in a .netrc file (not recommended for security reasons)

+++ {"user_expressions": []}

```{admonition} Important Authentication Update
Previously, icepyx required you to explicitly use the `.earthdata_login()` function to login. Running this function is deprecated and will result in an error, as icepyx will call the login function as needed. The user will still need to provide their credentials using one of the three methods described above.
```

+++

### Additional Parameters and Subsetting

Once we have generated our session, we must build the required configuration parameters needed to actually download data. These will tell the system how we want to download the data. As with the CMR search parameters, these will be built automatically when you run `region_a.order_granules()`, but you can also create and view them with `region_a.reqparams`. The default parameters, given below, should work for most users.
- `page_size` = 2000. This is the number of granules we will request per order.
- `page_num` = 1. Determine the number of pages based on page size and the number of granules available. If no page_num is specified, this calculation is done automatically to set page_num, which then provides the number of individual orders we will request given the number of granules.
- `request_mode` = 'async'
- `agent` = 'NO'
- `include_meta` = 'Y'

#### More details about the configuration parameters
`request_mode` is "asynchronous" by default, which allows concurrent requests to be queued and processed without the need for a continuous connection between you and the API endpoint.
In contrast, using a "synchronous" `request_mode` means that the request relies on a direct, continuous connection between you and the API endpoint.
Outputs are directly downloaded, or "streamed", to your working directory.
For this tutorial, we will set the request mode to asynchronous.

**Use the streaming `request_mode` with caution: While it can be beneficial to stream outputs directly to your local directory, note that timeout errors can result depending on the size of the request, and your request will not be queued in the system if NSIDC is experiencing high request volume. For best performance, NSIDC recommends setting `page_size=1` to download individual outputs, which will eliminate extra time needed to zip outputs and will ensure faster processing times per request.**

Recall that we queried the total number and volume of granules prior to applying customization services. `page_size` and `page_num` can be used to adjust the number of granules per request up to a limit of 2000 granules for asynchronous, and 100 granules for synchronous (streaming). For now, let's select 9 granules to be processed in each zipped request. For ATL06, the granule size can exceed 100 MB so we want to choose a granule count that provides us with a reasonable zipped download size. 

```{code-cell} ipython3
print(region_a.reqparams)
# region_a.reqparams['page_size'] = 9
# print(region_a.reqparams)
```

#### Subsetting

In addition to the required parameters (CMRparams and reqparams) that are submitted with our order, for ICESat-2 data products we can also submit subsetting parameters to NSIDC.
For a deeper dive into subsetting, please see our [Subsetting Tutorial Notebook](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access2-subsetting.html), which covers subsetting in more detail, including how to get a list of subsetting options, how to build your list of subsetting parameters, and how to generate a list of desired variables (most datasets have more than 200 variable fields!), including using pre-built default lists (these lists are still in progress and we welcome contributions!).

Subsetting utilizes the NSIDC's built in subsetter to extract only the data you are interested (spatially, temporally, variables of interest, etc.). The advantages of using the NSIDC's subsetter include:
* easily reproducible downloads, particularly when coupled with an icepyx query object
* smaller file size, meaning faster downloads, less storage required, and no need to subset the data on your own
* still easy to go back and order more data/variables with the same or similar search parameters
* no extraneous data means you can move directly to analysis and easily navigate your dataset

Certain subset parameters are specified by default unless `subset=False` is included as an input to `order_granules()` or `download_granules()` (which calls `order_granules()` under the hood). A separate, companion notebook tutorial covers subsetting in more detail, including how to get a list of subsetting options, how to build your list of subsetting parameters, and how to generate a list of desired variables (most products have more than 200 variable fields!), including using pre-built default lists (these lists are still in progress and we welcome contributions!).

As for the CMR and required parameters, default subset parameters can be built and viewed using `subsetparams`. Where an input spatial file is used, rather than a bounding box or manually entered polygon, the spatial file will be used for subsetting (unless subset is set to False) but not show up in the `subsetparams` dictionary.

icepyx also makes it easy to take advantage of the reformatting (e.g. file format conversion) options offered by NSIDC. These are covered in more detail in the [Subsetting Tutorial Notebook](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access2-subsetting.html).

```{code-cell} ipython3
region_a.subsetparams()
```

### Place the order
Then, we can send the order to NSIDC using the order_granules function. Information about the granules ordered and their status will be printed automatically. Status information can also be emailed to the address associated with your EarthData account when the `email` kwarg is set to `True`. Additional information on the order, including request URLs, can be viewed by setting the optional keyword input 'verbose' to True.

```{code-cell} ipython3
region_a.order_granules()
# region_a.order_granules(verbose=True, subset=False, email=False)
```

```{code-cell} ipython3
#view a short list of order IDs
region_a.granules.orderIDs
```

### Download the order
Finally, we can download our order to a specified directory (which needs to have a full path but doesn't have to point to an existing directory) and the download status will be printed as the program runs. Additional information is again available by using the optional boolean keyword `verbose`.

```{code-cell} ipython3
path = './download'
region_a.download_granules(path)
```

**Credits**
* original notebook by: Jessica Scheick
* notebook contributors: Amy Steiker and Tyler Sutterley
* source material: [NSIDC Data Access Notebook](https://github.com/ICESAT-2HackWeek/ICESat2_hackweek_tutorials/tree/master/03_NSIDCDataAccess_Steiker) by Amy Steiker and Bruce Wallin and [2020 Hackweek Data Access Notebook](https://github.com/ICESAT-2HackWeek/2020_ICESat-2_Hackweek_Tutorials/blob/main/06-07.Data_Access/02-Data_Access_rendered.ipynb) by Jessica Scheick and Amy Steiker
