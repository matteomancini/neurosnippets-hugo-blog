+++
title =  "DieSitCom - A minimal Streamlit-based viewer for DICOM data"
tags = ["brainviz", "python", "streamlit", "ITK", "MRI"]
date = "2021-03-17"
image = "img/dcm.gif"
caption = "If you think that both NIFTI and DICOM formats make things too complicated check out the [normal file format](https://xkcd.com/2116/)."
+++


## DICOM another file

There are two kind of (neuroimaging) people in this world: those who like the NIFTI format, and those who lie. Jokes aside, it is quite common for many _neuroimagers_ to deal with the _endpoint_ of the acquisition pipeline, to the point that not everyone has dealt directly with what actually comes out of an MRI scanner, data in [DICOM format](https://towardsdatascience.com/understanding-dicom-bce665e62b72 'Understanding DICOM'). But at some point, that time comes: for one reason or another, `dcm2niix` does not give us everything we need, and we need to go back to the _source_. But do we really want to look for a software to handle DICOM data and install it? Probably not. But there is good news: with [ITK](https://itk.org 'Insight ToolKit') (and specifically [SimpleITK](https://simpleitk.readthedocs.io/en/master/link_DicomSeriesReader_docs.html 'SimpleITK Documentation')), reading data and metadata from a DICOM folder is a matter of just a few lines of code. While we are at it, we may as well make a dashboard out of it!

In this post, I will show how to put together that dashboard using [Streamlit](https://www.streamlit.io 'Streamlit'), so guess what, we are doing that in _50 lines of code_! As in most cases one may be interested in some _obscure_ parameter hidden in the DICOM header, we will make sure that both the images and the metadata are accessible. This example and the related requirements are on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/brainviz/diesitcom).


## Don’t count your DICOMs before they are read

We can recycle some of the code written for the [skull-stripping dashboard](http://neurosnippets.com/posts/diesitcom/#post 'The Skull-Strip Club'). As in that example, we need first to import the relevant packages and define two functions, one for selecting the input and the other for plotting:

```
import os
import matplotlib.pyplot as plt
import pandas as pd
import streamlit as st
import SimpleITK as sitk


def dir_selector(folder_path='.'):
    dirnames = [d for d in os.listdir(folder_path) if os.path.isdir(os.path.join(folder_path, d))]
    selected_folder = st.sidebar.selectbox('Select a folder', dirnames)
    if selected_folder is None:
        return None
    return os.path.join(folder_path, selected_folder)


def plot_slice(vol, slice_ix):
    fig, ax = plt.subplots()
    plt.axis('off')
    selected_slice = vol[slice_ix, :, :]
    ax.imshow(selected_slice, origin='lower', cmap='gray')
    return fig
```

The functions are slightly different from the skull-stripping dashboard: instead of looking for NIFTI files, we are looking for folders (and we will need to handle the case where a given folder is not DICOM-related) once again embedding a dropdown menu within the function; in terms of plotting, this time the function is shorter as we do not need to handle overlays.

The next block of code, that is also the last one (\*_already!?_\*), deals with defining the objects we need and inter-connect them:

```
st.sidebar.title('DieSitCom')
dirname = dir_selector()

if dirname is not None:
    try:
        reader = sitk.ImageSeriesReader()
        dicom_names = reader.GetGDCMSeriesFileNames(dirname)
        reader.SetFileNames(dicom_names)
        reader.LoadPrivateTagsOn()
        reader.MetaDataDictionaryArrayUpdateOn()
        data = reader.Execute()
        img = sitk.GetArrayViewFromImage(data)
    
        n_slices = img.shape[0]
        slice_ix = st.sidebar.slider('Slice', 0, n_slices, int(n_slices/2))
        output = st.sidebar.radio('Output', ['Image', 'Metadata'], index=0)
        if output == 'Image':
            fig = plot_slice(img, slice_ix)
            plot = st.pyplot(fig)
        else:
            metadata = dict()
            for k in reader.GetMetaDataKeys(slice_ix):
                metadata[k] = reader.GetMetaData(slice_ix, k)
            df = pd.DataFrame.from_dict(metadata, orient='index', columns=['Value'])
            st.dataframe(df)
    except RuntimeError:
        st.text('This does not look like a DICOM folder!')
```

Once again, `streamlit` offers a great example of how quick is it to put together a simple but functional dashboard. In less than 30 lines of code, we are doing the following:

* creating all the objects we need - a `title`, a `slider` and a `radio` button (all of these in the `sidebar`), a `pyplot` container for the figure, and finally a `dataframe` container for the metadata;
* when any folder name is selected through the function `dir_selector()`, the DICOM data and metadata are read using `SimpleITK`;
* we use the `slider` to select the slice of interest, and the `radio` button to choose if we are interested in the actual images or in the metadata;
* we embed most of what I just listed in a `try`/`except` block to avoid error messages whenever the dropdown menu ends up on a non-DICOM folder.

Now we have to find some DICOM to test it!


## DICOM Island

Publicly available DICOM data are not easy to find, but we are in luck: on the [`dash-vtk` repository](https://github.com/plotly/dash-vtk 'GitHub repository') (that also provides an amazing example to use Dash for [3D rendering](https://dash-gallery.plotly.host/dash-vtk-explorer/ 'dash-vtk explorer') of DICOM data leveraging [VTK](https://vtk.org 'Visualization Toolkit')) there are some sample DICOM datasets, including a brain. To retrieve the data, we can clone the repository or, to download directly that specific folder, we can use `svn`:


```
svn checkout https://github.com/plotly/dash-vtk/trunk/demos/data/mri_brain
```

It's time for `streamlit`:

```
streamlit run diesitcom.py
```



## Useful references

* [Diving into DICOMs](https://towardsdatascience.com/understanding-dicom-bce665e62b72 'Towards Data Science')
* [Streamlit](https://www.streamlit.io 'Streamlit')
* [Reading DICOM with SimpleITK](https://simpleitk.readthedocs.io/en/master/link_DicomSeriesReader_docs.html 'SimpleITK Documentation')
* [dash-vtk](https://github.com/plotly/dash-vtk 'GitHub repository')
* [3D rendering of DICOM data with dash-vtk](https://dash-gallery.plotly.host/dash-vtk-explorer/ 'dash-vtk explorer')


