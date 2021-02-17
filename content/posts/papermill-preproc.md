+++
title =  "Grind those brains! Diffusion MRI preprocessing with papermill"
tags = ["python", "fsl", "MRtrix3", "dMRI"]
date = "2021-02-17"
image = "img/papermill.svg"
caption = "A sketch of a papermill-based workflow. Hopefully this approach saves you from [American Chopper-style discussions](https://docs.google.com/presentation/d/1n2RlMdmv1p25Xy5thJUhkKGvjtV-dkAIsUXP-AL4ffI/edit#slide=id.g3d168d2fd3_0_159 'Joel Grus slides')."
+++


## Notebooks: you love them or you hate them

Notebooks are an interesting beast: they have become extremely [_diverse_](https://pg.ucsd.edu/publications/computational-notebooks-design-space_VLHCC-2020.pdf 'The Design Space of Computational Notebooks: An Analysis of 60 Systems in Academia and Industry') and popular, everyone loves them, from declared non-programmers to enthusiastic pythonistas, from Kaggle fans to scientists. Well, that's not exactly true: notebooks are actually one of those things that you end up loving them or strongly feeling _against_ them - with indifference being very rare and usually due to not having interacted with them a lot. Although I tend to the _lovers side_, I can see why people get frustrated with some of their issues, especially the _hidden states_ (for more about issues of notebooks, check the presentation from [Joel Grus](https://joelgrus.com) at JupyterCon 2018 - video [here](https://www.youtube.com/watch?v=7jiPeIFXb6U) and slide deck [here](https://docs.google.com/presentation/d/1n2RlMdmv1p25Xy5thJUhkKGvjtV-dkAIsUXP-AL4ffI/edit#slide=id.g362da58057_0_1)). Despite these issues, even haters admit that the way notebooks allow to _mix and match_ markdown, code and figures can have interesting applications. One of these applications, pioneered at [Netflix](https://netflixtechblog.com/notebook-innovation-591ee3221233 'Netflix Tech Blog'), is using them to process, report and troubleshoot at once (_\*gasp\*_) in production. It may sounds crazy, but if we think about it for a moment it makes sense: if we could avoid the hidden state issues running the notebooks as scripts, we end up with an object that after being executed will contain results, plots and error messages! But how can we do that? With [`papermill`](https://papermill.readthedocs.io/en/latest/ 'Papermill Documentation')!

In this post, I will show how to put together a _parameterized_ notebook for diffusion MRI preprocessing and how to use `papermill` to run the notebook on a given dataset. I will assume that the input data are volumes acquired in anterior-posterior (AP) and posterior-anterior (PA) directions, without fixing any of those as the main direction yet. Using parameters, we will also be able to handle different readout times for the two directions, and an arbitraty order of the gradient values.
The goal is putting together a tool that is still easy to run as a `bash` script would be, but that provides also plots (to check intermediate results) and, in case of errors, the _environment_ itself for easier troubleshooting - basically a script that turns into a report! Both the notebook and the related script are on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/workflows/papermill-preproc). 



## Crushing hidden states: parameterized notebooks

A _parameterized_ notebook is pretty much the same thing as an ordinary notebook, except for handling parameters in specific points, as we will see very shortly. One important advantage though is that they are thought to be run sequentially, avoiding most of the weird out-of-order execution issues. As we will see in the last section, `papermill`, an amazing Python package, takes care both of handling the parameters and executing them.

Let's start by importing the packages we will need (`nibabel` and `matplotlib`) and by defining a couple of tailored plotting functions:

```

import nibabel as nib
import matplotlib.pyplot as plt

def show_mid_slices(image):
    """ Function to display row of image middle slices """
    shape = image.shape
    slices = [image[int(shape[0]/2), :, :],
             image[:, int(shape[1]/2), :],
             image[:, :, int(shape[2]/2)]]
    
    fig, axes = plt.subplots(1, len(slices), figsize=(500,200))
    for i, slice in enumerate(slices):
        axes[i].imshow(slice.T, cmap="gray", origin="lower")

        
def show_vol_slices(image):
    """ Function to display slices from several volumes """
    shape = image.shape
    vols = shape[3]
    slices = image[:, :, int(shape[2]/2), :]
    
    fig, axes = plt.subplots(int(vols/5)+1, 5, figsize=(50,20*(int(vols/5)+1)))
    print((500,200*(int(vols/5)+1)))
    for i in range(vols):
        if vols > 5:
            axes[int(i/5),i%5].imshow(slices[:,:,i].T, cmap="gray", origin="lower")
        else:
            axes[i].imshow(slices[:,:,i].T, cmap="gray", origin="lower")
    
```

The first function, slightly adapted from a similar one provided in the [`nibabel` tutorials](https://nipy.org/nibabel/coordinate_systems.html 'Nibabel'), gives an idea of how a volume looks like showing the middle slices in each of the three views; the second one instead shows a single slice (in this case an axial one) for each volume in a 4D dataset. So far, there's nothing different from a normal notebook. Let's see what's next:


```

# pa and ap parameters are mandatory
pa = ""
ap = ""
# bvec and bval need to be specified only for the main encoding direction
# and only if the basename is different from the nifti file
bvec = ""
bval = ""
# the main encoding direction (PA or AP)
main_dir = "PA"
# the readout may be omitted if it is the same for both directions
readout_pa = 0.1
readout_ap = 0.1
# the position of the actual b0 volumes
pa_b0 = 0
ap_b0 = 0

```

Here is the important part for `papermill`: in this cell we define the parameters for the notebook, the ones that we can change when running `papermill`. To make clear that this is the cell of the parameters, we will need to add the tag `parameters`
(to show the tags of each cell, we need to enable them by going through the menu: `View->Cell Toolbar->Tags`). For the mandatory parameters (the actual diffusion data), the default value is an empty string - this will make the notebook failing immediately if those are not provided. For the rest of them, it may not be necessary to change them when running `papermill` - it depends on the data. We now need to handle the details about the main encoding direction of the data and the gradient directions and values:

```
if main_dir == "PA":
    data = pa
    ix = 1
else:
    data = ap
    ix = 2
    
if bvec == "":
    bvec = '/'.join([*data.split('/')[:-1], '']) + data.split('/')[-1].split('.')[0] + '.bvec'
    bval = '/'.join([*data.split('/')[:-1], '']) + data.split('/')[-1].split('.')[0] + '.bval'
    
```

First, we are keeping track of which data are related to the main direction (default is `PA`), and towards the end of the notebook we will take advantage of that. Then we are playing a bit with strings to allow us some additional flexibility: in this way, if the gradient data share the basename with the diffusion data, we don't have to specify those when we call `papermill`.

Now that we _parameterized_ the notebook, we can continue with the visualization and the preprocessing. Let's start with having a look at a couple of b0 volumes from the original data:

```

pa_img = nib.load(pa)
pa_img_data = pa_img.get_fdata()

show_mid_slices(pa_img_data[:,:,:,pa_b0]) 

ap_img = nib.load(ap)
ap_img_data = ap_img.get_fdata()

show_mid_slices(ap_img_data[:,:,:,ap_b0])

```

We can now proceed with the first preprocessing steps:


```

%%bash -s "$pa" "$ap"
dwidenoise $1 PA_denoised.nii
mrdegibbs PA_denoised.nii PA_unringed.nii
dwidenoise $2 AP_denoised.nii
mrdegibbs AP_denoised.nii AP_unringed.nii

```

Here we are using `dwidenoise` and `mrdegibbs` from [MRtrix3](https://mrtrix.readthedocs.io/en/latest/ 'MRtrix3 Documentation'). The easiest way to use these tools is directly from the shell, so I used one of the Jupyter _magic_ tricks to write directly some `bash` lines, sending over the values from a couple of variables defined on the Python side. At this point, we can visualize again how things look if we want. After that, we can go with `topup` from [FSL](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/ 'FSL website'):

```

%%bash -s "$pa_b0" "$ap_b0" "$readout_pa" "$readout_ap"
fslroi PA_unringed.nii b0_blip_up.nii.gz $1 1
fslroi AP_unringed.nii b0_blip_down.nii.gz $2 1
fslmerge -t b0_blip_up_down.nii.gz b0_blip_up.nii.gz b0_blip_down.nii.gz

printf "0 1 0 $3\n0 -1 0 $4" > params.txt

topup --imain=b0_blip_up_down --datain=params.txt --config=b02b0.cnf --out=topup_results --iout=hifi

fslmaths hifi -Tmean hifi
bet hifi hifi_brain -m

```

Once again, it is a `bash` snippet with some _magic_. After visualizing the result, the last step left is `eddy` (from [FSL](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/ 'FSL website') again):

```

%%bash -s "$data" "$ix" "$bvec" "$bval"
vols=$(mrinfo $1 | grep "Dimensions" | cut -d 'x' -f 4 | tr -d ' ')
indx=""
for  ((i=1; i<=$vols; i+=1)); do indx="$indx $2"; done
echo $indx > index.txt

eddy --imain=$1 --mask=hifi_brain_mask --acqp=params.txt --index=index.txt \
    --bvecs=$3 --bvals=$4 --topup=topup_results --out=eddy_corrected --data_is_shelled

```

The important thing to notice here is that we are running `eddy` on the data acquired along the main direction, and it is easy to know which one is it because of the if-else statement at the beginning. We are done with the pre-processing! The very last step is visualizing the final result:

```

eddy_img = nib.load('eddy_corrected.nii.gz')
eddy_img_data =eddy_img.get_fdata()

show_vol_slices(eddy_img_data)

```


## Turning the mill

So we have a _parameterized_ notebook ready to be run through `papermill`, but how does that work? First, we need some data. For this example, I chose the [MASiVar dataset](https://openneuro.org/datasets/ds003416/versions/1.0.0 'MASiVar dataset') shared by Leon Cai and colleagues on [OpenNeuro](https://openneuro.org 'OpenNeuro'), which includes non-preprocessed diffusion data for several subjects and several sessions (and several scanners!). The dataset can be easily downloaded through [DataLad](https://www.datalad.org 'DataLad'):

```
datalad install https://github.com/OpenNeuroDatasets/ds003416.git
datalad get -J 4 ds003416/sub-cIIIsA01/ses-s1Bx1/*

```

As usual, to make things easier let's copy the data we want to process in a dedicated folder:

```
mkdir sample_data
cp ds003416/sub-cIIIsA01/ses-s1Bx1/dwi/*104* sample_data/
cp ds003416/sub-cIIIsA01/ses-s1Bx1/dwi/*105* sample_data/
mkdir results
cd results

```

We have everything we need. It's `papermill` time!

```
papermill ../dwi_preproc.ipynb \
    -p ap ../sample_data/sub-cIIIsA01_ses-s1Bx1_acq-b1000n3r21x21x22peAPA_run-104_dwi.nii.gz \
    -p pa ../sample_data/sub-cIIIsA01_ses-s1Bx1_acq-b1000n40r21x21x22peAPP_run-105_dwi.nii.gz \
    -p ap_b0 3 results.ipynb
```

Let's have a look at the _anatomy_ of a `papermill` call: first, we pass the notebook we created; then, we pass all the parameters we want to change - in this case, in addition to the data filenames, I also speicify the position of the b0 volume for the AP dataset (we don't need to for the PA one, since it is at the default position 0); finally, we pass another notebook filename - the one that will be run with the specified parameters.

The final touch: if we are going to preprocess a lot of data, we probably don't want to open Jupyter and have a look at all the notebooks. To have a handy way to double-check that everything looks ok, we can convert the resulting notebook to HTML - creating our very own pre-processing report!

```
jupyter nbconvert results.ipynb --no-input --to html

```


## Useful references

* [Papermill](https://papermill.readthedocs.io/en/latest/ 'Papermill Documentation')
* [How notebooks are used at Netflix](https://netflixtechblog.com/notebook-innovation-591ee3221233 'Netflix Tech Blog')
* [MASiVar dataset](https://openneuro.org/datasets/ds003416/versions/1.0.0 'OpenNeuro')
* ["MASiVar: Multisite, Multiscanner, and Multisubject Acquisitions for Studying Variability in Diffusion Weighted Magnetic Resonance Imaging"](https://www.biorxiv.org/content/10.1101/2020.12.03.408567v1 'bioRxiv')