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

# WIP: Comparing ICESat-2 Altimetry Elevations with DEM
This notebook compares elevations from ICESat-2 to those from a DEM.

Note that this notebook was created for a specific event using not-publicly available files.
Thus, it is provided as an example workflow but needs to be updated to use a public DEM and icepyx data read-in capabilities.

+++

### Setup
#### The Notebook was run on ICESat2 Hackweek 2019 pangeo image
#### For full functionality,
- Please install [icepyx](https://github.com/icesat2py/icepyx), [topolib](https://github.com/ICESAT-2HackWeek/topohack), [contextily](https://github.com/darribas/contextily) using `git clone xxxxx`, `pip install -e .` workflow (see below; **you must restart your kernel after installing the packages**)
- Download [NASA ASP](https://github.com/NeoGeographyToolkit/StereoPipeline) tar ball and unzip, we execute the commands from the notebook, using the path to the untared bin folder for the given commands.

```{code-cell} ipython3
%%bash
cd ~
# git clone https://github.com/icesat2py/icepyx.git
# git clone https://github.com/ICESAT-2HackWeek/topohack.git
# git clone https://github.com/darribas/contextily.git

cd contextily
pip install -e .
cd ../topohack
pip install -e .
cd ../icepyx
pip install -e .
```

```{code-cell} ipython3
%cd ~
#needs to be wherever icepyx, contextily, and topolib are installed in the previous step (ideally $HOME)
# %pwd
```

### ICESat-2 product being explored : [ATL08](https://nsidc.org/data/atl08)
- Along track heights for canopy (land and vegitation) and  terrain
- Terrain heights provided are aggregated over every 100 m along track interval, output contains "h_te_best_fit: height from best fit algorithm for all photons in the range", median height and others. Here we use h_te_best_fit.
- See this preliminary introduction and quality assessment [paper](https://www.mdpi.com/2072-4292/11/14/1721) for more detail

+++

## Import packages, including icepyx

```{code-cell} ipython3
import icepyx as ipx
import os
import shutil
import h5py
import xarray as xr
# dependencies
import getpass
#from topolib.subsetDat import subsetBBox;
from topolib import icesat2_data
import glob
import rasterio
from topolib import gda_lib
from topolib import dwnldArctic
import numpy as np
import geopandas as gpd
from multiprocessing import Pool
import contextily as ctx
import pandas as pd
%matplotlib inline
import matplotlib.pyplot as plt
```

```{code-cell} ipython3
%cd ~/icepyx/doc/examples/
```

## Preprocess #1
- Download using icepyx

+++

### Create an ICESat-2 data object with the desired search parameters
- See the ICESat-2 DAAC Data Access notebook for more details on downloading data from the NSIDC

```{code-cell} ipython3
region_a = ipx.Query('ATL08', [-73.9, 10.7, -73.4, 11.1], ['2018-12-01','2019-09-01'], \
                          start_time='00:00:00', end_time='23:59:59')
```

## Finding and downloading data
In order to download any data from NSIDC, we must first authenticate ourselves using a valid Earthdata login (available for free on their website). This will create a valid token to interface with the DAAC as well as start an active logged-in session to enable data download. The token is attached to the data object and stored, but the session must be passed to the download function. Then we can order the granules.

+++

### Log in to Earthdata

```{code-cell} ipython3
#search for available granules
region_a.avail_granules()
```

```{code-cell} ipython3
region_a.granules.avail
```

+++ {"user_expressions": []}

```{admonition} Important Authentication Update
Previously, icepyx required you to explicitly use the `.earthdata_login()` function to login. Running this function is deprecated and will result in an error, as icepyx will call the login function as needed. The user will still need to provide their credentials.
```

+++

### Place the order

```{code-cell} ipython3
region_a.order_granules(subset=False)
#region_a.order_granules(verbose=True)
```

```{code-cell} ipython3
#view a short list of order IDs
region_a.granules.orderIDs
```

### Download the order
Finally, we can download our order to a specified directory (which needs to have a full path but doesn't have to point to an existing directory) and the download status will be printed as the program runs. Additional information is again available by using the optional boolean keyword 'verbose'.

```{code-cell} ipython3
wd=%pwd
path = wd + '/download'
```

```{code-cell} ipython3
region_a.download_granules(path)
```

### Clean up the download folder by removing individual order folders:

```{code-cell} ipython3
#Clean up Outputs folder by removing individual granule folders 
for root, dirs, files in os.walk(path, topdown=False):
    for file in files:
        try:
            shutil.move(os.path.join(root, file), path)
        except OSError:
            pass
        
for root, dirs, files in os.walk(path):
    for name in dirs:
        os.rmdir(os.path.join(root, name))
```

## Preprocess #2
- Convert data into geopandas dataframe, which allows for doing basing geospatial operations

```{code-cell} ipython3
# glob to list of files (run block of code creating wd and path variables if starting processing here)
ATL08_list = sorted(glob.glob(path+'/*.h5'))
```

## Examine content of 1 ATLO8 hdf file

```{code-cell} ipython3
filename = ATL08_list[5]
with h5py.File(filename, 'r') as f:
    # List all groups
    pairs=[1, 2, 3]
    beams=['l','r']
    print("Keys: %s" % f.keys())
    a_group_key = list(f.keys())[0]
#
```

```{code-cell} ipython3
ATL08_list
```

```{code-cell} ipython3
# dict containing data entries to retrieve
dataset_dict = {'land_segments':['delta_time','longitude','latitude','atl06_quality_summary','quality','terrain_flg'], 'land_segments/terrain':['h_te_best_fit']}
```

```{code-cell} ipython3
#gda_lib.ATL08_to_dict(ATL08_list[0],dataset_dict)
```

```{code-cell} ipython3
## the data can be converted to geopandas dataframe, see ATL08_2_gdf function in topolib gda_lib
temp_gdf = gda_lib.ATL08_2_gdf(ATL08_list[0],dataset_dict)
```

```{code-cell} ipython3
temp_gdf.head()
```

```{code-cell} ipython3
%matplotlib inline
```

```{code-cell} ipython3
temp_gdf.plot()
```

```{code-cell} ipython3
len(temp_gdf)
```

```{code-cell} ipython3
colombia_crs = {'init':'epsg:32618'}
plot_web = {'init':'epsg:3857'}
```

```{code-cell} ipython3
temp_gdf.keys()
```

## Convert the list of hdf5 files into more familiar Pandas Dataframe

```{code-cell} ipython3
gdf_list = [(gda_lib.ATL08_2_gdf(x,dataset_dict)) for x in ATL08_list]
gdf_colombia = gda_lib.concat_gdf(gdf_list)
```

## Preprocess #3
- Visualise data footprints

```{code-cell} ipython3
fig,ax = plt.subplots(figsize=(10,10))
temp_web = gdf_colombia.to_crs(plot_web)
clim = np.percentile(temp_web['h_te_best_fit'].values,(2,98))
temp_web.plot('h_te_best_fit',ax=ax,s=3,legend=True,cmap='inferno',vmin=clim[0],vmax=clim[1])
ctx.add_basemap(ax=ax)
ax.set_xticks([])
ax.set_yticks([])
```

## We will use the TANDEM-X Global DEM for our comparison. The resolution of the globally available product is 90 m, with *horizontal* and *vertical* accuracy better than 2 to 3 m.
- TANDEM-X DEM for the region was downloaded and preprocessed, filtered using scripts from the [tandemx](https://github.com/dshean/tandemx) repository

```{code-cell} ipython3
dem_file = os.path.join(wd,'supporting_files/TDM1_DEM_90m_colombia_DEM_masked_aea.tif')
hs_file = os.path.splitext(dem_file)[0]+'_hs.tif'
dem_ds = rasterio.open(dem_file)
```

```{code-cell} ipython3
! gdaldem hillshade $dem_file $hs_file
```

```{code-cell} ipython3
hs_ds = rasterio.open(hs_file)
```

```{code-cell} ipython3
def gdf_on_raster(gdf,ds,ax,hs_ds=None,cmap='inferno'):
    gdf = gdf.to_crs(ds.crs)
    xmin,ymin,xmax,ymax = ds.bounds
    ndv = gda_lib.get_ndv(ds)
    img = ds.read(1)
    img = np.ma.masked_less_equal(img,ndv)
    clim = np.nanpercentile(img,(2,98))
    if hs_ds:
        hs = hs_ds.read(1)
        ndv = gda_lib.get_ndv(hs_ds)
        hs = np.ma.masked_less_equal(hs,ndv)
        ax.imshow(hs,cmap='gray',extent=[xmin,xmax,ymin,ymax])
        im = ax.imshow(img,alpha=0.6,cmap=cmap,extent=[xmin,xmax,ymin,ymax])
        print(clim)
    else:
        im = ax.imshow(img,cmap=cmap,vmin=clim[0],vmax=clim[1],extent=[xmin,xmax,ymin,ymax])
    gdf.plot('p_b',ax=ax,s=1)
    plt.colorbar(im,ax=ax,extend='both',label='Elevation (m)')
```

```{code-cell} ipython3
xmin,ymin,xmax,ymax = dem_ds.bounds
```

```{code-cell} ipython3
## Filter points based on DEM extent
```

```{code-cell} ipython3
gdf_colombia['x_atc'] = gdf_colombia['delta_time']
gdf_colombia_dem_extent = gdf_colombia.to_crs(dem_ds.crs).cx[xmin:xmax,ymin:ymax]
```

```{code-cell} ipython3
fig,ax = plt.subplots(figsize=(10,5))
gdf_on_raster(gdf_colombia_dem_extent,dem_ds,ax,hs_ds)
```

## Section 1
- This contains demonstration of elevation profile along 1 track, which has 6 beams

```{code-cell} ipython3
### Picking out 1 track
### check with different ATLO8 inputs
test_track = ATL08_list[3]
print(test_track)
test_gdf = gda_lib.ATL08_2_gdf(test_track,dataset_dict).to_crs(dem_ds.crs).cx[xmin:xmax,ymin:ymax]
fig,ax = plt.subplots(figsize=(10,5))
gdf_on_raster(test_gdf,dem_ds,ax,hs_ds)
```

```{code-cell} ipython3
## Working with track from 20190105 to show how we can use this to plot elevation values along the collect just by using ICESat-2
np.unique(test_gdf['p_b'].values)
```

```{code-cell} ipython3
# Limit analysis to 1 pair beam combination
mask = test_gdf['p_b']== '1.0_0.0'
test_gdf_pb = test_gdf[mask]
fig,ax = plt.subplots(figsize=(5,4))
plot_var = test_gdf_pb['h_te_best_fit'].values
ax.scatter(np.arange(len(plot_var)),plot_var,s=1)
ax.set_xlabel('Along-track id')
ax.set_ylabel('ATL08-Terrain Height')
ax.grid('--')
```

```{code-cell} ipython3
#or do it for all the 6 tracks
track_identifier = list(np.unique(test_gdf['p_b'].values))
fig,axa = plt.subplots(3,2,figsize=(10,10))
ax = axa.ravel()
for idx,track in enumerate(track_identifier):
    mask = test_gdf['p_b']== track
    test_gdf_pb = test_gdf[mask]
    plot_var = test_gdf_pb['h_te_best_fit'].values
    ax[idx].scatter(np.arange(len(plot_var)),plot_var,s=1)
    ax[idx].set_xlabel('Along-track id')
    ax[idx].set_ylabel('ATL08-Terrain Height')
    ax[idx].grid('--')
    ax[idx].set_title('Track: {} Beam: {}'.format(track.split('_',15)[0],track.split('_',15)[1]))
plt.tight_layout()
```

## Section 2:
- Compare ICESat-2 Elevation with that of reference DEM (in this case TANDEM-X)

+++

### Sample elevations from DEM at ATLO8-locations using nearest neighbour algorithm 

```{code-cell} ipython3
del_time,elev = gda_lib.sample_near_nbor(dem_ds,gdf_colombia_dem_extent)
```

```{code-cell} ipython3
gdf_colombia_dem_extent['dem_z'] = elev
```

### Plot elevation differences (ICESat-2 minus TANDEM-X) as a function of elevation

```{code-cell} ipython3
gdf_colombia_dem_extent['z_diff'] = gdf_colombia_dem_extent['h_te_best_fit'] - gdf_colombia_dem_extent['dem_z']
fig,ax = plt.subplots(figsize=(5,4))
# Sort elevation values
gdf_colombia_dem_extent.sort_values(by='dem_z',inplace=True)
gdf_colombia_dem_extent_filt = gdf_colombia_dem_extent[gdf_colombia_dem_extent['z_diff']<1000]
ax.scatter(gdf_colombia_dem_extent_filt.dem_z.values,gdf_colombia_dem_extent_filt.z_diff.values,s=1)
ax.set_ylim(-50,50)
ax.set_xlabel('Elevation (TANDEM-X) (m)')
ax.set_ylabel('Elevation difference (m)')
```

- The difference above might be noise or real signal (" the dates of ICESAT-2 footprints are between December to March 2018-2019, while TANDEM-X contains a mosaic of elevations between 2012-2014")
- It's hard to make out anything from the above plot, let's try a box plot

```{code-cell} ipython3
dem_bins = list(np.arange(0,5500,500))
# mask out differences larger than 100 m ?
filt_lim = (-100,100)
mask = (gdf_colombia_dem_extent['z_diff']<=100) & (gdf_colombia_dem_extent['z_diff']>=-100)
gdf_colombia_dem_extent_filt_box = gdf_colombia_dem_extent[mask]
gdf_colombia_dem_extent_filt_box['bin'] = pd.cut(gdf_colombia_dem_extent_filt_box['dem_z'],bins=dem_bins)
fig,ax = plt.subplots(figsize=(5,4))
gdf_colombia_dem_extent_filt_box.boxplot(column='z_diff',by='bin',ax=ax)
ax.set_xlabel('Elevation (TANDEM-X) (m)')
ax.set_xticklabels(dem_bins)
#ax.set_ylabel('Elevation difference (m)')
ax.set_title('')
ax.set_ylabel('ICESat-2 minus TANDEM-X (m)')
#plt.tight_layout()
```

- The x labels in the plot are lower intervals of boxplots, we see that the median differences are close to zero for most elevation ranges with a maximum difference of -10 m. Also, we see a constant negative bias in all the elevation difference. This might be due to a bias present between the 2 sources. This bias maybe due to offset between the 2 datasets which might come down after coregistration.

+++

## Section 3
- Application of ICESat-2 as control surface for DEMs coregistration
- Or, to find offsets and align ICESat-2 tracks to a control surface

+++

## Going fancy, include only if you want to :)

+++

### Application of ICESat-2 as control for DEM co-registration ?
- Can use point cloud alignment techniques to align DEMs to points, for now as a starting point we can use the transformation matrix to inform on the horizontal and vertical offset between ICESat-2 tracks and DEMs
- We will be using a flavor of Iterative Closest Point alignment algorithm, implemented in [Ames Stereo Pipeline](https://github.com/NeoGeographyToolkit/StereoPipeline)

```{code-cell} ipython3
gdf_colombia_dem_extent.keys()
```

```{code-cell} ipython3
### Save the geodataframe in the specified way as expected by Ames Stereo Pipeline
icesat2_pc = '/home/jovyan/icesat2/icesat2_colombia_pc.csv' 
gdf_colombia_dem_extent[['latitude','longitude','h_te_best_fit']].to_csv(icesat2_pc,header=False,index=None)
```

```{code-cell} ipython3
gdf_colombia_dem_extent.head()
```

```{code-cell} ipython3
### Save the geodataframe in the specified way as expected by Ames Stereo Pipeline
icesat2_pc = '/home/jovyan/icesat2/icesat2_colombia_pc.csv'
pc_rename_dict = {'latitude':'lat','longitude':'lon','h_te_best_fit':'height_above_datum'}
gdf_colombia_dem_extent = gdf_colombia_dem_extent.rename(columns=pc_rename_dict)
#gdf_colombia_dem_extent['height_above_datum'] = gdf_colombia_dem_extent['h_te_best_fit']

gdf_colombia_dem_extent[['lon','lat','height_above_datum']].to_csv(icesat2_pc,header=True,index=None)
```

```{code-cell} ipython3
!exportPATH="/home/jovyan/icesat2/StereoPipeline/bin:$PATH"
```

```{code-cell} ipython3
! ls
```

```{code-cell} ipython3
align_fol = '/home/jovyan/icesat2/align/run'
#max-displacement is set to 10, given ICESat-2 reported operational accuracy
pc_align_opts="--csv-format '1:lon 2:lat 3:height_above_datum' --max-displacement 10 --save-transformed-source-points --alignment-method point-to-point --datum WGS84"
!/home/jovyan/icesat2/StereoPipeline/bin/pc_align $pc_align_opts $icesat2_pc $dem_file -o $align_fol
```

- Alignment results suggest that there is an offset of ~5.4 m between the ICESat-2 points and TANDEM-X DEM, so that could have contributed to the offsets which we see above

```{code-cell} ipython3
##Lets rerun the analysis with the new DEM to see if the alignment improved anything or not
## Regrid the transformed pointcloud into DEM at 90 m posting
!/home/jovyan/icesat2/StereoPipeline/bin/point2dem --tr 90 --t_srs EPSG:32618 $align_fol-trans_source.tif
```

```{code-cell} ipython3
gdf_colombia_dem_extent = gdf_colombia_dem_extent.loc[:,~gdf_colombia_dem_extent.columns.duplicated()]
```

```{code-cell} ipython3
gdf_colombia_dem_extent['height_above_datum'].values[5]
```

```{code-cell} ipython3
trans_dem_file = '/home/jovyan/icesat2/align/run-trans_source-DEM.tif'
trans_dem_ds = rasterio.open(trans_dem_file)
del_time,elev = gda_lib.sample_near_nbor(trans_dem_ds,gdf_colombia_dem_extent)
gdf_colombia_dem_extent['trans_dem_z'] = elev
dem_bins = list(np.arange(0,5500,500))
# mask out differences larger than 100 m ?
filt_lim = (-100,100)
gdf_colombia_dem_extent['trans_z_diff'] = gdf_colombia_dem_extent.height_above_datum - gdf_colombia_dem_extent.trans_dem_z

mask = (gdf_colombia_dem_extent['trans_z_diff']<=100) & (gdf_colombia_dem_extent['trans_z_diff']>=-100)
gdf_colombia_dem_extent_filt_box = gdf_colombia_dem_extent[mask]
gdf_colombia_dem_extent_filt_box['bin'] = pd.cut(gdf_colombia_dem_extent_filt_box['dem_z'],bins=dem_bins)
fig,ax = plt.subplots(figsize=(5,4))
gdf_colombia_dem_extent_filt_box.boxplot(column='trans_z_diff',by='bin',ax=ax)
ax.set_xlabel('Elevation (TANDEM-X) (m)')
ax.set_xticklabels(dem_bins)
ax.set_title('')
ax.set_ylabel('ICESat-2 minus TANDEM-X DEM after coregistration (m)')

```

- We see that after coregistration, the bias reduces to an extent. Note that this is a very preliminary analysis, results will be better after filtering the ATL08 points based on quality metrics and finding truly static surfaces (snow free during acquisition time of ICESat-2 points)

+++

#### Credits
* notebook by: [Jessica Scheick](https://github.com/JessicaS11) and [Shashank Bhushan](https://github.com/ShashankBice)
