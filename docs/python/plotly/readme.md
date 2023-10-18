# Plotly snippets

- [Plotly snippets](#plotly-snippets)
  - [Basic configurations](#basic-configurations)
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
    - [Changing ticks spacing](#changing-ticks-spacing)
    - [Manual change of ticks](#manual-change-of-ticks)
    - [Fixing narrow boxplots when using colors](#fixing-narrow-boxplots-when-using-colors)
    - [Removing "=" in facets](#removing--in-facets)
    - [Add ABline](#add-abline)
    - [Adding custom prefix or suffix to ticks](#adding-custom-prefix-or-suffix-to-ticks)
    - [Change text position](#change-text-position)
    - [Change opacity](#change-opacity)
    - [Specific color mapping](#specific-color-mapping)
    - [Change colorbar title](#change-colorbar-title)
    - [When multiple subplots](#when-multiple-subplots)
      - [Don't match all axis](#dont-match-all-axis)
      - [Show ticks on all subplot's axis](#show-ticks-on-all-subplots-axis)
    - [Barplots](#barplots)
      - [Stack bars on top of each other](#stack-bars-on-top-of-each-other)
      - [Stack multiple bars in the same place](#stack-multiple-bars-in-the-same-place)
  - [Plotting large number of points](#plotting-large-number-of-points)
    - [Snippet for large scatterplot](#snippet-for-large-scatterplot)

```{python}
import plotly.express as px
p = px.scatter(...)
```


## Basic configurations

### General theming

```{Python}
import plotly.io as pio
```

---

#### Force notebook renderer

```{Python}
pio.renderers.default = "notebook"
```

---

#### Dark mode

```{Python}
pio.templates.default = "plotly_dark"
```

### Layout patterns

---

#### Supress margins

```{Python}
no_margins = dict(l=0, r=0, t=0, b=0)
(...)
p.update_layout(margins=no_margins)
```

---

#### Floating legends

```{Python}
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

```{Python}
px_custom = {
    "margin": no_margins,
    "legend": floating_legend,
}
p.update_figure(**px_custom)
```

## Subplots

### Create a subplot figure object

```{Python}
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

```{Python}
fig.add_trace(
    px. (or) go. ...,
    row=1,
    col=1,
)
```

## Colors

### Make a projection of a palette on specific points

```{Python}
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

```{Python}
for data in fig.data:
    data.line.update(
      color="rgb(0, 255, 255)"
    )
```

## General figure modification

---

### Changing ticks spacing

```{Python}
p.update_layout(xaxis=dict(dtick=12))
```

---

### Manual change of ticks

```{Python}
p.update_xaxes(
    tickmode = 'array',
    tickvals = {array_like},
    tick
)```


---

### Putting xticks at the top

```{Python}
p.update_xaxes(side="top")
```

---

### Fixing narrow boxplots when using colors

Bug fixed in version 4.8.

Workaround: add `boxmode="overlay"` in the arguments.

---

### Removing "=" in facets

```{Python}
p.for_each_annotation(
    lambda t: t.update(text=t.text.split("=")[1])
)
```

### Add ABline

Unfortunately, traces will be present in the legend, but there is a way to avoid that (in the code).

Because of the ABline, there will be a zoom out, so you will also need to either set x/y ranges, or set the ABline points more intelligently.

```{Python}
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

```{Python}
p.update_traces(ticksuffix="%")
p.update_traces(tickprefix="Hello")
```

---

### Change text position

When plotting you can add labels by using `text=`. The position can be modified by using :

```{Python}
p.update_traces(textposition='top center')
```

### Change opacity

```{Python}
p.update_traces(opacity=.5)
```

### Specific color mapping

Specific color mappings can be made with dictionnaries :
`{modality: color, ...}`

One easy trick to outline specific modality is to create a function that highlights a specific modality.

```{Python}
highlight = lambda data, mod: {mod: "red", **{s: "blue" for s in data.source.unique() if s != mod}}
```

This can be called directly in the creation of the plot:

```{Python}
px.scatter(
    dataframe,
    x="x",
    y="y",
    color="some_3rd_dimension",
    color_discrete_map=highlight(dataframe, "some_3rd_dimension")
)
```

### Change colorbar title

```{Python}
fig.update_coloraxes(colorbar=dict(title="This is my colorbar title"))
```

### When multiple subplots

#### Don't match all axis

```{Python}
p.update_yaxes(matches=None)
(or)
p.update_xaxes(matches=None)
```

---

#### Show ticks on all subplot's axis

```{Python}
p.for_each_yaxis(lambda yaxis: yaxis.update(showticklabels=True))
(or)
p.for_each_xaxis(lambda xaxis: xaxis.update(showticklabels=True))
```

### Barplots

#### Stack bars on top of each other

```{Python}
p.update_layout(barmode="stack")
```

#### Stack multiple bars in the same place

```{Python}
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

```{python}
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

```{python}
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

```{python}
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
