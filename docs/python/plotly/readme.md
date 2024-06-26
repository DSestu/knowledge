---
layout: default
title: Plotly snippets
nav_order: 1
parent: Python
---

# Plotly snippets

- [Plotly snippets](#plotly-snippets)
  - [Basic configurations](#basic-configurations)
    - [Fix static image export in windows](#fix-static-image-export-in-windows)
    - [General theming](#general-theming)
      - [Force notebook renderer](#force-notebook-renderer)
      - [Dark mode](#dark-mode)
    - [Layout patterns](#layout-patterns)
      - [Supress margins](#supress-margins)
      - [Floating legends](#floating-legends)
      - [No margins \& floating legends](#no-margins--floating-legends)
  - [Subplots](#subplots)
    - [Create a subplot figure object](#create-a-subplot-figure-object)
  - [Colors](#colors)
    - [Make a projection of a palette on specific points](#make-a-projection-of-a-palette-on-specific-points)
    - [A-posteriori modification of trace colors](#a-posteriori-modification-of-trace-colors)
  - [General figure modification](#general-figure-modification)
    - [Completely changing the hover data](#completely-changing-the-hover-data)
    - [Changing ticks spacing](#changing-ticks-spacing)
    - [Manual change of ticks](#manual-change-of-ticks)
    - [Putting xticks at the top](#putting-xticks-at-the-top)
    - [Fixing narrow boxplots when using colors](#fixing-narrow-boxplots-when-using-colors)
    - [Removing "=" in facets](#removing--in-facets)
    - [Add ABline](#add-abline)
    - [Adding custom prefix or suffix to ticks](#adding-custom-prefix-or-suffix-to-ticks)
    - [Change text position](#change-text-position)
    - [Change opacity](#change-opacity)
    - [Specific color mapping](#specific-color-mapping)
    - [Change colorbar title](#change-colorbar-title)
    - [Remove colorbar](#remove-colorbar)
    - [When multiple subplots](#when-multiple-subplots)
      - [Don't match all axis](#dont-match-all-axis)
      - [Show ticks on all subplot's axis](#show-ticks-on-all-subplots-axis)
    - [Barplots](#barplots)
      - [Stack bars on top of each other](#stack-bars-on-top-of-each-other)
      - [Stack multiple bars in the same place](#stack-multiple-bars-in-the-same-place)
  - [Plotting large number of points](#plotting-large-number-of-points)
    - [Snippet for large scatterplot](#snippet-for-large-scatterplot)

```python
import plotly.express as px
p = px.scatter(...)
```

## Basic configurations

### Fix static image export in windows

The kaleido package from pypi doesn't work properly.

In order for the png renderer to work you need to install this version of kaleido (for AMD processors):

```bash
pip https://github.com/plotly/Kaleido/releases/download/v0.1.0.post1/kaleido-0.1.0.post1-py2.py3-none-win_amd64.whl
```

### General theming

```python
import plotly.io as pio
```


---

#### Force notebook renderer

```python
pio.renderers.default = "notebook"
```

---

#### Dark mode

```python
pio.templates.default = "plotly_dark"
```

---

### Changing default font

```python
import plotly.io as pio
import plotly.graph_objects as go

# Create a custom template
pio.templates["my_template"] = go.layout.Template(
  layout=dict(font_family="MathJax")
)
# Apply the template
pio.templates.default = "plotly_white+my_template"
```

### Layout patterns

---

#### Supress margins

```python
no_margins = dict(l=0, r=0, t=0, b=0)
(...)
p.update_layout(margins=no_margins)
```

---

#### Floating legends

```python
floating_legend = dict(
    yanchor="top",
    y=0.99,
    xanchor="left",
    x=0.01
)
(...)
p.update_layout(legend=floating_legend)
```

---

#### No margins & floating legends

```python
px_custom = {
    "margin": no_margins,
    "legend": floating_legend,
}
p.update_figure(**px_custom)
```

## Subplots

### Create a subplot figure object

```python
from plotly.subplots import make_subplots

fig = make_subplots(
    rows=nrows,
    cols=ncols,
    # You have to explicitely write the type of plot in the specs
    specs=[{"type": "mapbox"} for _ in range(nrows*ncols)]
    horizontal_spacing=...,
    vertical_spacing=...,
    # You can name the subplots here
    subplot_titles=["Sp1", "Sp2", ...]
)
```

Then, you have to add traces with the `row`, `col` parameters (**starts at 1**)

```python
fig.add_trace(
    px.(...).data[0] (or) go. ...,
    row=1,
    col=1,
)
```

### Change axis labels

```python
fig['layout']['xaxis']['title']='Label x-axis 1'
fig['layout']['xaxis2']['title']='Label x-axis 2'
fig['layout']['yaxis']['title']='Label y-axis 1'
fig['layout']['yaxis2']['title']='Label y-axis 2'
```

```python
label_x = ""
label_y = textbf("Percent of usage (%)")
for i in range(max_col):
    number = str(i) if i > 0 else ""
    figure["layout"][f"xaxis{number}"]["title"] = label_x
    figure["layout"][f"yaxis{number}"]["title"] = label_y
```

## Colors

### Make a projection of a palette on specific points

```python
px.colors.sample_colorscale(
    # Color scale
    px.colors.sequential.Bluered,
    # Coordinates of the projection (array-like) [0; 1]
    [0.3, 0.4, ...]
)
```

### A-posteriori modification of trace colors

Traces are found in `fig.data` as a `list`.

Colors can be interpreted with Hex and RGB(A) notations.

```python
for data in fig.data:
    data.line.update(
      color="rgb(0, 255, 255)"
    )
```

## General figure modification

---

### Format percents in text

```python
fig.update_traces(
  # Rounds and add %
  textemplate="%{text:.2f}%"
)
```

---

### Completely changing the hover data

You can completely change the hover format with the following approach:

* If you want to use variables inside the formatting:

```python
px.area(
    (...),
    custom_data=["my_col_1", "my_col_2"],
)
```

Then you can change the format:

```python
px.area(
    (...)
).update_traces(
    hovertemplate="<br>".join([
        # Use the first column from custom_data, bolded
        "<b>%{customdata[0]}</b>",
        # Use the value of the X axis, even if the name is not "x"
        "The x-axis equals to %{x}",    
        # Use the value of the Y axis, even if the name is not "y"
        "The y-axis equals to %{y}",
    ])
)
```

**Note: Case of multiple Y plots**

The case of multiple Y plots is as following:

```python
fig = px.area(
  (...),
  y=["var1", "var2", "var3"]
)
```

You may want to include this variable name in the hover template.

> You can only do this with manual tweaking of figure data.

The `customdata` is stored as a numpy array, you need to change it manually.

The shape of `customdata` is `(number_of_rows, number_of_custom_columns)`

```python
import numpy as np

for trace in fig.data:
  trace["customdata"] = np.hstack([
    customdata := trace["customdata"],
    # Replace trace.name with any data you want to display
    np.repeat([trace.name], customdata.shape[0]).reshape(-1, 1)
  ])
```


---

### Changing ticks spacing

```python
p.update_layout(xaxis=dict(dtick=12))
```

---

### Manual change of ticks

```python
p.update_xaxes(
    tickmode = 'array',
    tickvals = {array_like},
    tick
)
```


---

### Putting xticks at the top

```python
p.update_xaxes(side="top")
```

---

### Fixing narrow boxplots when using colors

Bug fixed in version 4.8.

Workaround: add `boxmode="overlay"` in the arguments.

---

### Removing "=" in facets

```python
p.for_each_annotation(
    lambda t: t.update(text=t.text.split("=")[1])
)
```

### Add ABline

Unfortunately, traces will be present in the legend, but there is a way to avoid that (in the code).

Because of the ABline, there will be a zoom out, so you will also need to either set x/y ranges, or set the ABline points more intelligently.

```python
p.add_trace(
    go.Scatter(
        x=[0, 1],
        y=[0, 1],
        mode="lines",
        # If you want to fill to bottom
        fill="tozeroy",
        # Line color (Here invisible)
        marker_color="rgba(255, 255, 255, 0)",
        # Fill color (Here white with 20% opacity)
        fillcolor="rgba(255, 255, 255, 0.2)",
    ).update(showlegend=False),
    # If you use a facet_row/column, apply to each subplot
    row="all",
    col="all",
)
```

---

### Adding custom prefix or suffix to ticks

```python
p.update_traces(ticksuffix="%")
p.update_traces(tickprefix="Hello")
```

---

### Change text position

When plotting you can add labels by using `text=`. The position can be modified by using :

```python
p.update_traces(textposition='top center')
```

### Change opacity

```python
p.update_traces(opacity=.5)
```

### Specific color mapping

Specific color mappings can be made with dictionnaries :
`{modality: color, ...}`

One easy trick to outline specific modality is to create a function that highlights a specific modality.

```python
highlight = lambda data, mod: {mod: "red", **{s: "blue" for s in data.source.unique() if s != mod}}
```

This can be called directly in the creation of the plot:

```python
px.scatter(
    dataframe,
    x="x",
    y="y",
    color="some_3rd_dimension",
    color_discrete_map=highlight(dataframe, "some_3rd_dimension")
)
```

### Change colorbar title

```python
fig.update_coloraxes(colorbar=dict(title="This is my colorbar title"))
```

### Remove colorbar

```python
fig.update_coloraxes(showscale=False)
```

### When multiple subplots

#### Don't match all axis

```python
p.update_yaxes(matches=None)
(or)
p.update_xaxes(matches=None)
```

---

#### Show ticks on all subplot's axis

```python
p.for_each_yaxis(lambda yaxis: yaxis.update(showticklabels=True))
(or)
p.for_each_xaxis(lambda xaxis: xaxis.update(showticklabels=True))
```

### Barplots

#### Stack bars on top of each other

```python
p.update_layout(barmode="stack")
```

#### Stack multiple bars in the same place

```python
p.update_layout(barmode="overlay")
```

---

## Plotting large number of points

When plotting a large number of points (several millions points), Plotly scatterplots suffers a lot.

We can use a complementary library `datashader`

```
import datashader as ds
```

> Datashader is about the rasterisation of plots. Instead of plotting all single points, it computes a density image that can be later shown with `px.imshow`

Example:

```python
cvs = ds.Canvas(
  plot_width= ..., # Number of horizontal pixels
  plot_height= ..., # Number of vertical pixels
)
agg = csv.points(
  dataframe,
  x_column_name,
  y_column_name,
) # Creates the numpy array containing density values

# Here we want to exclude pixels where there are no points
zero_mask = agg.values == 0
# For big data reasons, we may want to look at the logarithm of density
agg.values = np.log10(
  agg.values, where=np.logical_not(zero_mask)
)
agg.values[zero_mask] = np.nan
# plot the final image
fig = px.imshow(
  agg,
  origin="lower",
  labels={"color": "Log10(count)"},
)
fig.update_traces(hoverongaps=False, opacity=.9)
fig.update_layout(coloraxis_colorbar=dict(title='Count', tickprefix='1.e'))
fig.show()
```

### Snippet for large scatterplot

```python
def big_scatter(
    data: pd.DataFrame,
    x: str,
    y: str,
    plot_width=400,
    plot_height=400,
    **plotly_kwargs,
):
    cvs = ds.Canvas(plot_width=plot_width, plot_height=plot_height)
    agg = cvs.points(data, x, y)
    zero_mask = agg.values == 0
    agg.values = np.log10(
        agg.values, where=np.logical_not(zero_mask)
    )
    agg.values[zero_mask] = np.nan
    px.imshow(
        agg,
        origin="lower",
        labels={"color": "Log10(count)"},
        **plotly_kwargs,
    ).update_traces(hoverongaps=False, opacity=.9).update_layout(coloraxis_colorbar=dict(title='Count', tickprefix='1.e')).show()
```

> Usage :

```python
big_scatter(
    df,
    x="hour",
    y="error",
    plot_width=23,
    plot_height=200,
    height=800,
    title=model
)

```
