+++
title =  "Interactive treemaps for interactive reviews"
tags = ["python", "dataviz", "plotly"]
date = "2021-01-20"
image = "img/treemaps.gif"
caption = "An interactive treemap is a good companion to a meta-analysis. Unless, you know, it becomes [too meta](https://xkcd.com/1447/ 'Meta-meta-meta-analysis')."
+++


## Beyond rows and columns


If you think about a literature review or a meta-analysis, your expectation in terms of format probably reflects the following description: a PDF file containing a large table with the main details from the surveyed studies. This immediate association is the result of some sort of _immutability in time_ of reviews, and more in general scientific papers. But there is no reason for papers to be conservative, static objects. In this post, I want to provide an example of an interactive element that can be used to complement a table in a review or in a meta-analysis. For a glimpse of how it looks in the wild, [this executable research article (ERA)](https://elifesciences.org/articles/61523/executable 'Mancini et al., 2020') on _Elife_ offers several flavours of interactive figures.

The Jupyter Notebook to generate the interactive treemap and the related requirements are on the [NeuroSnippets repository](https://github.com/matteomancini/neurosnippets/tree/master/dataviz/interactive-treemaps). To try it out directly from the browser, you can use [Google Colab](https://colab.research.google.com/github/matteomancini/neurosnippets/blob/master/dataviz/interactive-treemaps/treemap.ipynb 'Open in Colab') or [MyBinder](https://mybinder.org/v2/gh/matteomancini/neurosnippets/master?filepath=dataviz/interactive-treemaps/treemap.ipynb 'Open in MyBinder').


## Making (interactive) stuff up


To keep the topic on _brainy_ stuff, I picked [this systematic review](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6360487/ 'Straatof et al., 2019') by Milou Straathof and colleagues. The topic is the correlation between structural and functional connectivity using different kinds of data. To keep things simple, I focused on the use of diffusion-weighted and functional MRI data, as described in [this table](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6360487/table/table1-0271678X18809547/?report=objectonly 'Resting-state functional connectivity and diffusion-based structural connectivity'). For each study reported, I collected all the relevant details and filled them in [this spreadsheet](https://github.com/matteomancini/neurosnippets/raw/master/dataviz/interactive-treemaps/dataset.xlsx 'NeuroSnippets GitHub'). I kept the correlation values for all the parcel resolution levels (high, low or voxel) reported, but considered only a single correlation value for each resolution. The goal was to arrange the studies in a treemap showing the correlation values reported and the related details depending on the resolution used.

Once we have a spreadsheet like this one, we can quickly read it with [`pandas`](https://pandas.pydata.org 'pandas') and do all kind of stuff. What we need to pick now is a suitable graphing library. I have chosen [`plotly`](https://plotly.com/python/ 'Plotly for Python') because it allows to make wonders in few lines of code, it is available in three programming languages (`javascript`, `python` and `R`) and last but not least it rapidly fills Jupyter notebooks with beautiful interactive figures.

There are two routes we can take for making a treemap: [`plotly.express`](https://plotly.com/python/plotly-express/ 'Plotly Express in python') and [`plotly.graph_objects`](https://plotly.com/python/graph-objects/ 'Plotly Graph Objects in Python'). The `plotly.express` module provides functions that can create at once a figure directly from a `pandas` dataframe. The catch is being less able to customize the final result. In `plotly.graph_objects`, we have more barebone functions. For this specific scenario, we will go the second route, as it will allow us to literally _fill_ the treemap.

Since we have picked `plotly.graph_objects`, rather then working directly with an arbitrary dataframe, we will need to build a specific one where we define all the elements of the treemaps, and for each of them their parent. What is a treemap in the end? It is a map based on a tree structure, so the inner elements (or _leaves_) need to be inside outer ones (or _branches_). As we want to organize the leaves using the respective resolution as a branch, we need to generate a hierarchical dataframe. In the `plotly` documentation there is [an example of how to do that](https://plotly.com/python/treemaps/#treemap-of-a-rectangular-dataframe-with-continuous-color-argument-in-pxtreemap 'Treemaps with a continuous colorscale') -- we will extend it in order to handle an edge case: leaves with the same names but in different branches. We have to deal with this case as we have studies reporting correlation values for both low and high resolution levels.
This is how it looks like:


```

import pandas as pd
import plotly.graph_objects as go

def build_hierarchical_dataframe(df, levels, value_column, color_columns=None):
    """
    Build a hierarchy of levels for Treemap charts.

    Levels are given starting from the bottom to the top of the hierarchy,
    ie the last level corresponds to the root.
    It can handle leaves with the same name but in different branches.
    """
    df_all_trees = pd.DataFrame(columns=['id', 'labels', 'parent', 'value', 'color'])
    for i, level in enumerate(levels):
        df_tree = pd.DataFrame(columns=['id', 'labels', 'parent', 'value', 'color'])
        dfg = df.groupby(levels[i:]).sum()
        dfg = dfg.reset_index()
        df_tree['labels'] = dfg[level].copy().astype(str)
        df_tree['parent'] = ''
        df_tree['id'] = dfg[level].copy().astype(str)
        if i < len(levels) - 1:
            j = i + 1
            while j < len(levels):
                df_tree['parent'] = (
                    dfg[levels[j]].copy().astype(str) + '/' + df_tree['parent']
                )
                df_tree['id'] = dfg[levels[j]].copy().astype(str) + '/' + df_tree['id']
                j += 1
        df_tree['parent'] = df_tree['parent'].str.rstrip('/')
        df_tree['value'] = dfg[value_column]
        df_tree['color'] = dfg[color_columns[0]] / dfg[color_columns[1]]
        df_all_trees = df_all_trees.append(df_tree, ignore_index=True)
    return df_all_trees
```


What we are doing in this function is:

1. initializing an empty dataframe;

2. looping through the levels of our hierarchy, from the inner to the outer;

3. setting the right parent for each element except for the top of the hierarchy;

4. using an additional _id_ field that prepends the parent to each element to avoid issues with same-name leaves.

This function can look intimidating, but fortunately it is immediate to use. Which arguments does it require? Well, first the actual dataframe; then a list of the columns that we want to use as levels in the hierarchy (in our case, the resolution and the study columns); a column that determines the size of each leaf element (that we won't quite use here); and finally, how to determine the color of each leaf element. One thing must be noticed: for the color, we need a list of _two_ columns; this allows the function to generalise well in other cases, but here we will be using only one column (as we will see very soon).
Ok, so now, where is the data?


## Some gymnastics with `pandas`

It is time to extract the data from the spreadsheet and do a bit of gymnastics with `pandas`. Here it comes:


```

df = pd.read_excel('dataset.xlsx')
df['Study'] = df['First author'] + ' et al., ' + df['Year'].astype(str)
df['Resolution'] = df['Resolution'] + ' resolution'

df['Link'] = df['DOI']
df['Link'].replace('http',"""<a style='color:white' href='http""",
                    inplace=True, regex=True)
df['Link'] = df['Link'] + """'>->Go to the paper</a>"""

fields = ['Title', 'Species (n)', 'Age',
          'Structural measure (diffusion)', 'Functional measure', 'Regions (n)',
         'Cortical or subcortical', 'Inter- or intra-hemispheric', 'Correlation method']
df['Summary'] = df['Link'] + '<br><br>'
for i in fields:
    df['Summary'] = df['Summary'] + i + ': ' + df[i].astype(str) + '<br><br>'
    
df['Review'] = 'Straathof et al., 2019'
df['Count'] = 1

df = df.sort_values('Study')

df_treemap = build_hierarchical_dataframe(df,
                                            ['Study', 'Resolution', 'Review'],
                                            'Count', ['Correlation', 'Count'])

```


In this snippets, what we are doing is:

1. initializing a dataframe directly from the spreadsheet;

2. defining a study columns combining first author and year, and adjusting the resolution column for displaying reasons;

3. doing a little trick to make the DOI become an actual clickable link inside the treemap;

4. looping through the columns we want to include inside each element of the treemap and adding them to a summary column;

5. adding a couple of other column: the review column, that will serve as the top of the hierarchy (the _parent of parents_), and the count column, which is (literally) a column of ones;

6. calling the function we defined in the previous section.

Why do we need the count field? In this example, we want all the elements of the treemaps to be equal, and as we need to define each element size a column of ones is the easiest way. The count column is also handy to respect the requirement for the color argument of our function.
We are almost there! 


## Let there be interactions


Let's finally generate the figure:
  

```
fig = go.Figure(go.Treemap(
    ids=df_treemap['id'].tolist(),
    labels=df_treemap['labels'].tolist(),
    parents=df_treemap['parent'].tolist(),
    values=df_treemap['value'].tolist(),
    branchvalues='total',
    hovertemplate='<b>%{label}</b><br>Correlation: %{color:.2f}',
    marker=dict(
        colors=df_treemap['color'],
        colorscale='cividis',
        showscale=True),
    name='',
    text=df['Summary'],
    textfont=dict(
            size=15,
        )
    ))

fig = fig.update_layout(
    title=dict(text='Brain connectivity: structure-function correlation', x=0.1),
    autosize=False,
    width=900,
    height=600,
    margin=dict(
        l=100,
        r=0,
        b=30,
        t=60,
    )
)


```

Apart from _stylistical_ adjustments, what is mainly going on here is that we are finally generating the treemap: aside from using the hierarchical dataframe columns (`id`, `labels`, `parent`, `value` and `color`), we are also setting the `text` and the `hovertemplate` arguments to define what will appear respectively inside the treemap and when hovering over its elements. Finally, the `branchvalues` argument will determine how the values of leaves will be combined in parent elements: as we set it to `'total'`, the size of each parent will be given by the number of leaves. Not only that: from the way the previous function was written, the color of each parent is given by the average of the color column value of its leaves!
So, where's the figure? Ah, here is it:


```
fig.show()

```


_Behold,_ an interactive treemap!

  

## Useful references


* ["An interactive meta-analysis of MRI biomarkers of myelin"](https://elifesciences.org/articles/61523/executable 'Mancini et al., 2020')

* ["A systematic review on the quantitative relationship between structural and functional network connectivity strength in mammalian brains"](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6360487/ 'Straathof et al., 2019')

* [Plotly Python Open Source Graphing Library](https://plotly.com/python/ 'Charts and Graphs available in Plotly')

* [Treemaps in Plotly](https://plotly.com/python/treemaps/ 'Treemap Charts')

* [Plotly cheat sheet](https://images.plot.ly/plotly-documentation/images/python_cheat_sheet.pdf)
