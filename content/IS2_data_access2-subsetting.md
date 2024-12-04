---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.4
---

+++ {"user_expressions": []}

# Subsetting ICESat-2 Data

This notebook ({download}`download <IS2_data_access2-subsetting.ipynb>`) illustrates the use of icepyx for subsetting ICESat-2 data ordered through the NSIDC DAAC. We'll show how to find out what subsetting options are available and how to specify the subsetting options for your order.

For more information on using icepyx to find, order, and download data, see our complimentary [ICESat-2 Data Access Notebook](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access.html).

Questions? Be sure to check out the FAQs throughout this notebook, indicated as italic headings.

+++

### _What is SUBSETTING anyway?_

_Anyone who's worked with geospatial data has probably encountered subsetting. Typically, we search for data wherever it is stored and download the chunks (aka granules, scenes, passes, swaths, etc.) that contain something we are interested in. Then, we have to extract from each chunk the pieces we actually want to analyze. Those pieces might be geospatial (i.e. an area of interest), temporal (i.e. certain months of a time series), and/or certain variables. This process of extracting the data we are going to use is called subsetting._

_In the case of ICESat-2 data coming from the NSIDC DAAC, we can do this subsetting step on the data prior to download, reducing our number of data processing steps and resulting in smaller, faster downloads and storage._

+++

Import packages, including icepyx

```{code-cell} ipython3
import icepyx as ipx

import numpy as np
import xarray as xr
import pandas as pd

import h5py
import os,json
from pprint import pprint
```

+++ {"user_expressions": []}

Create a query object and log in to Earthdata

For this example, we'll be working with a sea ice product (ATL09) for an area along West Greenland (Disko Bay).

```{code-cell} ipython3
region_a = ipx.Query('ATL09',[-55, 68, -48, 71],['2019-02-22','2019-02-28'], \
                           start_time='00:00:00', end_time='23:59:59')
```

+++ {"user_expressions": []}

```{admonition} Important Authentication Update
Previously, icepyx required you to explicitly use the `.earthdata_login()` function to login. Running this function is deprecated and will result in an error, as icepyx will call the login function as needed. The user will still need to provide their credentials.
```

+++ {"user_expressions": []}

## Discover Subsetting Options

You can see what subsetting options are available for a given product by calling `show_custom_options()`. The options are presented as a series of headings followed by available values in square brackets. Headings are:

- **Subsetting Options**: whether or not temporal and spatial subsetting are available for the data product
- **Data File Formats (Reformatting Options)**: return the data in a format other than the native hdf5 (submitted as a key=value kwarg to `order_granules(format='NetCDF4-CF')`)
- **Data File (Reformatting) Options Supporting Reprojection**: return the data in a reprojected reference frame. These will be available for gridded ICESat-2 L3B data products.
- **Data File (Reformatting) Options NOT Supporting Reprojection**: data file formats that cannot be delivered with reprojection
- **Data Variables (also Subsettable)**: a dictionary of variable name keys and the paths to those variables available in the product

```{code-cell} ipython3
region_a.show_custom_options(dictview=True)
```

+++ {"user_expressions": []}

By default, spatial and temporal subsetting based on your initial inputs is applied to your order unless you specify `subset=False` to `order_granules()` or `download_granules()` (which calls `order_granules` under the hood if you have not already placed your order) functions.
Additional subsetting options must be specified as keyword arguments to the order/download functions.

Although some file format conversions and reprojections are possible using the `format`, `projection`,and `projection_parameters` keywords, the rest of this tutorial will focus on variable subsetting, which is provided with the `Coverage` keyword.

+++ {"user_expressions": []}

### _Why do I have to provide spatial bounds to icepyx even if I don't use them to subset my data order?_

_Because they're still needed for the granule level search._
_Spatial inputs are usually required for any data search, on any platform, even if your search parameters cover the entire globe._

_The spatial information you provide is used to search the data repository and determine which granules might contain data over your area of interest._
_When you use that spatial information for subsetting, it's actually asking the NSIDC subsetter to extract the appropriate data from each granule._
_Thus, even if you set `subset=False` and download entire granules, you still need to provide some inputs on what geographic area you'd like data for._

+++ {"user_expressions": []}

## About Data Variables in a query object

A given ICESat-2 product may have over 200 variable + path combinations.
icepyx includes a custom `Variables` module that is "aware" of the ATLAS sensor and how the ICESat-2 data products are stored.
The [ICESat-2 Data Variables Example](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_variables.html) provides a detailed set of examples on how to use icepyx's built in `Variables` module.

Thus, this notebook uses a default list of wanted variables to showcase subsetting and refers the user to the aforementioned Jupyter Notebook for a more thorough exploration of ICESat-2 product variables.

+++ {"user_expressions": []}

### Determine what variables are available for your data product

There are multiple ways to get a complete list of available variables.
To increase readability, some display options (2 and 3, below) show the 200+ variable + path combinations as a dictionary where the keys are variable names and the values are the paths to that variable.

1. `region_a.order_vars.avail`, a list of all valid path+variable strings
2. `region_a.show_custom_options(dictview=True)`, all available subsetting options
3. `region_a.order_vars.parse_var_list(region_a.order_vars.avail)`, a dictionary of variable:paths key:value pairs

```{code-cell} ipython3
region_a.order_vars.avail()
```

+++ {"user_expressions": []}

By passing the boolean `options=True` to the `avail` method, you can obtain lists of unique possible variable inputs (var_list inputs) and path subdirectory inputs (keyword_list and beam_list inputs) for your data product. These can be helpful for building your wanted variable list.

```{code-cell} ipython3
region_a.order_vars.avail(options=True)
```

## _Why not just download all the data and subset locally? What if I need more variables/granules?_

_Taking advantage of the NSIDC subsetter is a great way to reduce your download size and thus your download time and the amount of storage required, especially if you're storing your data locally during analysis. By downloading your data using icepyx, it is easy to go back and get additional data with the same, similar, or different parameters (e.g. you can keep the same spatial and temporal bounds but change the variable list). Related tools (e.g. [`captoolkit`](https://github.com/fspaolo/captoolkit)) will let you easily merge files if you're uncomfortable merging them during read-in for processing._

+++

### Building the default wanted variable list

```{code-cell} ipython3
region_a.order_vars.wanted
```

```{code-cell} ipython3
region_a.order_vars.append(defaults=True)
pprint(region_a.order_vars.wanted)
```

## Applying variable subsetting to your order and download

In order to have your wanted variable list included with your order, you must pass it as a keyword argument to the `subsetparams()` attribute or the `order_granules()` or `download_granules()` (which calls `order_granules` under the hood if you have not already placed your order) functions.

```{code-cell} ipython3
region_a.subsetparams(Coverage=region_a.order_vars.wanted)
```

Or, you can put the `Coverage` parameter directly into `order_granules`:
`region_a.order_granules(Coverage=region_a.order_vars.wanted)`

However, then you cannot view your subset parameters (`region_a.subsetparams`) prior to submitting your order.

```{code-cell} ipython3
region_a.order_granules()# <-- you do not need to include the 'Coverage' kwarg to
                             # order if you have already included it in a call to subsetparams
```

```{code-cell} ipython3
region_a.download_granules('/home/jovyan/icepyx/dev-notebooks/vardata') # <-- you do not need to include the 'Coverage' kwarg to
                             # download if you have already submitted it with your order
```

### _Why does the subsetter say no matching data was found?_

_Sometimes, granules ("files") returned in our initial search end up not containing any data in our specified area of interest._
_This is because the initial search is completed using summary metadata for a granule._
_You've likely encountered this before when viewing available imagery online: your spatial search turns up a bunch of images with only a few border or corner pixels, maybe even in no data regions, in your area of interest._
_Thus, when you go to extract the data from the area you want (i.e. spatially subset it), you don't get any usable data from that image._

+++

### Check the variable list in your downloaded file

Compare the available variables associated with the full product relative to those in your downloaded data file.

```{code-cell} ipython3
# put the full filepath to a data file here. You can get this in JupyterHub by navigating to the file,
# right clicking, and selecting copy path. Then you can paste the path in the quotes below.
fn = ''
```

## Check the downloaded data

Get all `latitude` variables in your downloaded file:

```{code-cell} ipython3
varname = 'latitude'

varlist = []
def IS2h5walk(vname, h5node):
    if isinstance(h5node, h5py.Dataset):
        varlist.append(vname)
    return

with h5py.File(fn,'r') as h5pt:
    h5pt.visititems(IS2h5walk)

for tvar in varlist:
    vpath,vn = os.path.split(tvar)
    if vn==varname: print(tvar)
```

### Compare to the variable paths available in the original data

```{code-cell} ipython3
region_a.order_vars.parse_var_list(region_a.order_vars.avail)[0][varname]
```

#### Credits

- notebook contributors: Zheng Liu, Jessica Scheick, and Amy Steiker
- some source material: [NSIDC Data Access Notebook](https://github.com/ICESAT-2HackWeek/ICESat2_hackweek_tutorials/tree/main/03_NSIDCDataAccess_Steiker) by Amy Steiker and Bruce Wallin
