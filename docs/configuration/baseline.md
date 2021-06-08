# Baseline Dimming

Before configuring any advanced strategies, you must first configure Kubedim
for baseline dimming.

Configuration is straightforward: modifying the ConfigMap originally provided
from the Getting Started section, add the HTTP paths which represent the entry
points of the application. An example is shown as follows:

```yaml
# ...
    dimming:
      enabled: true
      dimmableComponents:
        - path: "recommender"
          method:
            shouldMatchAll: true
        - path: "news"
          method:
            shouldMatchAll: true
        - path: "cart"
          method:
            method: "GET"
          exclusions:
            - method: "GET"
              substring: "basket.html"
# ...
```

The `method` attribute must be present on each component:
- By setting `shouldMatchAll` to `true`, Kubedim will match the optional
  components on all HTTP requests to the `recommender` path.
- By setting a `method` attribute under `method`, with value as one of the HTTP
  methods (e.g., `GET`, `POST`, `PUT`, etc.), only calls to the path with the
  specified method will be dimmed.

The `exclusions` attribute allows specified referrers to be excluded from
matching. One typical case is when components are optional when called from most
pages, but compulsory when called from a specific page. In the case above,
calls to `GET /cart` from referrers containing `basket.html` will not be
dimmable.
