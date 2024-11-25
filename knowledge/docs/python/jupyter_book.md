# Jupyter Book 


## Plotly compatibility & no execution

In `_config.yml` add:


```yaml
execute:
  execute_notebooks: off

sphinx:
  config:
    html_js_files:
    - https://cdnjs.cloudflare.com/ajax/libs/require.js/2.3.4/require.min.js
    suppress_warnings: ["mystnb.unknown_mime_type"]

html:
  extra_footer: <script src="https://cdn.plot.ly/plotly-latest.min.js"></script>
```
