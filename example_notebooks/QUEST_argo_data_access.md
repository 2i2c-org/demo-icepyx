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

# QUEST Example: Finding Argo and ICESat-2 data

In this notebook, we are going to find Argo and ICESat-2 data over a region of the Pacific Ocean. Normally, we would require multiple data portals or Python packages to accomplish this. However, thanks to the [QUEST (Query, Unify, Explore SpatioTemporal) module](https://icepyx.readthedocs.io/en/latest/contributing/quest-available-datasets.html), we can use icepyx to find both!

```{code-cell} ipython3
# Basic packages
import geopandas as gpd
import matplotlib.pyplot as plt
import numpy as np
from pprint import pprint

# icepyx and QUEST
import icepyx as ipx
```

+++ {"user_expressions": []}

## Define the Quest Object

QUEST builds off of the general querying process originally designed for ICESat-2, but makes it applicable to other datasets.

Just like the ICESat-2 Query object, we begin by defining our Quest object. We provide the following bounding parameters:
* `spatial_extent`: Data is constrained to the given box over the Pacific Ocean.
* `date_range`: Only grab data from April 18-19, 2022 (to keep download sizes small for this example).

```{code-cell} ipython3
# Spatial bounds, given as SW/NE corners
spatial_extent = [-154, 30, -143, 37]

# Start and end dates, in YYYY-MM-DD format
date_range = ['2022-04-18', '2022-04-19']

# Initialize the QUEST object
reg_a = ipx.Quest(spatial_extent=spatial_extent, date_range=date_range)

print(reg_a)
```

+++ {"user_expressions": []}

Notice that we have defined our spatial and temporal domains, but we do not have any datasets in our QUEST object. The next section leads us through that process.

+++ {"user_expressions": []}

## Getting the data

Let's first query the ICESat-2 data. If we want to extract information about the water column, the ATL03 product is likely the desired choice.
* `short_name`: ATL03

```{code-cell} ipython3
# ICESat-2 product
short_name = 'ATL03'

# Add ICESat-2 to QUEST datasets
reg_a.add_icesat2(product=short_name)
print(reg_a)
```

+++ {"user_expressions": []}

Let's see the available files over this region.

```{code-cell} ipython3
pprint(reg_a.datasets['icesat2'].avail_granules(ids=True))
```

+++ {"user_expressions": []}

Note the ICESat-2 functions shown here are the same as those used for direct icepyx queries. The user is referred to other [example workbooks](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_access.html) for detailed explanations about icepyx functionality.

Accessing ICESat-2 data requires Earthdata login credentials. When running the `download_all()` function below, an authentication check will be passed when attempting to download the ICESat-2 files.

+++ {"user_expressions": []}

Now let's grab Argo data using the same constraints. This is as simple as using the below function.

```{code-cell} ipython3
# Add argo to the desired QUEST datasets
reg_a.add_argo()
```

+++ {"user_expressions": []}

When accessing Argo data, the variables of interest will be organized as vertical profiles as a function of pressure. By default, only temperature is queried, so the user should supply a list of desired parameters using the code below. The user may also limit the pressure range of the returned data by passing `presRange="0,200"`.

*Note: Our example shows only physical Argo float parameters, but the process is identical for including BGC float parameters.*

```{code-cell} ipython3
# Customized variable query to retrieve salinity instead of temperature
reg_a.add_argo(params=['salinity'])
```

+++ {"user_expressions": []}

Additionally, a user may view or update the list of requested Argo and Argo-BGC parameters at any time through `reg_a.datasets['argo'].params`. If a user submits an invalid parameter ("temp" instead of "temperature", for example), an `AssertionError` will be raised. `reg_a.datasets['argo'].presRange` behaves anologously for limiting the pressure range of Argo data.

```{code-cell} ipython3
# update the list of argo parameters
reg_a.datasets['argo'].params = ['temperature','salinity']

# show the current list
reg_a.datasets['argo'].params
```

+++ {"user_expressions": []}

As for ICESat-2 data, the user can interact directly with the Argo data object (`reg_a.datasets['argo']`) to search or download data outside of the `Quest.search_all()` and `Quest.download_all()` functionality shown below.

The approach to directly search or download Argo data is to use `reg_a.datasets['argo'].search_data()`, and `reg_a.datasets['argo'].download()`. In both cases, the existing parameters and pressure ranges are used unless the user passes new `params` and/or `presRange` kwargs, respectively, which will directly update those values (stored attributes).

+++ {"user_expressions": []}

With our current setup, let's see what Argo parameters we will get.

```{code-cell} ipython3
# see what argo parameters will be searched for or downloaded
reg_a.datasets['argo'].params
```

```{code-cell} ipython3
reg_a.datasets['argo'].search_data()
```

+++ {"user_expressions": []}

Now we can access the data for both Argo and ICESat-2! The below function will do this for us.

**Important**: The Argo data will be compiled into a Pandas DataFrame, which must be manually saved by the user as demonstrated below. The ICESat-2 data is saved as processed HDF-5 files to the directory provided.

```{code-cell} ipython3
path = './quest/downloaded-data/'

# Access Argo and ICESat-2 data simultaneously
reg_a.download_all(path=path)
```

+++ {"user_expressions": []}

We now have one available Argo profile, containing `temperature` and `pressure`, in a Pandas DataFrame. BGC Argo is also available through QUEST, so we could add more variables to this list.

If the user wishes to add more profiles, parameters, and/or pressure ranges to a pre-existing DataFrame, then they should use `reg_a.datasets['argo'].download(keep_existing=True)` to retain previously downloaded data and have the new data added.

+++ {"user_expressions": []}

The `reg_a.download_all()` function also provided a file containing ICESat-2 ATL03 data. Recall that because these data files are very large, we focus on only one file for this example.

The below workflow uses the icepyx Read module to quickly load ICESat-2 data into an Xarray DataSet. To read in multiple files, see the [icepyx Read tutorial](https://icepyx.readthedocs.io/en/latest/example_notebooks/IS2_data_read-in.html) for how to change your input source.

```{code-cell} ipython3
filename = 'processed_ATL03_20220419002753_04111506_006_02.h5'

reader = ipx.Read(data_source=path+filename)
```

```{code-cell} ipython3
# decide which portions of the file to read in
reader.vars.append(beam_list=['gt2l'], 
                   var_list=['h_ph', "lat_ph", "lon_ph", 'signal_conf_ph'])
```

```{code-cell} ipython3
ds = reader.load()
ds
```

+++ {"user_expressions": []}

To make the data more easily plottable, let's convert the data into a Pandas DataFrame. Note that this method is memory-intensive for ATL03 data, so users are suggested to look at small spatial domains to prevent the notebook from crashing. Here, since we only have data from one granule and ground track, we have sped up the conversion to a dataframe by first removing extra data dimensions we don't need for our plots. Several of the other steps completed below using Pandas have analogous operations in Xarray that would further reduce memory requirements and computation times.

```{code-cell} ipython3
is2_pd =(ds.squeeze()
        .reset_coords()
        .drop_vars(["source_file","data_start_utc","data_end_utc","gran_idx"])
        .to_dataframe()
        )
```

```{code-cell} ipython3
is2_pd
```

```{code-cell} ipython3
# Create a new dataframe with only "ocean" photons, as indicated by the "ds_surf_type" flag
is2_pd = is2_pd.reset_index(level=[0,1])
is2_pd_ocean = is2_pd[is2_pd.ds_surf_type==1].drop(columns="photon_idx")
is2_pd_ocean
```

```{code-cell} ipython3
# Set Argo data as its own DataFrame
argo_df = reg_a.datasets['argo'].argodata
```

```{code-cell} ipython3
# Convert both DataFrames into GeoDataFrames
is2_gdf = gpd.GeoDataFrame(is2_pd_ocean, 
                           geometry=gpd.points_from_xy(is2_pd_ocean['lon_ph'], is2_pd_ocean['lat_ph']),
                           crs='EPSG:4326'
)
argo_gdf = gpd.GeoDataFrame(argo_df, 
                            geometry=gpd.points_from_xy(argo_df.lon, argo_df.lat),
                            crs='EPSG:4326'
)
```

+++ {"user_expressions": []}

To view the relative locations of ICESat-2 and Argo, the below cell uses the `explore()` function from GeoPandas. The time variables cause errors in the function, so we will drop those variables first. 

Note that for large datasets like ICESat-2, loading the map might take a while.

```{code-cell} ipython3
# Drop time variables that would cause errors in explore() function
is2_gdf = is2_gdf.drop(['delta_time','atlas_sdp_gps_epoch'], axis=1)
```

```{code-cell} ipython3
# Plot ICESat-2 track (medium/high confidence photons only) on a map
m = is2_gdf[is2_gdf['signal_conf_ph']>=3].explore(column='rgt', tiles='Esri.WorldImagery',
                                                  name='ICESat-2')

# Add Argo float locations to map
argo_gdf.explore(m=m, name='Argo', marker_kwds={"radius": 6}, color='red')
```

+++ {"user_expressions": []}

While we're at it, let's plot temperature and pressure profiles for each of the Argo floats in the area.

```{code-cell} ipython3
# Plot vertical profile of temperature vs. pressure for all of the floats
fig, ax = plt.subplots(figsize=(12, 6))
for pid in np.unique(argo_df['profile_id']):
    argo_df[argo_df['profile_id']==pid].plot(ax=ax, x='temperature', y='pressure', label=pid)
plt.gca().invert_yaxis()
plt.xlabel('Temperature [$\degree$C]')
plt.ylabel('Pressure [hPa]')
plt.ylim([750, -10])
plt.tight_layout()
```

+++ {"user_expressions": []}

Lastly, let's look at some near-coincident ICESat-2 and Argo data in a multi-panel plot.

```{code-cell} ipython3
# Only consider ICESat-2 signal photons
is2_pd_signal = is2_pd_ocean[is2_pd_ocean['signal_conf_ph']>=0]

## Multi-panel plot showing ICESat-2 and Argo data

# Calculate Extent
lons = [-154, -143, -143, -154, -154]
lats = [30, 30, 37, 37, 30]
lon_margin = (max(lons) - min(lons)) * 0.1
lat_margin = (max(lats) - min(lats)) * 0.1

# Create Plot
fig,([ax1,ax2],[ax3,ax4]) = plt.subplots(2, 2, figsize=(12, 6))

# Plot Relative Global View
world = gpd.read_file(gpd.datasets.get_path('naturalearth_lowres'))
world.plot(ax=ax1, color='0.8', edgecolor='black')
argo_df.plot.scatter(ax=ax1, x='lon', y='lat', s=25.0, c='green', zorder=3, alpha=0.3)
is2_pd_signal.plot.scatter(ax=ax1, x='lon_ph', y='lat_ph', s=10.0, zorder=2, alpha=0.3)
ax1.plot(lons, lats, linewidth=1.5, color='orange', zorder=2)
ax1.set_xlim(-160,-100)
ax1.set_ylim(20,50)
ax1.set_aspect('equal', adjustable='box')
ax1.set_xlabel('Longitude', fontsize=18)
ax1.set_ylabel('Latitude', fontsize=18)

# Plot Zoomed View of Ground Tracks
argo_df.plot.scatter(ax=ax2, x='lon', y='lat', s=50.0, c='green', zorder=3, alpha=0.3)
is2_pd_signal.plot.scatter(ax=ax2, x='lon_ph', y='lat_ph', s=10.0, zorder=2, alpha=0.3)
ax2.plot(lons, lats, linewidth=1.5, color='orange', zorder=1)
ax2.set_xlim(min(lons) - lon_margin, max(lons) + lon_margin)
ax2.set_ylim(min(lats) - lat_margin, max(lats) + lat_margin)
ax2.set_aspect('equal', adjustable='box')
ax2.set_xlabel('Longitude', fontsize=18)
ax2.set_ylabel('Latitude', fontsize=18)

# Plot ICESat-2 along-track vertical profile. A dotted line notes the location of a nearby Argo float
is2 = ax3.scatter(is2_pd_signal['lat_ph'], is2_pd_signal['h_ph']+13.1, s=0.1)
ax3.axvline(34.43885, linestyle='--', linewidth=3, color='black')
ax3.set_xlim([34.3, 34.5])
ax3.set_ylim([-20, 5])
ax3.set_xlabel('Latitude', fontsize=18)
ax3.set_ylabel('Approx. IS-2 Depth [m]', fontsize=16)
ax3.set_yticklabels(['15', '10', '5', '0', '-5'])

# Plot vertical ocean profile of the nearby Argo float
argo_df.plot(ax=ax4, x='temperature', y='pressure', linewidth=3)
# ax4.set_yscale('log')
ax4.invert_yaxis()
ax4.get_legend().remove()
ax4.set_xlabel('Temperature [$\degree$C]', fontsize=18)
ax4.set_ylabel('Argo Pressure', fontsize=16)

plt.tight_layout()

# Save figure
#plt.savefig('/icepyx/quest/figures/is2_argo_figure.png', dpi=500)
```

Recall that the Argo data must be saved manually.
The dataframe associated with the Quest object can be saved using `reg_a.save_all(path)`

```{code-cell} ipython3
reg_a.save_all(path)
```
