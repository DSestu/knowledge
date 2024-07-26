---
layout: default
title: Plotly snippets
nav_order: 1
parent: Python
---

# Plotly snippets

```
import plotly.express as px
p = px.scatter(...)
```

# Basic configurations

## Fix static image export in windows

The kaleido package from pypi doesn't work properly.

In order for the png renderer to work you need to install this version of kaleido (for AMD processors):

```bash
pip install https://github.com/plotly/Kaleido/releases/download/v0.1.0.post1/kaleido-0.1.0.post1-py2.py3-none-win_amd64.whl
```

## General theming

```
import plotly.io as pio
```


---

## Force notebook renderer

```
pio.renderers.default = "notebook"
```

---

## Dark mode

```
pio.templates.default = "plotly_dark"
```

---

## Changing default font

```
import plotly.io as pio
import plotly.graph_objects as go

# Create a custom template
pio.templates["my_template"] = go.layout.Template(
  layout=dict(font_family="MathJax")
)
# Apply the template
pio.templates.default = "plotly_white+my_template"
```

# Layout patterns

---

## Supress margins

```
no_margins = dict(l=0, r=0, t=0, b=0)
(...)
p.update_layout(margins=no_margins)
```

---

## Floating legends

```
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

## No margins & floating legends

```
px_custom = {
    "margin": no_margins,
    "legend": floating_legend,
}
p.update_figure(**px_custom)
```

# Subplots

## Create a subplot figure object

```
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

```
fig.add_trace(
    px. (or) go. ...,
    row=1,
    col=1,
)
```

# Colors

## Make a projection of a palette on specific points

```
px.colors.sample_colorscale(
    # Color scale
    px.colors.sequential.Bluered,
    # Coordinates of the projection (array-like) [0; 1]
    [0.3, 0.4, ...]
)
```

## A-posteriori modification of trace colors

Traces are found in `fig.data` as a `list`.

Colors can be interpreted with Hex and RGB(A) notations.

```
for data in fig.data:
    data.line.update(
      color="rgb(0, 255, 255)"
    )
```

# General figure modification

---

## Format percents in text

```
fig.update_traces(
  # Rounds and add %
  textemplate="%{text:.2f}%"
)
```

---

## Completely changing the hover data

You can completely change the hover format with the following approach:

* If you want to use variables inside the formatting:

```
px.area(
    (...),
    custom_data=["my_col_1", "my_col_2"],
)
```

Then you can change the format:

```
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

```
fig = px.area(
  (...),
  y=["var1", "var2", "var3"]
)
```

You may want to include this variable name in the hover template.

> You can only do this with manual tweaking of figure data.

The `customdata` is stored as a numpy array, you need to change it manually.

The shape of `customdata` is `(number_of_rows, number_of_custom_columns)`

```
import numpy as np

for trace in fig.data:
  trace["customdata"] = np.hstack([
    customdata := trace["customdata"],
    # Replace trace.name with any data you want to display
    np.repeat([trace.name], customdata.shape[0]).reshape(-1, 1)
  ])
```


---

## Changing ticks spacing

```
p.update_layout(xaxis=dict(dtick=12))
```

---

## Manual change of ticks

```
p.update_xaxes(
    tickmode = 'array',
    tickvals = {array_like},
    tick
)
```


---

## Putting xticks at the top

```
p.update_xaxes(side="top")
```

---

## Fixing narrow boxplots when using colors

Bug fixed in version 4.8.

Workaround: add `boxmode="overlay"` in the arguments.

---

## Removing "=" in facets

```
p.for_each_annotation(
    lambda t: t.update(text=t.text.split("=")[1])
)
```

## Add ABline

Unfortunately, traces will be present in the legend, but there is a way to avoid that (in the code).

Because of the ABline, there will be a zoom out, so you will also need to either set x/y ranges, or set the ABline points more intelligently.

```
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

## Adding custom prefix or suffix to ticks

```
p.update_traces(ticksuffix="%")
p.update_traces(tickprefix="Hello")
```

---

## Change text position

When plotting you can add labels by using `text=`. The position can be modified by using :

```
p.update_traces(textposition='top center')
```

## Change opacity

```
p.update_traces(opacity=.5)
```

## Specific color mapping

Specific color mappings can be made with dictionnaries :
`{modality: color, ...}`

One easy trick to outline specific modality is to create a function that highlights a specific modality.

```
highlight = lambda data, mod: {mod: "red", **{s: "blue" for s in data.source.unique() if s != mod}}
```

This can be called directly in the creation of the plot:

```
px.scatter(
    dataframe,
    x="x",
    y="y",
    color="some_3rd_dimension",
    color_discrete_map=highlight(dataframe, "some_3rd_dimension")
)
```

## Change colorbar title

```
fig.update_coloraxes(colorbar=dict(title="This is my colorbar title"))
```

## Remove colorbar

```
fig.update_coloraxes(showscale=False)
```

## When multiple subplots

### Don't match all axis

```
p.update_yaxes(matches=None)
(or)
p.update_xaxes(matches=None)
```

---

### Show ticks on all subplot's axis

```
p.for_each_yaxis(lambda yaxis: yaxis.update(showticklabels=True))
(or)
p.for_each_xaxis(lambda xaxis: xaxis.update(showticklabels=True))
```

## Barplots

### Stack bars on top of each other

```
p.update_layout(barmode="stack")
```

### Stack multiple bars in the same place

```
p.update_layout(barmode="overlay")
```

---

# Plotting large number of points

When plotting a large number of points (several millions points), Plotly scatterplots suffers a lot.

We can use a complementary library `datashader`

```
import datashader as ds
```

> Datashader is about the rasterisation of plots. Instead of plotting all single points, it computes a density image that can be later shown with `px.imshow`

Example:

```
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

## Snippet for large scatterplot

```
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

```
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
