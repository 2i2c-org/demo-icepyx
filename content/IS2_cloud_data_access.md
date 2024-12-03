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

+++ {"user_expressions": []}

# ICESat-2 AWS cloud data access
This notebook ({download}`download <IS2_cloud_data_access.ipynb>`) illustrates the use of icepyx for accessing ICESat-2 data currently available through the AWS (Amazon Web Services) us-west2 hub s3 data bucket.

## Notes
1. ICESat-2 data became publicly available on the cloud on 29 September 2022. Thus, access methods and example workflows are still being developed by NSIDC, and the underlying code in icepyx will need to be updated now that these data (and the associated metadata) are available. We appreciate your patience and contributions (e.g. reporting bugs, sharing your code, etc.) during this transition!
2. This example and the code it describes are part of ongoing development. Current limitations to using these features are described throughout the example, as appropriate.
3. You **MUST** be working within an AWS instance. Otherwise, you will get a permissions error.

+++ {"user_expressions": []}

## Querying for data and finding s3 urls

```{code-cell} ipython3
import icepyx as ipx
```

```{code-cell} ipython3
# Make sure the user sees important warnings if they try to read a lot of data from the cloud
import warnings
warnings.filterwarnings("always")
```

+++ {"user_expressions": []}

We will start the way we often do: by creating an icepyx Query object.

```{code-cell} ipython3
short_name = 'ATL03'
spatial_extent = [-45, 58, -35, 75]
date_range = ['2019-11-30','2019-11-30']
```

```{code-cell} ipython3
reg = ipx.Query(short_name, spatial_extent, date_range)
```

+++ {"user_expressions": []}

### Get the granule s3 urls

With this query object you can get a list of available granules. This function returns a list containing the list of the granule IDs and a list of the corresponding urls. Use `cloud=True` to get the needed s3 urls.

```{code-cell} ipython3
gran_ids = reg.avail_granules(ids=True, cloud=True)
gran_ids
```

+++ {"user_expressions": []}

## Determining variables of interest

+++ {"user_expressions": []}

There are several ways to view available variables. One is to use the existing Query object:

```{code-cell} ipython3
reg.order_vars.avail()
```

+++ {"user_expressions": []}

Another way is to use the variables module:

```{code-cell} ipython3
ipx.Variables(product=short_name).avail()
```

+++ {"user_expressions": []}

We can also do this using a specific s3 filepath from the Query object:

```{code-cell} ipython3
ipx.Variables(path=gran_ids[1][0]).avail()
```

+++ {"user_expressions": []}

From any of these methods we can see that `h_ph` is a variable for this data product, so we will read that variable in the next step.

+++ {"user_expressions": []}

#### A Note on listing variables using s3 urls

We can use the Variables module with an s3 url to explore available data variables the same way we do with local files. An important difference, however, is how the available variables list is created. When reading a local file the variables module will traverse the entire file and search for variables that are present in that file. This method it too time intensive with the s3 data, so instead the the product / version of the data product is read from the file and all possible variables associated with that product/version are reporting as available. As long as you are using the NSIDC provided s3 paths provided via Earthdata search and the Query object these lists will be the same.

+++ {"user_expressions": []}

#### A Note on authentication

Notice that accessing cloud data requires two layers of authentication: 1) authenticating with your Earthdata Login 2) authenticating for cloud access. These both happen behind the scenes, without the need for users to provide any explicit commands.

Icepyx uses earthaccess to generate your s3 data access token, which will be valid for *one* hour. Icepyx will also renew the token for you after an hour, so if viewing your token over the course of several hours you may notice the values will change.

If you do want to see your s3 credentials, you can access them using:

```{code-cell} ipython3
# uncommenting the line below will print your temporary aws login credentials
# reg.s3login_credentials
```

+++ {"user_expressions": []}

```{admonition} Important Authentication Update
Previously, icepyx required you to explicitly use the `.earthdata_login()` function to login. Running this function is deprecated and will result in an error, as icepyx will call the login function as needed. The user will still need to provide their credentials.
```

+++ {"user_expressions": []}

## Choose a data file and access the data

**Note: If you get a PermissionDenied Error when trying to read in the data, you may not be sending your request from an AWS hub in us-west2. We're currently working on how to alert users if they will not be able to access ICESat-2 data in the cloud for this reason**

We are ready to read our data! We do this by creating a reader object and using the s3 url returned from the Query object.

```{code-cell} ipython3
# the first index, [1], gets us into the list of s3 urls
# the second index, [0], gets us the first entry in that list.
s3url = gran_ids[1][0]
# s3url =  's3://nsidc-cumulus-prod-protected/ATLAS/ATL03/004/2019/11/30/ATL03_20191130221008_09930503_004_01.h5'
```

+++ {"user_expressions": []}

Create the Read object

```{code-cell} ipython3
reader = ipx.Read(s3url)
```

+++ {"user_expressions": []}

This reader object gives us yet another way to view available variables.

```{code-cell} ipython3
reader.vars.avail()
```

+++ {"user_expressions": []}

Next, we append our desired variable to the `wanted_vars` list:

```{code-cell} ipython3
reader.vars.append(var_list=['h_ph'])
```

+++ {"user_expressions": []}

Finally, we load the data

```{code-cell} ipython3
%%time

# This may take 5-10 minutes
reader.load()
```

+++ {"user_expressions": []}

### Some important caveats

While the cloud data reading is functional within icepyx, it is very slow. Approximate timing shows it takes ~6 minutes of load time per variable per file from s3. Because of this you will receive a warning if you try to load either more than three variables or two files at once.

The slow load speed is a demonstration of the many steps involved in making cloud data actionable - the data supply chain needs optimized source data, efficient low level data readers, and high level libraries which are enabled to use the fastest low level data readers. Not all of these pieces fully developed right now, but the progress being made it exciting and there is lots of room for contribution!

+++ {"user_expressions": []}

#### Credits
* notebook by: Jessica Scheick and Rachel Wegener
