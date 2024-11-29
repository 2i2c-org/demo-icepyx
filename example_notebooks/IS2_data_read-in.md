---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.4
kernelspec:
  display_name: python3
  language: python
  name: python3
---

+++ {"user_expressions": []}

# Reading ICESat-2 Data in for Analysis
This notebook ({download}`download <IS2_data_read-in.ipynb>`) illustrates the use of icepyx for reading ICESat-2 data files, loading them into a data object.
Currently the default data object is an Xarray Dataset, with ongoing work to provide support for other data object types.

For more information on how to order and download ICESat-2 data, see the [icepyx data access tutorial](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access.html).

### Motivation
Most often, when you open a data file, you must specify the underlying data structure and how you'd like the information to be read in.
A simple example of this, for instance when opening a csv or similarly delimited file, is letting the software know if the data contains a header row, what the data type is (string, double, float, boolean, etc.) for each column, what the delimiter is, and which columns or rows you'd like to be loaded.
Many ICESat-2 data readers are quite manual in nature, requiring that you accurately type out a list of string paths to the various data variables.

icepyx simplifies this process by relying on its awareness of ICESat-2 specific data file variable storage structure.
Instead of needing to manually iterate through the beam pairs, you can provide a few options to the `Read` object and icepyx will do the heavy lifting for you (as detailed in this notebook).

### Approach
If you're interested in what's happening under the hood: icepyx uses the [xarray](https://docs.xarray.dev/en/stable/) library to read in each of the requested variables of the dataset. icepyx formats each requested variable and then merges the read-in data from each of the variables to create a single data object. The use of xarray is powerful, because the returned data object can be used with relevant xarray processing tools.

+++

Import packages, including icepyx

```{code-cell} ipython3
import icepyx as ipx
```

+++ {"user_expressions": []}

---------------------------------

## Quick-Start Guide
For those who might be looking into playing with this (but don't want all the details/explanations)

```{code-cell} ipython3
path_root = '/full/path/to/your/ATL06_data/'
reader = ipx.Read(path_root)
```

```{code-cell} ipython3
reader.vars.append(beam_list=['gt1l', 'gt3r'], var_list=['h_li', "latitude", "longitude"])
```

```{code-cell} ipython3
ds = reader.load()
ds
```

```{code-cell} ipython3
ds.plot.scatter(x="longitude", y="latitude", hue="h_li", vmin=-100, vmax=2000)
```

+++ {"user_expressions": []}

---------------------------------------
## Key steps for loading (reading) ICESat-2 data

Reading in ICESat-2 data with icepyx happens in a few simple steps:
1. Let icepyx know where to find your data (this might be local files or urls to data in cloud storage)
2. Create an icepyx `Read` object
3. Make a list of the variables you want to read in (does not apply for gridded products)
4. Load your data into memory (or read it in lazily, if you're using Dask)

We go through each of these steps in more detail in this notebook.

+++ {"user_expressions": []}

### Step 0: Get some data if you haven't already
Here are a few lines of code to get you set up with a few data files if you don't already have some on your local system.

```{code-cell} ipython3
region_a = ipx.Query('ATL06',[-55, 68, -48, 71],['2019-02-22','2019-02-28'], \
                           start_time='00:00:00', end_time='23:59:59')
```

```{code-cell} ipython3
region_a.download_granules(path=path_root)
```

+++ {"user_expressions": []}

```{admonition} Important Authentication Update
Previously, icepyx required you to explicitly use the `.earthdata_login()` function to login. Running this function is deprecated and will result in an error, as icepyx will call the login function as needed. The user will still need to provide their credentials.
```

+++ {"user_expressions": []}

### Step 1: Set data source path

Provide a full path to the data to be read in (i.e. opened).
Currently accepted inputs are:
* a string path to directory - all files from the directory will be opened
* a string path to single file - one file will be opened
* a list of filepaths - all files in the list will be opened
* a glob string (see [glob](https://docs.python.org/3/library/glob.html)) - any files matching the glob pattern will be opened

```{code-cell} ipython3
path_root = '/full/path/to/your/data/'
```

```{code-cell} ipython3
# filepath = path_root + 'ATL06-20181214041627-Sample.h5'
```

```{code-cell} ipython3
# list_of_files = ['/my/data/ATL06/processed_ATL06_20190226005526_09100205_006_02.h5', 
#                  '/my/other/data/ATL06/processed_ATL06_20191202102922_10160505_006_01.h5']
```

+++ {"user_expressions": []}

#### Glob Strings

[glob](https://docs.python.org/3/library/glob.html) is a Python library which allows users to list files in their file systems whose paths match a given pattern. Icepyx uses the glob library to give users greater flexibility over their input file lists.

glob works using `*` and `?` as wildcard characters, where `*` matches any number of characters and `?` matches a single character. For example:

* `/this/path/*.h5`: refers to all `.h5` files in the `/this/path` folder (Example matches: "/this/path/processed_ATL03_20191130221008_09930503_006_01.h5" or "/this/path/myfavoriteicsat-2file.h5")
* `/this/path/*ATL07*.h5`: refers to all `.h5` files in the `/this/path` folder that have ATL07 in the filename. (Example matches: "/this/path/ATL07-02_20221012220720_03391701_005_01.h5" or "/this/path/processed_ATL07.h5")
* `/this/path/ATL??/*.h5`: refers to all `.h5` files that are in a subfolder of `/this/path` and a subdirectory of `ATL` followed by any 2 characters (Example matches: "/this/path/ATL03/processed_ATL03_20191130221008_09930503_006_01.h5", "/this/path/ATL06/myfile.h5")

See the glob documentation or other online explainer tutorials for more in depth explanation, or advanced glob paths such as character classes and ranges.

+++ {"user_expressions": []}

#### Recursive Directory Search

+++ {"user_expressions": []}

glob will not by default search all of the subdirectories for matching filepaths, but it has the ability to do so.

If you would like to search recursively, you can achieve this by either:
1. passing the `recursive` argument into `glob_kwargs` and including `\**\` in your filepath
2. using glob directly to create a list of filepaths

Each of these two methods are shown below.

+++ {"user_expressions": []}

Method 1: passing the `recursive` argument into `glob_kwargs`

```{code-cell} ipython3
ipx.Read('/path/to/**/folder', glob_kwargs={'recursive': True})
```

+++ {"user_expressions": []}

You can use `glob_kwargs` for any additional argument to Python's builtin `glob.glob` that you would like to pass in via icepyx.

+++ {"user_expressions": []}

Method 2: using glob directly to create a list of filepaths

```{code-cell} ipython3
import glob
```

```{code-cell} ipython3
list_of_files = glob.glob('/path/to/**/folder', recursive=True)
ipx.Read(list_of_files)
```

+++ {"user_expressions": []}

```{admonition} Read Module Update
Previously, icepyx required two additional conditions: 1) a `product` argument and 2) that your files either matched the default `filename_pattern` or that the user provided their own `filename_pattern`. These two requirements have been removed. `product` is now read directly from the file metadata (the root group's `short_name` attribute). Flexibility to specify multiple files via the `filename_pattern` has been replaced with the [glob string](https://docs.python.org/3/library/glob.html) feature, and by allowing a list of filepaths as an argument.

The `product` and `filename_pattern` arguments are now deprecated and were removed in icepyx version 1.0.0.
```

+++ {"user_expressions": []}

### Step 2: Create an icepyx read object

Using the `data_source` described in Step 1, we can create our Read object.

```{code-cell} ipython3
reader = ipx.Read(data_source=path_root)
```

+++ {"user_expressions": []}

The Read object now contains the list of matching files that will eventually be loaded into Python. You can inspect its properties, such as the files that were located or the identified product, directly on the Read object.

```{code-cell} ipython3
reader.filelist
```

```{code-cell} ipython3
reader.product
```

+++ {"user_expressions": []}

### Step 3: Specify variables to be read in

To load your data into memory or prepare it for analysis, icepyx needs to know which variables you'd like to read in.
If you've used icepyx to download data from NSIDC with variable subsetting (which is the default), then you may already be familiar with the icepyx `Variables` module and how to create and modify lists of variables.
We showcase a specific case here, but we encourage you to check out [the icepyx Variables example](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_variables.html) for a thorough trip through how to create and manipulate lists of ICESat-2 variable paths (examples are provided for multiple data products).

If you want to see a \[likely very long\] list of all path + variable combinations available to you, this unmutable (unchangeable) list is generated by default from the first file in your list (so not all variables may be contained in all of the files, depending on how you are accessing the data).

```{code-cell} ipython3
reader.vars.avail()
```

+++ {"user_expressions": []}

To make things easier, you can use icepyx's built-in default list that loads commonly used variables for your non-gridded data product, or create your own list of variables to be read in.
icepyx will determine what variables are available for you to read in by creating a list from one of your source files.
If you have multiple files that you're reading in, icepyx will automatically generate a list of filenames and take the first one to get the list of available variables.

Thus, if you have different variables available across files (even from the same data product), you may run into issues and need to come up with a workaround (we can help you do so!).
We anticipate most users will have the minimum set of variables they are seeking to load available across all data files, so we're not currently developing this feature.
Please get in touch if it would be a helpful feature for you or if you encounter this problem!

You may create a variable list for gridded ICESat-2 products. However, all variables in the file will still be added to your DataSet. (This is an area we're currently exploring on expanding - please let us know if you're working on this and would like to contribute!)

+++ {"user_expressions": []}

For a basic case, let's say we want to read in height, latitude, and longitude for all beam pairs.
We create our variables list as

```{code-cell} ipython3
reader.vars.append(var_list=['h_li', "latitude", "longitude"])
```

+++ {"user_expressions": []}

Then we can view a dictionary of the variables we'd like to read in.

```{code-cell} ipython3
reader.vars.wanted
```

+++ {"user_expressions": []}

Don't forget - if you need to start over, and re-generate your wanted variables list, it's easy!

```{code-cell} ipython3
reader.vars.remove(all=True)
```

+++ {"user_expressions": []}

### Step 4: Loading your data

Now that you've set up all the options, you're ready to read your ICESat-2 data into memory!

```{code-cell} ipython3

```

```{code-cell} ipython3
ds = reader.load()
```

+++ {"user_expressions": []}

Within a Jupyter Notebook, you can get a summary view of your data object.

***ATTENTION: icepyx loads your data by creating an Xarray DataSet for each input granule and then merging them. In some cases, the automatic merge fails and needs to be handled manually. In these cases, icepyx will return a warning with the error message from the failed Xarray merge and a list of per-granule DataSets***

This can happen if you unintentionally provide the same granule multiple times with different filenames or in segmented products where the rgt+cycle automatically generated `gran_idx` values match. In this latter case, you can simply provide unique `gran_idx` values for each DataSet in `ds` and run `import xarray as xr` and `ds_merged = xr.merge(ds)` to create one merged DataSet.

```{code-cell} ipython3
ds
```

+++ {"user_expressions": []}

## On to data analysis!

From here, you can begin your analysis.
Ultimately, icepyx aims to include an Xarray extension with ICESat-2 aware functions that allow you to do things like easily use only data from strong beams.
That functionality is still in development.
For fun, we've included a basic plot made with Xarray's built in functionality.

```{code-cell} ipython3
ds.plot.scatter(x="longitude", y="latitude", hue="h_li", vmin=-100, vmax=2000)
```

+++ {"user_expressions": []}

A developer note to users:
our next steps will be to create an xarray extension with ICESat-2 aware functions (like "get_strong_beams", etc.).
Please let us know if you have any ideas or already have functions developed (we can work with you to add them, or add them for you!).

+++ {"user_expressions": []}

#### Credits
* original notebook by: Jessica Scheick
* notebook contributors: Wei Ji and Tian

```{code-cell} ipython3

```

```{code-cell} ipython3

```
