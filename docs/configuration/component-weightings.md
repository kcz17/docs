# Component Weightings

If you have not yet configured Kubedim for baseline dimming, follow the steps
[here](configuration/baseline.md).

Adding weightings to the configuration is straightforward. Using our
example as before:

```yaml
# ...
      dimmableComponents:
        - path: "recommender"
          method:
            shouldMatchAll: true
          probability: 0.5
        - path: "news"
          method:
            shouldMatchAll: true
          probability: 0.5
        - path: "cart"
          method:
            method: "GET"
          exclusions:
            - method: "GET"
              substring: "basket.html"
          probability: 0.98
# ...
```
We add `probability` fields for every component, with a value between 0 and 1.
A probability of value 0 leads to a component never being dimmed; a probability
of value 1 means a component is always dimmed. (Baseline dimming is equivalent
to the component weightings strategy with all probabilities set to 1.)

However, it can be unclear which would be the best weightings to use. As a
result, Kubedim comes with an offline training tool which can automatically
determine the best weightings.

## Offline Training

In order to train a set of component weightings, it is necessary to link the
training tool with a load testing script which has been set up to simulate
production behaviour.

The tool randomly samples a set of component weightings and sets different
weightings over a number of iterations, building a model of how the system
responds to different weightings.

Currently, the only supported load testing driver is
[k6](https://k6.io/). Your k6 script must be configured to run with an 
`externally-controlled` executor so it can be controlled by the offline training
tool.

To set up and run the offline training tool, follow the following steps.

First, download the offline training tool from https://github.com/kcz17/train,
building the binary using `go build`.  Next, create a `config.yaml` file, 
filling out the configuration file as follows:

```yaml
endpoints:
  targetHost: "[Kubedim IP address]"
  targetPort: 30002 # Kubedim's reverse proxy port.
  dimmerAdminHost: "[Kubedim IP address]"
  dimmerAdminPort: 30003 # Kubedim's admin port.
dimmableComponentPaths: # The optional component paths, matching the ConfigMap.
  - "recommender"
  - "news"
  - "cart"
loadGenerator: # Details of your load generation script.
  driver: "k6"
  k6:
    host: "localhost"
    port: "6565"
loadProfile: # Details of the load generation shape.
  # numIterations is the number of different weightings to sample. A greater
  # number will lead to a more accurate model.
  numIterations: 300 
  maxUsers: 300
  rampUpSeconds: 10
  peakSeconds: 50
  rampDownSeconds: 10
  secondsBetweenRuns: 10
```

Next, start the k6 load generation script, which will listen for instructions
from the offline training tool. Then, run the training tool using `./train`, and
wait for its completion.

Kubedim will output a list of priorities to use, which can be set in the 
`probability` fields in Kubedim's ConfigMap.
