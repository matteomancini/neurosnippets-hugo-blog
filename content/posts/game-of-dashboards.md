+++
title =  "Game of Dashboards: one brain, four frameworks [Part 1: Streamlit, Dash]"
tags = ["dataviz", "python", "streamlit", "dash", "fMRI"]
date = "2021-03-31"
image = "img/game_of_dashboards.svg"
caption = "Sure, with so many frameworks available coding is much easier, but be aware of the [Twitter Bot Effect](https://xkcd.com/1646/)."
+++


_This is the first part of the dashboard framework comparison. The second part is [here](https://neurosnippets.com/posts/game-of-dashboards-2/#post)._


## A matter of choice?

I'm quite a fan of dashboards (you couldn't tell right?). I think that they offer an interesting _medium_ with a double potential: useful for actual work, especially quality control and exploratory analyses, but also for presentation purposes - a dashboard can become the actual _final product_ of research. It is not surprising then that there are so many choices in terms of dashboard frameworks. Are they all inter-changeable though? Which one should you pick for _that next thing_? There's only one way to answer this question: make the _same_ dashboard using different frameworks.

In this post (the first of two parts), I will focus on a _brainy_ use case and go through the implementation using different python frameworks. I will highlight their differences and point out advantages and disadvantages. Here I will start showcasing [Streamlit](https://streamlit.io) and [Dash](https://dash.plotly.com 'Dash'). All the related code is available on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/dataviz/game-of-dashboards). 



## Ground rules

For each framework, I will aim to make more or less the same dashboard: an _image-based explorer_ for fMRI data. The dashboard will read the data (using [`nibabel`](https://nipy.org/nibabel/ 'NiBabel')), compute their average over time (using [`numpy`](https://numpy.org 'NumPy')) and display a slice of this average volume. Navigating the volume and the slice through _dedicated components_, one should be able to retrieve the BOLD signal timecourse for a given voxel in a separate plot.

I will be using the data from the [Washington University 120 dataset](https://openneuro.org/datasets/ds000243/versions/00001 'OpenNeuro') shared by Steve Petersen and Brad Schlaggar. As usual, we can easily download the data through [DataLad](https://www.datalad.org 'DataLad'):

```
datalad install https://github.com/OpenNeuroDatasets/ds000243.git
cd ds000243/
datalad get sub-001/func/sub-001_task-rest_run-1_bold.nii.gz
cp sub-001/func/sub-001_task-rest_run-1_bold.nii.gz ../bold.nii.gz
```


## Be quick or be dead: Streamlit

The simplest way to make a dashboard is arguably [Streamlit](https://streamlit.io). I've already used it in previous posts ([here](https://neurosnippets.com/posts/diesitcom/#post) and [here](https://neurosnippets.com/posts/quick-stripper/#post)), and I mentioned several times its greatest advantages: ready-to-use components that can be instantiated in very few lines of code, and the interactions coming out almost of nowhere. It literally feels like _magic_. Is it always the ideal choice though? We are about to find out.

Let's start as usual importing the packages and defining some functions we will need:

```
import nibabel as nib
import numpy as np
import streamlit as st
import matplotlib.pyplot as plt


@st.cache
def load(filename):
    img = nib.load(filename)
    img_data = img.get_fdata()
    mean_vol = np.mean(img_data, axis=3)
    return img_data, mean_vol


def plot_zslice(vol, coord_xy, z):
    fig, ax = plt.subplots()
    ax.imshow(vol[:, :, z].T, origin='lower', cmap='gray')
    ax.scatter([coord_xy[0]], [coord_xy[1]], facecolors='none', edgecolors='r')
    return fig


def plot_tc(vol_4d, coord):
    fig, ax = plt.subplots()
    ax.plot(np.arange(vol_4d.shape[3]), vol_4d[coord[0], coord[1], coord[2], :])
    return fig
```

Apart from the plotting functions that resemble previous examples, you may notice that there's a novel addition: the `load()` function has a decorator! The role of `streamlit.cache` is to keep the results from a given function in the [_cache_](https://docs.streamlit.io/en/stable/caching.html) whenever the input has not changed. Otherwise, the whole thing will be always re-executed every time the user interacts with a component. So far that's the only `streamlit`-related line!

Now that we have both the necessary packages and functions, we can get to the core of the dashboard:

```
filename = 'bold.nii.gz'
vol_4d, mean_vol = load(filename)
vol_size = mean_vol.shape

x = st.sidebar.slider('x', 0, vol_size[0], int(vol_size[0]/2))
y = st.sidebar.slider('y', 0, vol_size[1], int(vol_size[1]/2))
z = st.sidebar.slider('z', 0, vol_size[2], int(vol_size[2]/2))
fig1 = plot_zslice(mean_vol, [x, y], z)
fig2 = plot_tc(vol_4d, [x,y,z])

col1, col2 = st.beta_columns(2)
col1.pyplot(fig1)
col2.pyplot(fig2)
```

Once the data are loaded, we create all the components we need to visualize images and timecourses as well as the components to interact with them. Why do we need all those `slider` components? In Streamlit, we need to rely on specific components to _update_ any visualization. For this application, it would be easier to just _click_ on the image to update the other plot. Unfortunately, this is not doable in Streamlit _yet_, at least out of the box. However, similar features are in the long list of potential enhancements for Streamlit (see this [issue](https://github.com/streamlit/streamlit/issues/298) and this [other one](https://github.com/streamlit/streamlit/issues/455)), so it is worth watching out for new releases. Also, the fact that it does not come _out of the box_ does not mean that it cannot be done: following up a [question on the Streamlit forum](https://discuss.streamlit.io/t/could-we-have-plotly-figurewidget-support/4271) and leveraging [Streamlit custom components](https://medium.com/streamlit/introducing-streamlit-components-d73f2092ae30), Fanilo Adrianasolo has showed [how this can be achieved](https://dev.to/andfanilo/streamlit-components-scatterplot-with-selection-using-plotly-js-3d7n), with a little more effort, embedding [plotly.js](https://plotly.com/javascript/).

The last three lines of the last snippet also offer an interesting feature to discuss: during the last fall, Streamlit introduced [dedicated layout components](https://blog.streamlit.io/introducing-new-layout-options-for-streamlit/) for tailoring how the dashboard looks like. In this case, we are relying on `streamlit.beta_columns` (that, as you may imagine, divides the dashboard in columns) in a very minimal way, but more complex layouts can be arranged in just a bunch of lines.

That's it, less than 40 lines of code, and the dashboard is ready to be run:

```
streamlit run bold_explorer_sl.py
```

It is noticeable that it does not feel as _snappy_ as a previous example ([BET Explorer](https://neurosnippets.com/posts/quick-stripper/#post)), but it should be evident from the previous paragraph, we are at the edge of (current) Streamlit use cases. Once again, the amazing Streamlit team is [already on it](https://github.com/streamlit/streamlit/issues/494).



## To tame a land: Dash

For anyone who is used to [Plotly](https://plotly.com/graphing-libraries/ 'Plotly'), [Dash](https://dash.plotly.com 'Dash') feels like its natural extension. Although we don't _magically_ get the interaction we may want without actually implementing it, in most cases a couple of short functions is all we need and even more. As a matter of fact, it is writing tailored functions that we can put together amazing results.

So we start importing the packages and loading the data:

```
import dash
import dash_core_components as dcc
import dash_html_components as html
import nibabel as nib
import numpy as np
import plotly.express as px
from dash.dependencies import Input, Output, State


img = nib.load('bold.nii.gz')
vol_4d = img.get_fdata()
mean_vol = np.mean(vol_4d, axis=3)
midslice = int(mean_vol.shape[2]/2)
fig = px.imshow(mean_vol[:,:,midslice].T, binary_string=True, origin='lower')
fig_tc = px.line()
```

Here we have a first difference with Streamlit: these lines of code (loading the data, retrieving an average volume from the whole dataset) will be executed just once, unless we refresh the page or restart the dashboard. We don't need to embed them in a specific function to make sure that we use the _cached_ values. On one hand this may sound intuitive ("_I don't have to think about cache!_"), but on the other hand it is something to keep in mind: changing one of these variables inside a _callback_ function is not the same as changing its value here! We will see about that in a bit.

We are ready to _sketch_ how the dashboard will look like:

```
app = dash.Dash(__name__)

app.layout = html.Div([html.Div([
        html.Div(
            [dcc.Graph(id='graph-mri', figure=fig),
             dcc.Slider(
                id='slice-slider',
                min=0,
                max=mean_vol.shape[2] - 1,
                value=midslice,
                step=1)],
            style={'width': '60%', 'display': 'inline-block', 'padding': '0 0'},
        ),
        html.Div(
            [dcc.Graph(id='graph-tc', figure=fig_tc)],
            style={'width': '40%', 'display': 'inline-block', 'padding': '0 0'},
        ), html.Div(id='current-slice', style={'display': 'none'})
    ])
])


```

Layout definition is clearly another big difference with Streamlit: the structure needs to be defined using HTML components and styled with CSS rules. Is this an advantage or a disadvantage? It depends very much on what we want to achieve: for a quick dashboard, we may not want to deal with this. If the goal is ultimately a web app it may not be an issue - it may even make things easier if it needs to be _integrated_ into something larger.

Although we defined all the elements and how they should be visually arranged, things do not _interact_ yet. We need to define some `callback` functions to achieve that:


```
@app.callback(
    Output('graph-tc', 'figure'),
    Input('graph-mri', 'clickData'),
    State('current-slice', 'children'),
    prevent_initial_call=True
)
def update_tc(clickData, vol_slice):
    tc = vol_4d[clickData['points'][0]['x'], clickData['points'][0]['y'], int(vol_slice), :]
    fig = px.line(x=np.arange(len(tc)), y=tc)
    return fig


@app.callback(
    [Output('graph-mri', 'figure'),
    Output('current-slice', 'children')],
    Input('slice-slider', 'value'))
def update_slice(vol_slice):
    fig = px.imshow(mean_vol[:,:,vol_slice].T, binary_string=True, origin='lower')
    return [fig, vol_slice]


if __name__ == "__main__":
    app.run_server(debug=True)
```

The function `update_tc()` takes care of generating the plot of the timecourse corresponding to the point we clicked in the image, while `update_slice()` updates the current volume slice on the basis of the slider. The decorators here specify inputs and outputs, and their order is _mirrored_ in the parameters and the returned values of the respective functions.

We need to pay attention to the fact that the current slice is needed by `updated_tc()` _as well_, to plot the right timecourse. To make sure that it is the case, one way (as done in the [histogrammer](https://neurosnippets.com/posts/mri-histogram/#post)) is to make a unique `callback` function, combining inputs and outputs, and find out which input has caused the function to be triggered using the [context object](https://dash.plotly.com/advanced-callbacks). The other way, as implemented here, is through data sharing between different `callback` functions. The current slice index is stored in a hidden `div` block (defined in the previous snippet and made invisible through the CSS property `display`). We can then update it through `update_slice()` and retrieve it through `update_tc()` when needed. As we don't want slice selection to trigger `update_tc()` (we didn't click!), in the `callback` decorator we handle the current slice as a [`State` rather than an `Input`](https://dash.plotly.com/basic-callbacks).

This brief explanation leads to two considerations. First, there are more concepts to grasp compared to streamlit (where we almost don't need to look at the documentation!!). Second, this _conceptual overhead_ allows us to tailor how the components interact with each other in great detail.

We are ready to go in around 60 lines of code:

```
python bold_explorer_dash.py
```

Quite a _fluid_ result, if you ask me!


## Useful references

* [Washington University 120 dataset](https://openneuro.org/datasets/ds000243/versions/00001 'OpenNeuro')
* [Streamlit](https://www.streamlit.io 'Streamlit')
* [Streamlit gallery](https://streamlit.io/gallery 'Streamlit Gallery')
* [Streamlit layout components](https://blog.streamlit.io/introducing-new-layout-options-for-streamlit/ 'Streamlit blog')
* [Streamlit custom components](https://medium.com/streamlit/introducing-streamlit-components-d73f2092ae30 'Medium')
* [Improve performances with Streamlit](https://docs.streamlit.io/en/stable/caching.html 'Streamlit documentation')
* [Dash](https://dash.plotly.com 'Dash')
* [Dash gallery](https://dash-gallery.plotly.host/Portal/ 'Dash Gallery')
* [Basic callbacks with Dash](https://dash.plotly.com/basic-callbacks 'Dash documentation')
* [Advanced callbacks with Dash](https://dash.plotly.com/advanced-callbacks 'Dash documentation')


