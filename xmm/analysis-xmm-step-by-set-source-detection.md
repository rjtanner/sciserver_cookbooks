---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.0
  kernelspec:
    display_name: (xmmsas)
    language: python
    name: conda-env-xmmsas-py
---

# EPIC Source Finding Thread: Step by Step
<hr style="border: 2px solid #fadbac" />

- **Description:** A step-by-step recipe to run the source detection chain (`edetect_chain`) in SAS.
- **Level:** Intermediate
- **Data:** XMM observation of the Lockman Hole (obsid=0123700101)
- **Requirements:** Must be run using the `HEASARCv6.35` (or higher) image, along with pySAS version 2.0. Run in the <tt>(xmmsas)</tt> conda environment on Sciserver. You should see <tt>(xmmsas)</tt> at the top right of the notebook. If not, click there and select <tt>(xmmsas)</tt>.
- **Credit:** Ryan Tanner (August 2025)
- **Support:** <a href="https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html">XMM Newton GOF Helpdesk</a>
- **Last verified to run:** 22 August 2025, for SAS v22.1 and pySAS v2.0

<hr style="border: 2px solid #fadbac" />


## 1. Introduction
This thread is a step-by-step recipe to run the source detection chain in SAS. It shows how to run all the individual SAS tasks constituting the source detection meta-task `edetect_chain`. This is based on [an analysis thread](https://www.cosmos.esa.int/web/xmm-newton/sas-thread-src-find-stepbystep) provided by the SOC at ESA.

#### SAS Tasks to be Used

- `eboxdetect`[(Documentation for eboxdetect)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/eboxdetect/index.html)
- `eexpmap`[(Documentation for eexpmap)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/eexpmap/index.html)
- `emask`[(Documentation for emask)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/emask/index.html)
- `emldetect`[(Documentation for edetect_chain)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/emldetect/index.html)
- `esensmap`[(Documentation for edetect_chain)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/esensmap/index.html)
- `esplinemap`[(Documentation for esplinemap)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/esplinemap/index.html)
- `evselect`[(Documentation for evselect)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/evselect/index.html)

#### Useful Links

- [`pysas` Documentation](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/pysas/index.html "pysas Documentation")
- [`pysas` on GitHub](https://github.com/XMMGOF/pysas)
- [Common SAS Threads](https://www.cosmos.esa.int/web/xmm-newton/sas-threads/ "SAS Threads")
- [Users' Guide to the XMM-Newton Science Analysis System (SAS)](https://xmm-tools.cosmos.esa.int/external/xmm_user_support/documentation/sas_usg/USG/SASUSG.html "Users' Guide")
- [The XMM-Newton ABC Guide](https://heasarc.gsfc.nasa.gov/docs/xmm/abc/ "ABC Guide")
- [XMM Newton GOF Helpdesk](https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html "Helpdesk") - Link to form to contact the GOF Helpdesk.

<div style='color: #333; background: #ffffdf; padding:20px; border: 4px solid #fadbac'>
<b>Running On Sciserver:</b><br>
When running this notebook inside Sciserver, make sure the HEASARC data drive is mounted when initializing the Sciserver compute container. <a href='https://heasarc.gsfc.nasa.gov/docs/sciserver/'>See details here</a>.
<br><br>
<b>Running Outside Sciserver:</b><br>
This notebook was designed to run on SciServer, but an equivelent notebook <a href="https://github.com/XMMGOF/pysas_docs">can be found on GitHub</a>. You will need to install the development version of pySAS found on GitHub (<a href="https://github.com/XMMGOF/pysas">pySAS on GitHub</a>). There are installation instructions on GitHub. 
<br>
</div>

<div class="alert alert-block alert-warning">
    <b>Warning:</b> By default this notebook will place observation data files in your <tt>scratch</tt> space. The <tt>scratch</tt> space on SciServer will only retain files for 90 days. If you wish to keep the data files for longer move them into your <tt>persistent</tt> directory.
</div>

```python
# Make sure pySAS is up to date
%pip install xmmpysas --upgrade
```

```python
# pySAS imports
import pysas
from pysas.sastask import MyTask

# Importing Js9
import jpyjs9
my_js9 = jpyjs9.JS9(width = 800, height = 800, side=True)

# Useful imports
import os, glob

# Astropy import
from astropy.io import fits

# Environment variables that need to be set to avoid a problem with edetect_chain on SciServer
os.environ['HEADASNOQUERY'] = ''
os.environ['HEADASPROMPT']  = '/dev/null'
```

```python
obsid = '0123700101'

# To get your user name. Or you can just put your user name in the path for your data.
from SciServer import Authentication as auth
usr = auth.getKeystoneUserWithToken(auth.getToken()).userName

data_dir = os.path.join('/home/idies/workspace/Temporary/',usr,'scratch/xmm_data')
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
my_obs.basic_setup(overwrite=False,repo='sciserver',rerun=False,run_rgsproc=False)
os.chdir(my_obs.work_dir)
```

```python
event_lists = []
event_lists.append(my_obs.files['M1evt_list'][0])
event_lists.append(my_obs.files['M2evt_list'][0])
event_lists.append(my_obs.files['PNevt_list'][0])
```

---
These next cell contains functions that will be used throughout this notebook.

```python
def display_fits_image(image_file):
    with fits.open(image_file) as hdu:
        my_js9.SetFITS(hdu)
    my_js9.SetColormap('heat',1,0.5)
    my_js9.SetScale("log")
    my_js9.SetZoom(0.5)

# Function to make regions and load into JS9
def make_regions(source_list):
    #my_js9.RemoveRegions('all')
    with fits.open(source_list) as hdu:
        data = hdu[1].data[hdu[1].data['ID_INST'] == 0]
    for i in range(len(data)):
        my_js9.AddRegions("circle", {'ra': data['RA'][i], 'dec': data['DEC'][i], 'radius': 10.0})
    
def filter_event_list(in_event_list,
                      pi_min,
                      pi_max,
                      filtered_event_list):

    with fits.open(in_event_list) as hdu:
        instrument = hdu[0].header['INSTRUME']

    if instrument == 'EPN':
        filter = 'XMMEA_EP'
        pattern = 0
    elif 'EMOS' in instrument:
        filter = 'XMMEA_EM'
        pattern = 12

    # Filter expression
    expression = '(PATTERN in [0:{pattern}])&&(PI in [{pi_min}:{pi_max}])&&(FLAG == 0)&&#{filter}'.format(filter=filter,pattern=pattern,pi_min=pi_min,pi_max=pi_max)

    inargs = {'table'           : in_event_list, 
              'withfilteredset' : 'yes', 
              "expression"      : expression, 
              'filteredset'     : filtered_event_list, 
              'filtertype'      : 'expression', 
              'keepfilteroutput': 'yes', 
              'updateexposure'  : 'yes', 
              'filterexposure'  : 'yes'}
    
    MyTask('evselect', inargs).run()

def make_hires_image(in_event_list,
                     pi_min,
                     pi_max,
                     out_image='image.fits'):

    # Filter expression
    expression = '(PI in [{pi_min}:{pi_max}])'.format(pi_min=pi_min,pi_max=pi_max)

    inargs = {'table'         : in_event_list+':EVENTS', 
              'withimageset'  : 'yes',
              "expression"    : expression, 
              'imageset'      : out_image,
              'imagebinning'  : 'binSize',
              'xcolumn'       : 'X',
              'ycolumn'       : 'Y',
              'ximagebinsize' : 40,
              'yimagebinsize' : 40}
    
    MyTask('evselect', inargs).run()

    return out_image
```

## 2. Filter the Observation

To filter this observation we will use the command [`espfilt`](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/espfilt/espfilt.html) which can do a lot of automatic filtering of solar flare activity. This task outputs a number of files that include (with the corresponding filename format):

    - Filtered event list (*-allevc.fits)
    - Filtered image file (*-allimc.fits)
    - Lightcurve file (*-fovlc.fits)
    - UNfiltered event list for the corners (*-corev.fits)
    - Filtered event list for the corners (*-corevc.fits)
    - Filtered image file for the corners (*-corimc.fits)
    - Lightcurve file for the corners (*-corlc.fits)
    - Post-filtering Good Time Interval (GTI) file (*-gti.fits)

The filenames will start with {instrument}{exposure ID}. For example, mos1S001, mos2S002, pnS003 etc.

```python
for file in event_lists:
    inargs = {'eventfile' : file}
    MyTask('espfilt', inargs).run()
```

After this we will apply some standard filters to clean up the event lists.

```python
cevent_lists = glob.glob('*allevc.fits')
attitude_file = glob.glob('*AttHk.ds')[0]

pimin = 200
pimax = 10000

# Putting filenames into a dictionary to make things easier later on
clean_event_lists = {}

for event_list in cevent_lists:
    with fits.open(event_list) as hdu:
        instrument = hdu[0].header['INSTRUME']
    clean_event_lists[instrument] = f'clean_event_list_{instrument}.fits'
    filter_event_list(event_list,pimin,pimax,clean_event_lists[instrument])
```

## 3. Make High Res Images for Analysis

For completeness we will create 3 high resolution images, one for each EPIC camera. In the example below we will only be using the image for the MOS1 camera, but at the end we will have a loop to run over all three images.

```python
image_file = {}

# Loop over the three unfiltered event lists for the pn, MOS1, and MOS2
for inst in clean_event_lists.keys():
    image_file[inst] = inst+'_image.fits'
    make_hires_image(clean_event_lists[inst],pimin,pimax,out_image=image_file[inst])
```

Here we display the images to check for quality.
<div class="alert alert-block alert-info">
    <b>Note:</b> To switch between images in JS9 click on the 'File' menu in the top left of the JS9 window. The various images currently loaded into JS9 will appear in the 'images' submenu with the default names 'Anonymous1', 'Anonymous2', 'Anonymous3', etc. Select an image to view it.
</div>

```python
for inst in image_file.keys():
    display_fits_image(image_file[inst])
```

## 4. Recreating the Chain in `edetect_chain`

### 4.1 Create an Exposure Map

First we create an exposure map. This is equivelent to running the following SAS command.

```
eexpmap attitudeset=0070_0123700101_AttHk.ds eventset=clean_event_list_EMOS1.fits imageset=EMOS1_image.fits expimageset=EMOS1_expmap.ds pimin="200" pimax="10000"
```

```python
expimagesets = 'EMOS1_expmap.ds'

inargs = {'imageset'    : image_file['EMOS1'], 
          'eventset'    : clean_event_lists['EMOS1'],
          'attitudeset' : attitude_file, 
          'expimageset' : expimagesets,
          'pimin'       : pimin, 
          'pimax'       : pimax}

MyTask('eexpmap', inargs).run()

display_fits_image(expimagesets)
```

### 4.2 Create a Detection Map

Here we will create a detection map (masking the areas of the field of view where source detection shall not not be performed). The equivelent SAS command is:
```
emask expimageset=EMOS1_expmap.ds threshold1=0.25 detmaskset=EMOS1_mask.ds
```
where `threshold1` indicates the maximum exposure fraction for a masked pixel (see the [emask parameters description](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/emask/node4.html)). In this specific example all pixels whose exposure time is lower than 0.25 times the maximum exposure are given value "0" in the mask, and excluded from the area where the source detection is performed. Users may relax this choice after a careful look at the exposure map, if they want to seek sources on a larger area.

```python
detmaskset = 'EMOS1_mask.ds'

inargs = {'expimageset' : expimagesets,
          'detmaskset'  : detmaskset, 
          'threshold1'  : 0.25}

MyTask('emask', inargs).run()

display_fits_image(detmaskset)
```

### 4.3 Perform a Sliding Box Detection

Now we perform a sliding box detection, using the locally estimated background. The equivelent SAS command is,
```
eboxdetect usemap=no likemin=8 withdetmask=yes detmasksets=EMOS1_mask.ds imagesets=EMOS1_image.fits expimagesets=EMOS1_expmap.ds pimin=200 pimax=10000 boxlistset=eboxlist_local.fits
```
where likemin is the [minimum detection likelihood](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/eboxdetect/node3.html) (see the [eboxdetect parameters description](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/eboxdetect/node4.html) as well).

This will create a temporary source list `'eboxlist_local.fits'` which will be used to determine the background.

```python
boxlistset = 'eboxlist_local.fits'

inargs = {'imagesets'    : image_file['EMOS1'],
          'detmasksets'  : detmaskset,
          'expimagesets' : expimagesets,
          'boxlistset'   : boxlistset,
          'usemap'       : False,
          'likemin'      : 8,
          'withdetmask'  : True,
          'pimin'        : 200,
          'pimax'        : 10000} 

MyTask('eboxdetect', inargs).run()
```

### 4.4 Create Background Map

Now we create a background map, after masking the sources detected during the previous step which are stored in the file `'eboxlist_local.fits'`,
```
esplinemap bkgimageset=MOS1_bkg.ds scut=0.005 imageset=MOS1_image_full.fits nsplinenodes=16 withdetmask=yes detmaskset=MOS1_mask.ds withexpimage=yes expimageset=MOS1_expmap.ds boxlistset=eboxlist_local.fits
```
where `nsplinenodes` is the number of nodes employed in the background map interpolation and scut the source surface brightness level (in counts per arcseconds squared), above which a source is masked (see the [`esplinemap` parameters description](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/esplinemap/node4.html)). 

Particular care needs to be used in the choice of `nsplinenodes`. A small number of nodes may fail to accurately reproduce the background spatial distribution; an excessive number of nodes may create faked structures. The optimal compromise depends on the spatial pattern of the X-ray emission (background+sources) in a given field of view. Users are encouraged to try different choices for this parameter, and compare the resulting maps with the observed background spatial distribution pattern. 

The parameter `scut` mainly drives the amount of source contamination to the interpolated background map, due to the broad wings of the Point Spread Function. The best value of `scut` is determined by the optimal balance between background integration area and acceptable source contamination level. This is obviously dependent on the amount of background and the on how crowded the field is. Users are encouraged to check the background maps produced by `esplinemap` for short-scale fluctuations, and tune the `scut` value accordingly.

```python
bkgimageset = 'EMOS1_bkg.ds'

inargs = {'imageset'     : image_file['EMOS1'],
          'detmaskset'   : detmaskset,
          'expimageset'  : expimagesets,
          'boxlistset'   : boxlistset,
          'bkgimageset'  : bkgimageset,
          'scut'         : 0.005,
          'nsplinenodes' : 16,
          'withdetmask'  : True,
          'withexpimage' : True} 

MyTask('esplinemap', inargs).run()
```

### 4.5 Perform a Second Run of Source Detection

Now we perform a second run of source detection, using the background map calculated during the previous step,
```
eboxdetect usemap=yes bkgimagesets=MOS1_bkg.ds likemin=8 withdetmask=yes detmasksets=MOS1_mask.ds imagesets=MOS1_image_full.fits expimagesets=MOS1_expmap.ds pimin=200 pimax=10000 boxlistset=eboxlist_map.fits
```

```python
boxlistset = 'eboxlist_map.fits'

inargs = {'imagesets'    : image_file['EMOS1'],
          'detmasksets'  : detmaskset,
          'expimagesets' : expimagesets,
          'boxlistset'   : boxlistset,
          'bkgimagesets' : bkgimageset,
          'usemap'       : True,
          'likemin'      : 8,
          'withdetmask'  : True,
          'pimin'        : 200,
          'pimax'        : 10000} 

MyTask('eboxdetect', inargs).run()
```

<div class="alert alert-block alert-info">
    <b>Note:</b> The SAS tasks <tt>esplinemap</tt> and <tt>eboxdetect</tt> have very similar input parameters, but they are <i>slightly</i> different. For example, <tt>esplinemap</tt> has the input parameter <tt>imageset</tt> while <tt>eboxdetect</tt> has <tt>imageset<b>s</b></tt>, (note the '<tt>s</tt>'). Many (but not all, e.g. <tt>boxlistset</tt>) of the input parameters for <tt>eboxdetect</tt> have an extra '<tt>s</tt>'.
</div>


### 4.6 Maximum Likelihood Fitting

Next we do a maximum likelihood fitting on the sources detected in the previous step. We do this to, 

- Optimise source centring (constrained to be the same for all the cameras and/or energy bands, if source detection is performed on multiple images)
- Determine source extend by fitting the local Point Spread Function
```
emldetect imagesets=MOS1_image_full.fits expimagesets=MOS1_expmap.ds bkgimagesets=MOS1_bkg.ds boxlistset=eboxlist_map.fits ecf=2.0 mllistset=emllist.fits mlmin=10 determineerrors=yes
```
The `ecf` is the Energy Conversion Factor to convert count rates (counts/s) to fluxes (10-11 erg/s/cm2) in a given energy band (a standard definition is reported in Section 6.2.1 of the [3XMM EPIC Source Catalogue User Guide](http://xmmssc.irap.omp.eu/Catalogue/3XMM-DR6/3XMM_DR6.html)).

```python
mllistset = 'emllist.fits'

inargs = {'imagesets'       : image_file['EMOS1'],
          'expimagesets'    : expimagesets,
          'boxlistset'      : boxlistset,
          'bkgimagesets'    : bkgimageset,
          'mllistset'       : mllistset,
          'determineerrors' : True,
          'mlmin'           : 10,
          'ecf'             : 2.0} 

MyTask('emldetect', inargs).run()
```

### 4.7 Create Sensitivity Maps

The next step is to create sensitivity maps, i.e. a pixel-by-pixel detection upper limit map,
```
esensmap expimagesets=MOS1_expmap.ds bkgimagesets=MOS1_bkg.ds detmasksets=MOS1_mask.ds mlmin=10 sensimageset=MOS1_sens_map.fits
```

```python
sensimageset = 'EMOS1_sens_map.fits'

inargs = {'detmasksets'  : detmaskset,
          'expimagesets' : expimagesets,
          'bkgimagesets' : bkgimageset,
          'sensimageset' : sensimageset,
          'mlmin'        : 10} 

MyTask('esensmap', inargs).run()

display_fits_image(sensimageset)
```

### 4.8 Display the Sources

Display the location of the detected sources on the MOS image,

```python
display_fits_image(image_file['EMOS1'])
make_regions(mllistset)
```

## 5. Loop Over All Three EPIC Cameras

```python
expimagesets = {}
detmaskset   = {}
boxlistset   = {}
bkgimageset  = {}
mllistset    = {}
sensimageset = {}

# Temporary file
boxlistset_local = 'eboxlist_local.fits'

for inst in image_file.keys():
    expimagesets[inst] = f'{inst}_expmap.ds'
    detmaskset[inst]   = f'{inst}_mask.ds'
    boxlistset[inst]   = f'{inst}_eboxlist_map.fits'
    bkgimageset[inst]  = f'{inst}_bkg.ds'
    mllistset[inst]    = f'{inst}_emllist.fits'
    sensimageset[inst] = f'{inst}_sens_map.fits'

    # Create exposure map
    inargs = {'imageset'    : image_file[inst], 
              'eventset'    : clean_event_lists[inst],
              'attitudeset' : attitude_file, 
              'expimageset' : expimagesets[inst],
              'pimin'       : pimin, 
              'pimax'       : pimax}
    
    MyTask('eexpmap', inargs).run()

    # Create detection map
    inargs = {'expimageset' : expimagesets[inst],
              'detmaskset'  : detmaskset[inst], 
              'threshold1'  : 0.25}
    
    MyTask('emask', inargs).run()
    
    # Perform sliding box detection
    # Use the temporary boxlist file 'boxlistset_local'
    inargs = {'imagesets'    : image_file[inst],
              'detmasksets'  : detmaskset[inst],
              'expimagesets' : expimagesets[inst],
              'boxlistset'   : boxlistset_local,
              'usemap'       : False,
              'likemin'      : 8,
              'withdetmask'  : True,
              'pimin'        : 200,
              'pimax'        : 10000} 
    
    MyTask('eboxdetect', inargs).run()

    # Create background map
    inargs = {'imageset'     : image_file[inst],
              'detmaskset'   : detmaskset[inst],
              'expimageset'  : expimagesets[inst],
              'boxlistset'   : boxlistset_local,
              'bkgimageset'  : bkgimageset[inst],
              'scut'         : 0.005,
              'nsplinenodes' : 16,
              'withdetmask'  : True,
              'withexpimage' : True} 
    
    MyTask('esplinemap', inargs).run()

    # Run source detection a second time
    # Use final boxlist files 'boxlistset[inst]'
    inargs = {'imagesets'    : image_file[inst],
              'detmasksets'  : detmaskset[inst],
              'expimagesets' : expimagesets[inst],
              'boxlistset'   : boxlistset[inst],
              'bkgimagesets' : bkgimageset[inst],
              'usemap'       : True,
              'likemin'      : 8,
              'withdetmask'  : True,
              'pimin'        : 200,
              'pimax'        : 10000} 
    
    MyTask('eboxdetect', inargs).run()

    # Maximum likelihood fitting
    inargs = {'imagesets'       : image_file[inst],
              'expimagesets'    : expimagesets[inst],
              'boxlistset'      : boxlistset[inst],
              'bkgimagesets'    : bkgimageset[inst],
              'mllistset'       : mllistset[inst],
              'determineerrors' : True,
              'mlmin'           : 10,
              'ecf'             : 2.0} 
    
    MyTask('emldetect', inargs).run()

    # Create sensitivity maps
    inargs = {'detmasksets'  : detmaskset[inst],
              'expimagesets' : expimagesets[inst],
              'bkgimagesets' : bkgimageset[inst],
              'sensimageset' : sensimageset[inst],
              'mlmin'           : 10} 
    
    MyTask('esensmap', inargs).run()

    # Display sources
    display_fits_image(image_file[inst])
    make_regions(mllistset[inst])
```
