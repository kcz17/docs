# Kubedim

Kubedim is a drop-in solution for Kubernetes to orchestrate intelligent load
shedding on optional components of a system in response to periods of high load.

In a system with optional components, Kubedim will automatically reject requests
when the system load is high. Kubedim characterises system load using either the
75th, 90th or 95th percentile response time: once the specified percentile
response time crosses a specified _setpoint_ (e.g., 3 seconds), Kubedim will
begin rejecting requests to optional components. The percentage of requests
rejected depends on the extent and duration of which response time surpasses
the setpoint.

## Prerequisites

- A Kubernetes cluster running v1.16 or later
- An [InfluxDB 2.0](https://docs.influxdata.com/influxdb/v2.0/get-started/) instance _(for profiling and logging)_ 

## Getting Started

### Scaffold Configuration

The core of Kubedim is composed of three components: a deployment, a service 
and, for configuration, a ConfigMap. We first scaffold dimming to enable reverse
proxying without causing brownout behaviour. We will then discuss and enable
brownout strategies later.

In the manifests for these components, we make the following assumptions, which
you should verify and change if appropriate:

- The namespace for the cloud application is `my-app`
- The dimmer reverse proxy and API are configured to listen on ports 8078 and 
  8079 respectively
- The dimmer reverse proxy and API endpoints are exposed outside the cluster on
  ports 8078 and 8079 respectively
- The front-end of our target application is made available within the cluster
  (i.e., via service) at `front-end:80`

#### Deployment

The deployment manifest declares that the cluster should create a single pod
with the dimmer binary.

```yaml
# Suggested path: kubedim/deployments/dimmer.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: dimmer
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: dimmer
  template:
    metadata:
      labels:
        name: dimmer
    spec:
      containers:
        - name: dimmer
          image: kcz17/dimmer:1.0.0
          ports:
            # Necessary to expose the reverse proxy via a service to the outside world
            - containerPort: 8078
            # Necessary to expose the admin endpoint via a service to the outside world
            - containerPort: 8079
          volumeMounts:
            - name: config-volume
              # The dimmer reads from this exact mount path, so it should not be changed
              mountPath: /app/config.yaml
              subPath: config.yaml
          env:
            # We will fill in the env section during later configuration steps
      volumes:
        - name: config-volume
          configMap:
            # Points to the ConfigMap
            name: dimmer-config
      nodeSelector:
        beta.kubernetes.io/os: linux
```

#### Service

This service configuration exposes the dimmer via a [`NodePort`](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)
configuration. If the dimmer reverse proxy runs on a cloud provider, a [`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
can be a feasible alternative.

In the unlikely use case that the dimmer endpoint should only be accessible
within a cluster (i.e., if Kubedim is placed behind a third party ingress
controller or a load balancing pod), then [`ClusterIP`](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
can be used.

```yaml
# Suggested path: kubedim/services/dimmer.yaml

apiVersion: v1
kind: Service
metadata:
  name: dimmer
  labels:
    name: dimmer
  namespace: my-app
spec:
  type: NodePort
  ports:
    - name: server
      port: 80
      # targetPort is the front-end port specified in deployment
      targetPort: 8078
      # nodePort is the port which the front-end will be accessible on via any node IP in the cluster
      nodePort: 30002
    - name: api
      port: 81
      # targetPort is the admin specified in the deployment
      targetPort: 8079
      # nodePort is the port which the admin endpoint will be accessible on in any node IP in the cluster
      nodePort: 30003
  selector:
    name: dimmer
```

#### Configuration

```yaml
# Suggested path: kubedim/configmaps/dimmer.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: dimmer-config
  namespace: my-app
data:
  config.yaml: |
    connection:
      # Instructs Kubedim to listen on the reverse proxy port we have provided.
      frontendPort: 8078
      # Instructs Kubedim to proxy requests to front-end:80
      backendHost: "front-end"
      backendPort: 80
      # Instructs Kubedim to listen on the admin port we have provided.
      adminPort: 8079
    logging:
      # We will not log any output for the time being.
      driver: "noop"
    # We do not enable dimming or any brownout strategies for the time being.
    dimming:
      enabled: false
      dimmableComponents:
        # We will specify dimmable components later.
      profiler:
        enabled: false
```

### Deployment and Testing

Deploy the files using `kubectl apply -f [manifest file path]`. If you have
followed the suggested paths, you can use `kubectl apply -R -f kubedim/`.

Kubedim will now be configured to proxy requests without any dimming. If you have
followed the instructions above without changes, your application will be
available via the reverse proxy at `http://[any node IP]:30002` .

### Next Steps

Visit the [Configuration](configuration.md) to learn how to enable logging,
enable dimming and configure brownout strategies.

## Uninstallation

Kubedim can be simply installed by running `kubectl delete -f [manifest file path]`
on the created manifest files. If you have followed the suggested paths, you can
use `kubectl delete -R -f kubedim/`.
