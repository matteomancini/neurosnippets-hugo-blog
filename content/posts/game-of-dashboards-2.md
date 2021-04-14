+++
title =  "Game of Dashboards: one brain, four frameworks [Part 2: Bokeh, HoloViz]"
tags = ["dataviz", "python", "bokeh", "holoviz", "fMRI"]
date = "2021-04-12"
image = "img/game_of_dashboards_2.svg"
caption = "Choosing a framework for a dashboard is just another way of posing once again [the general problem](https://xkcd.com/974/)."
+++


_This is the second part of the dashboard framework comparison. The first part is [here](https://neurosnippets.com/posts/game-of-dashboards/#post)._


## In the previous episode...

Let's pick up where we left off, shall we? Last time I sketched the idea of making the same dashboard with four different frameworks. We are halfway through: so far we've seen [Streamlit](https://streamlit.io) and [Dash](https://dash.plotly.com 'Dash'). But there is more out there!

In this second part I will be showcasing [Bokeh](https://bokeh.org 'Bokeh') and [HoloViz](https://holoviz.org 'HoloViz'). All the related code is available on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/dataviz/game-of-dashboards). 



## Running Free: Bokeh

In the python world/space/_dimension_, there is at least another big (and amazing) library for interactive visualization apart from Plotly, and that library is [Bokeh](https://docs.bokeh.org/en/latest/ 'Bokeh Documentation'). Despite their differences, both Plotly and Bokeh are a Python (and, in [both cases](https://plotly.com/graphing-libraries/ 'Plotly in other languages'), [beyond!](https://docs.bokeh.org/en/latest/docs/dev_guide/bindings.html 'Bokeh in other languages')) wrapper to Javascript-based visualization. As you may guess, the similarities do not stop here (otherwise Bokeh would not be in this post!): using [Bokeh server](https://docs.bokeh.org/en/latest/docs/user_guide/server.html 'Bokeh server') one can make amazing [dashboards](https://docs.bokeh.org/en/latest/docs/gallery.html 'Bokeh Gallery'). 

You probably know how it works now - we start importing the packages and defining the usual couple of functions:

```
from bokeh.layouts import layout
from bokeh.models import Slider
from bokeh.plotting import figure, curdoc
from bokeh.events import Tap
import nibabel as nib
import numpy as np


def update_slice(attr, old, new):
    new_data = dict(image=[mean_vol[:,:,slider.value].T])
    implot_source.data = new_data


def update_tc(event):
    new_data = dict(x=np.arange(0, vol_4d.shape[3]),
                    y=vol_4d[
                        np.floor(event.x).astype(int),
                        np.floor(event.y).astype(int),
                        slider.value, :]
                   )
    tcplot_source.data = new_data

```

We can already notice quite a difference if we compare the code to the examples from the previous post: interestingly enough, we are not _returning_ figures as we were doing in Streamlit and Dash - in fact, we are not returning anything! Each function creates a dictionary on the basis of some property and/or event that has occured (in a bit we will dig more into that), and use that dictionary to update the data of a _source_ object. It's hard to fully grasp what this means without working out the whole thing so let's proceed.

With our modules and functions, we can now _do stuff_:

```
img = nib.load('bold.nii.gz')
vol_4d = img.get_fdata()
mean_vol = np.mean(vol_4d, axis=3)
midslice = int(mean_vol.shape[2]/2)

plot1 = figure(toolbar_location=None, tools='hover')
plot1.axis.visible = False
plot1.xgrid.visible = False
plot2 = figure(background_fill_color="#fafafa", tools='hover')
slider = Slider(start=0, end=mean_vol.shape[2]-1, step=1, value=midslice, title='Slice')
```

The first block is quite similar to the other examples: loading the data using `nibabel`, computing the average volume, and so on. In the second block we are defining the components of our dashboard: two `figure` objects, both with the hovering tool enabled, and a `Slider`, that goes from the first to the last slice of our volume. It is worth noticing that the `figure` objects do not contain anything in terms of pictures or plots, we will need to fill them in some way.

As the _foundation_ of our dashboard is ready, we are ready to make things _interesting_:


```
implot = plot1.image(image=[mean_vol[:,:,midslice].T],
                     x=0, y=0, dw=mean_vol.shape[0], dh=mean_vol.shape[1])
implot_source = implot.data_source

tcplot = plot2.line(x=[], y=[], line_width=3)
tcplot_source = tcplot.data_source

slider.on_change('value', update_slice)
plot1.on_event(Tap, update_tc)

curdoc().add_root(layout([[slider],[plot1, plot2]]))
```

In the first two blocks we are adding content to our `figure` objects: we are respectively adding an image plot and a line one through the related methods of the `figure` object, and keeping a reference to those plots through the objects `implot` and `tcplot`. We are also keeping a reference of their `data_source` properties, which (as the name implies) describe the data that has been fed to the related plot. The third block is the key one for making things _interactive_: first, we decide that whenever the slider value changes, the function `update_slice` needs to be called. If we go back to that function, we can see that it updates the slice selected from the volume using the value from the slider - and the update is based on changing [the content of the source data](https://docs.bokeh.org/en/latest/docs/user_guide/data.html 'Providing data to Bokeh'). In the same way, we decide that whenever there is a `Tap` event (a mouse click) on the image plot, the function `update_tc` needs to be called to update line plot through its source. The very last line literally assembles the final document: using the `layout` object, each component can be positioned in a grid fashion through a _list of lists_, where the inner lists are rows and each of their elements is a column.

We are at 45 lines of code, that is less than Dash but more than Streamlit. We can try the dashboard out typing:

```
bokeh serve bold_explorer_bokeh.py
```

The result is as snappy and fluid as with Dash, and we are similarly able to further refine the behaviour of the dashboard.



## Phantom of the Opera: Holoviz

The last but not least framework in our list is [HoloViz](https://holoviz.org 'HoloViz'). HoloViz leverages other existing tools, like Plotly, Bokeh and many others, to provide a [layered high-level framework](https://holoviz.org/background.html 'HoloViz background') able to provide quick solutions (and we will see what _quick_ means here) but also chances to get detail-oriented. As I'll explain further at the very end of this post, HoloViz has a crucial parallelism to Plotly and Dash: it provides an actual _ecosystem_ of tools. In this specific application, we will focus on [Panel](https://panel.holoviz.org 'Panel') - the ideal solution for dashboards.

This time we will _take our time_ and define at the beginning just one of the functions we need (we'll see why in a bit):

```
import numpy as np
import panel as pn
import plotly.graph_objs as go
import nibabel as nib
import plotly.express as px


def get_slice(zslice=0):
    return px.imshow(mean_vol[:,:,zslice].T, binary_string=True, width=500, origin='lower')

```

There isn't much to comment here, except that we are saving lines of code accessing directly the average volume (that we'll need to define before calling this function, otherwise we'll be in trouble).

Let's look at the usual initialization step:

```
img = nib.load('bold.nii.gz')
vol_4d = img.get_fdata()
mean_vol = np.mean(vol_4d, axis=3)
pn.extension("plotly")
sliceplot = pn.interact(get_slice, zslice=np.arange(0, mean_vol.shape[2]))
```

Two things are new here: first, to use `plotly` in the dashboard, we are loading the related `panel` extension; second, and this is really mind-blowing, we are making our data volume navigable through a slider _in one line_. I'm serious - the last line here allows the dashboard to interact with the function `get_slice` through the parameter `zslice`, appropriately set to the list of slice indices. At it is set to a list, `panel.interact` automatically prepare a slider to go through the list. An important thing to notice here: `sliceplot` contains both the references to the slider and to the actual image plot, so if we want to access just one of them (as we will in a bit) we need to use `sliceplot` as it was an array (well, actually it is one!).

We are halfway through our dashboard, we still need to retrieve the desired time course through clicking the image:

```
@pn.depends(sliceplot[1][0].param.click_data)
def get_tc(click_data):
    if click_data is None:
        return px.line()
    else:
        return px.line(x=np.arange(0, vol_4d.shape[3]),
                       y=vol_4d[click_data['points'][0]['x'], click_data['points'][0]['y'],
                                sliceplot[0][0].value, :])


app = pn.Row(sliceplot, get_tc)
app.servable()
```

Here is the other function we were missing! The function uses the `click_data` parameter from the previous plot: if `click_data` is not `None` (so someone has clicked!), a `line` plot is generated using the coordinates of the click and the current value of the slider. To connect this function to the related widget, we are using a decorator, and it is because of that decorator that we cannot declare this function at the beginning (otherwise `sliceplot` would not exist!). The last two lines takes care of positioning the components (one after the other in a `Row`) and making this whole thing a deployable app.

And I guess we're done. _What do you mean we're done? That's 30 lines of code!!_ That's right. Honestly, the fact that you can make a whole dashboard in just 30 lines of code still blows me away. If you don't believe it, it's just a matter of typing:

```
panel serve bold_explorer_hv.py
```

Anyone will have a hard time telling this result from the previous one!


## Final showdown


At this point, a question is legit: _what should I use_? And unfortunately, the answer is one of the most hated: _it depends_.

First, an easy advice: if you have a script that needs to be turned into a dashboard, Streamlit is the easy answer. As most components can be implemented in _one line_, there is almost no need to _read_ any documentation. Even if you need start an actual dashboard, Streamlit can be a good solution if the time constraints are strict - especially if the main purpose of the dashboard is to play with some parameters and run over and over a target command/tool/monster (as in the [BET Explorer](https://neurosnippets.com/posts/quick-stripper/#post)). The application showcased here is at the edge of what can be done with Streamlit for now, but so far the Streamlit team has added feature after feature, and click-based interactions [may be coming soon](https://github.com/streamlit/streamlit/issues/494). A point to take into account when comparing with the other frameworks is layout customization: the recently added [layout components](https://blog.streamlit.io/introducing-new-layout-options-for-streamlit/ 'Streamlit layout') offer a good level of freedom to structure a dashboard, with the limit being multi-page applications (that is [still doable](https://discuss.streamlit.io/t/multi-page-apps/266 'Streamlit multi-page request') [playing around](https://discuss.streamlit.io/t/multi-page-app-with-session-state/3074 'Streamlit multi-page trick')).

What if Streamlit is not enough for a target application? Well, it starts to become a _flavour-dependent_ choice. The underlying interactive visualization library could give some direction: Dash may sound natural for Plotly users, and the same stands for Bokeh - but then in HoloViz _you can easily use both_. Working with the layout of the dashboard leads to substantial differences: in Dash it is necessary to rely on the HTML components and on CSS style sheets. While this offers an amazing flexibility, it requires both skills and effort - while with Bokeh and HoloViz there are solutions to avoid the HTML+CSS layer.

The _domain_ of the target application also may help in the choice of a framework: an interesting added value of Dash is its _ecosystem_ - in addition to the core components, there are several of them for specific fields. A incomplete list includes: handling [3D volumes](https://github.com/plotly/dash-slicer 'Dash Slicer') for medical imaging; representing [network models](https://dash.plotly.com/cytoscape 'Cytoscape'); [drawing](https://dash.plotly.com/canvas 'Drawing with Dash') and [image annotations](https://dash.plotly.com/annotations 'Image annotations with Dash') for computer vision. The HoloViz ecosystem also offers [potential domain-specific tools](https://holoviz.org/background.html 'HoloViz Ecosysten'), especially for [geospatial data](http://geoviews.org 'GeoViews').

My final recommendation is one: try them out!



## Useful references

* [Washington University 120 dataset](https://openneuro.org/datasets/ds000243/versions/00001 'OpenNeuro')
* [Bokeh](https://bokeh.org 'Bokeh')
* [Bokeh gallery](https://docs.bokeh.org/en/latest/docs/gallery.html 'Bokeh Gallery')
* [Providing data to Bokeh](https://docs.bokeh.org/en/latest/docs/user_guide/data.html 'Bokeh documentation')
* [Going deeper in Bokeh events](https://docs.bokeh.org/en/latest/docs/reference/events.html 'Bokeh reference')
* [HoloViz](https://holoviz.org 'HoloViz')
* [HoloViz gallery](https://panel.holoviz.org/gallery/index.html 'HoloViz Gallery')
* [HoloViz goal and ecosystem](https://holoviz.org/background.html 'HoloViz background')
* [Panel](https://panel.holoviz.org 'Panel')


