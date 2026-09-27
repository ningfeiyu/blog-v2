+++
title = 'Search'
date = '2025-03-24T23:00:00-03:00'
series = ['content','features']
+++

This theme includes built-in search powered by FlexSearch. It works by generating a search index at build time. To enable it, add the following to your config file

```toml
[outputFormats]
 [outputFormats.SearchIndex]
  mediaType = "application/json"
  baseName = "searchindex"
  isPlainText = true
  notAlternative = true

[mediaTypes]
 [mediaTypes."application/json"]
  suffixes = ["json"]

[outputs]
 home = ["HTML","RSS","SearchIndex"]
```

The JSON output is served as search index at path `/searchindex.json`, which FlexSearch consumes on the client's browser using JavaScript.

Search can be accessed at path `/search` (theme provided content page), or by adding the `search` shortcode in a page.
