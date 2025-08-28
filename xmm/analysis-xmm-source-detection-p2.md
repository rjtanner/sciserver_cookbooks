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

# Source Detection with `edetect_chain` -- Part 2
<hr style="border: 2px solid #fadbac" />

- **Description:** Using `edetect_chain` to automatically detect sources.
- **Level:** Intermediate
- **Data:** XMM observation of the Lockman Hole (obsid=0123700101)
- **Requirements:** Must be run using the `HEASARCv6.35` (or higher) image, along with pySAS version 2.0. Run in the <tt>(xmmsas)</tt> conda environment on Sciserver. You should see <tt>(xmmsas)</tt> at the top right of the notebook. If not, click there and select <tt>(xmmsas)</tt>.
- **Credit:** Ryan Tanner (August 2025)
- **Support:** <a href="https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html">XMM Newton GOF Helpdesk</a>
- **Last verified to run:** 22 August 2025, for SAS v22.1 and pySAS v2.0

<hr style="border: 2px solid #fadbac" />


## 1. Introduction
This tutorial is the second part of automatic source detection using `edetect_chain`. This tutorial assumes you have already worked through [Part 1](./analysis-xmm-source-detection-p1.ipynb) and any prerequisite notebooks.

Here we show how to simultaneously search for sources in all three EPIC cameras across 5 images (for each camera for a total of 15 images) extracted in the 0.2-0.5 keV, 0.5-1 keV, 1-2 keV, 2-4.5 keV, and 4.5-12 keV energy bands, respectively.

#### SAS Tasks to be Used

- `espfilt`[(Documentation for espfilt)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/espfilt/index.html)
- `edetect_chain`[(Documentation for edetect_chain)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/edetect_chain/index.html)

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

```python
def display_fits_image(image_file):
    with fits.open(image_file) as hdu:
        my_js9.SetFITS(hdu)
    my_js9.SetColormap('heat',1,0.5)
    my_js9.SetScale("log")
    my_js9.SetZoom(0.5)
```

```python
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
```

```python
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

```python
# Function to make regions and load into JS9
def make_regions(source_list):
    #my_js9.RemoveRegions('all')
    with fits.open(source_list) as hdu:
        data = hdu[1].data[hdu[1].data['ID_INST'] == 0]
    for i in range(len(data)):
        my_js9.AddRegions("circle", {'ra': data['RA'][i], 'dec': data['DEC'][i], 'radius': 10.0})
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
pimax = 12000

# Putting filenames into a dictionary to make things easier later on
clean_event_lists = {}

for event_list in cevent_lists:
    with fits.open(event_list) as hdu:
        instrument = hdu[0].header['INSTRUME']
    clean_event_lists[instrument] = f'clean_event_list_{instrument}.fits'
    filter_event_list(event_list,pimin,pimax,clean_event_lists[instrument])
```

## 3. Make High Res Images for Analysis

Now we will create 18 high resolution images, 6 for each EPIC camera (1 full band image + 5 narrower band images).

```python
# List of the min energy for each energy band
pi_min_list = [200,200,500,1000,2000,4500]
# List of the max energy for each energy band
pi_max_list = [12000,500,1000,2000,4500,12000]
# List of the names of each energy band
energy_bands = ['full','b1','b2','b3','b4','b5']
```

```python
band_images = {}

# Loop over the three unfiltered event lists for the pn, MOS1, and MOS2
for inst in clean_event_lists.keys():
    band_images[inst] = []
    # Loop over all six energy bands
    for i,band in enumerate(energy_bands):
        image_file = inst+'_image_'+band+'.fits'
        band_images[inst].append(image_file)
        make_hires_image(clean_event_lists[inst],pi_min_list[i],pi_max_list[i],out_image=image_file)
```

Here we display *only* the full band images to check for quality.
<div class="alert alert-block alert-info">
    <b>Note:</b> To switch between images in JS9 click on the 'File' menu in the top left of the JS9 window. The various images currently loaded into JS9 will appear in the 'images' submenu with the default names 'Anonymous1', 'Anonymous2', 'Anonymous3', etc. Select an image to view it.
</div>

```python
for inst in band_images.keys():
    display_fits_image(band_images[inst][0])
```

## 4. Run `edetect_chain`

We will now run `edetect_chain` two different ways.

First we will run it individually for each instrument. This takes the 5 image files created for each energy band.

Next we will run `edetect_chain` over all 5 bands and all 3 EPIC cameras simultaneously. This takes as an input 15 different image files.

```python
# Function to run edetect_chain
def run_edetect_chain(imagesets,filtered_event_list,attitude_file,eml_list,eboxl_list,eboxm_list,
                      pimin,pimax,efc):
    inargs = {'imagesets'   : imagesets, 
              'eventsets'   : filtered_event_list,
              'attitudeset' : attitude_file, 
              'eml_list'    : eml_list,
              'eboxl_list'  : eboxl_list,
              'eboxm_list'  : eboxm_list,
              'pimin'       : pimin, 
              'pimax'       : pimax,
              'ecf'         : efc,
              'esp_nsplinenodes' : 16,
              'esen_mlmin'       : 15}
    
    MyTask('edetect_chain', inargs).run()
```

Everything in the next cell is to get the inputs for `edetect_chain` into their proper format. We will effectively be running the following three commands:

```
edetect_chain imagesets=' \
  "m1_image_b1.fits" "m1_image_b2.fits" "m1_image_b3.fits" "m1_image_b4.fits" "m1_image_b5.fits"' \
  eventsets=mos1S001-allevc.fits attitudeset=0070_0123700101_AttHk.ds \
  pimin='200 500 1000 2000 4500' pimax='500 1000 2000 4500 12000' \
  ecf='1.734 1.746 2.041 0.737 0.145' \
  eboxl_list='mos1_eboxlist_l.fits' eboxm_list='mos1_eboxlist_m.fits' \
  esp_nsplinenodes=16 eml_list='mos1_emllist.fits' esen_mlmin=15

edetect_chain imagesets=' \
  "m2_image_b1.fits" "m2_image_b2.fits" "m2_image_b3.fits" "m2_image_b4.fits" "m2_image_b5.fits"' \
  eventsets=mos2S002-allevc.fits attitudeset=0070_0123700101_AttHk.ds \
  pimin='200 500 1000 2000 4500' pimax='500 1000 2000 4500 12000' \
  ecf='0.991 1.387 1.789 0.703 0.150' \
  eboxl_list='mos2_eboxlist_l.fits' eboxm_list='mos2_eboxlist_m.fits' \
  esp_nsplinenodes=16 eml_list='mos2_emllist.fits' esen_mlmin=15

edetect_chain imagesets=' \
  "pn_image_b1.fits" "pn_image_b2.fits" "pn_image_b3.fits" "pn_image_b4.fits" "pn_image_b5.fits"' \
  eventsets=pnS003-allevc.fits attitudeset=0070_0123700101_AttHk.ds \
  pimin='200 500 1000 2000 4500' pimax='500 1000 2000 4500 12000' \
  ecf='9.525 8.121 5.867 1.953 0.578' \
  eboxl_list='pn_eboxlist_l.fits' eboxm_list='pn_eboxlist_m.fits' \
  esp_nsplinenodes=16 eml_list='pn_emllist.fits' esen_mlmin=15
```

```python
pi_min = []
pi_max = []

for pimin in pi_min_list[1:]: pi_min.append(f'{pimin}')
for pimax in pi_max_list[1:]: pi_max.append(f'{pimax}')
pi_min = " ".join(pi_min)
pi_max = " ".join(pi_max)

efcs = {'EMOS1' : '1.734 1.746 2.041 0.737 0.145',
        'EMOS2' : '0.991 1.387 1.789 0.703 0.150',
        'EPN'   : '9.525 8.121 5.867 1.953 0.578'}

imagesets = {}

for inst in band_images.keys():
    imagesets[inst] = []
    for file in band_images[inst][1:]:
        imagesets[inst].append(f'"{file}"')
    imagesets[inst] = " ".join(imagesets[inst])
```

The cell above creates dictionaries for the various `edetect_chain` input arguments where the dictionary keys are the names of the three instruments `mos1`, `mos2`, and `pn`. The next cell will display the input values created in the cell above.

```python
eml_list = {}

for inst in band_images.keys():
    eml_list[inst] = inst+'_emllist.fits'
    eboxl_list     = inst+'_eboxlist_l.fits'
    eboxm_list     = inst+'_eboxlist_m.fits'
    print(f'Input arguments for {inst}:\n')
    print(f'imageset    = \'{imagesets[inst]}\'')
    print(f'eventsets   = \'{clean_event_lists[inst]}\'')
    print(f'attitudeset = \'{attitude_file}\'')
    print(f'eml_list    = \'{eml_list[inst]}\'')
    print(f'eboxl_list  = \'{eboxl_list}\'')
    print(f'eboxm_list  = \'{eboxm_list}\'')
    print(f'pimin       = \'{pi_min}\'')
    print(f'pimax       = \'{pi_max}\'')
    print(f'ecf         = \'{efcs[inst]}\'\n')
```

<div class="alert alert-block alert-info">
<b>Note:</b> The next cell will run <tt>edetect_chain</tt> three times. In total it will take ~15 minutes to run.
</div>

```python
eml_list = {}

for inst in band_images.keys():
    eml_list[inst] = inst+'_emllist.fits'
    eboxl_list     = inst+'_eboxlist_l.fits'
    eboxm_list     = inst+'_eboxlist_m.fits'
    run_edetect_chain(imagesets[inst],
                      clean_event_lists[inst],
                      attitude_file,
                      eml_list[inst],
                      eboxl_list,
                      eboxm_list,
                      pi_min,
                      pi_max,
                      efcs[inst])
```

Running the following cell will plot the full band images again and overlay the sources found by `edetect_chain`.

```python
for inst in clean_event_lists.keys():
    display_fits_image(band_images[inst][0])
    make_regions(eml_list[inst])
```

## 5. Run Combined `edetect_chain` With All Instruments

Alternatively you can run `edetect_chain` once for all three instruments combined by inputing all the image files and necessary information together. Effectively this is like running the following command:
```
edetect_chain imagesets=' \
"m1_image_b1.fits" "m1_image_b2.fits" "m1_image_b3.fits" "m1_image_b4.fits" "m1_image_b5.fits" \
"m2_image_b1.fits" "m2_image_b2.fits" "m2_image_b3.fits" "m2_image_b4.fits" "m2_image_b5.fits" \
"pn_image_b1.fits" "pn_image_b2.fits" "pn_image_b3.fits" "pn_image_b4.fits" "pn_image_b5.fits"' \
  eventsets='MOS1clean.fits MOS2clean.fits PNclean.fits' attitudeset=AttHk.ds \
  pimin='200 500 1000 2000 4500 200 500 1000 2000 4500 200 500 1000 2000 4500' \
  pimax='500 1000 2000 4500 12000 500 1000 2000 4500 12000 500 1000 2000 4500 12000' \
  ecf='1.734 1.746 2.041 0.737 0.145 0.991 1.387 1.789 0.703 0.150 9.525 8.121 5.867 1.953 0.578' \
  eboxl_list='all_eboxlist_l.fits' eboxm_list='all_eboxlist_m.fits' \
  esp_nsplinenodes=16 eml_list='all_emllist.fits' esen_mlmin=15
```
First we need to arrange the inputs like before, but now combining them all together.

```python
all_eml_list   = 'all_emllist.fits'
all_eboxl_list = 'all_eboxlist_l.fits'
all_eboxm_list = 'all_eboxlist_m.fits'

all_imagesets = []
all_clean_event_lists = []
all_pi_min = []
all_pi_max = []
all_efcs   = []

for inst in band_images.keys():
    all_imagesets.append(imagesets[inst])
    all_clean_event_lists.append(clean_event_lists[inst])
    all_pi_min.append(pi_min)
    all_pi_max.append(pi_max)
    all_efcs.append(efcs[inst])

all_imagesets = " ".join(all_imagesets)
all_clean_event_lists = " ".join(all_clean_event_lists)
all_pi_min = " ".join(all_pi_min)
all_pi_max = " ".join(all_pi_max)
all_efcs   = " ".join(all_efcs)
```

```python
print(f'Input arguments for combined edetect_chain:\n')
print(f'imageset    = \'{all_imagesets}\'')
print(f'eventsets   = \'{all_clean_event_lists}\'')
print(f'attitudeset = \'{attitude_file}\'')
print(f'eml_list    = \'{all_eml_list}\'')
print(f'eboxl_list  = \'{all_eboxl_list}\'')
print(f'eboxm_list  = \'{all_eboxm_list}\'')
print(f'pimin       = \'{all_pi_min}\'')
print(f'pimax       = \'{all_pi_max}\'')
print(f'ecf         = \'{all_efcs}\'\n')
```

<div class="alert alert-block alert-info">
<b>Note:</b> The next cell will take ~25 minutes to run.
</div>

```python
run_edetect_chain(all_imagesets,
                  all_clean_event_lists,
                  attitude_file,
                  all_eml_list,
                  all_eboxl_list,
                  all_eboxm_list,
                  all_pi_min,
                  all_pi_max,
                  all_efcs)
```

The images from the three EPIC cameras can be merged using the command `emosaic`, which has the inputs:

- imagesets : The full band image files to be merged.
- mosaicedset : The output filename for the mosiac image.

```python
merge_imagesets = []

for inst in clean_event_lists.keys():
    merge_imagesets.append(band_images[inst][0])

merge_imagesets = " ".join(merge_imagesets)
print(f'imagesets = \'{merge_imagesets}\'')
```

```python
mosaicedset = 'mosaic.fits'

inargs = {'imagesets'   : merge_imagesets, 
          'mosaicedset' : mosaicedset}

MyTask('emosaic', inargs).run()
```

```python
display_fits_image(mosaicedset)
make_regions(all_eml_list)
```
