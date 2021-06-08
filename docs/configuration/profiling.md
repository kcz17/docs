# Profiling

If you have not yet configured Kubedim for baseline dimming, follow the steps
[here](configuration/baseline.md).

## Configuring InfluxDB

Profiling requires an external InfluxDB instance to be configured and visible to
the cluster, as it will be used as a data store used by Kubedim's profiler to
assign priorities to users.

Ensure you provision the following:

- An organisation for Kubedim
- A token which has access to this organisation
- A bucket to store sessions (e.g., `session_history`)
- A bucket to store logs (e.g., `profiling_log`)

## Deploying the Priority Database

Kubedim's profiler must write to a store which can be quickly retrieved by
Kubedim. We use Redis for this.

### Deployment

First, we create a new deployment manifest for the profiler database.

```yaml
# Suggested path: kubedim/deployments/profiler-db.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: profiler-db
  labels:
    name: profiler-db
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: profiler-db
  template:
    metadata:
      labels:
        name: profiler-db
    spec:
      containers:
        - name: profiler-db
          image: redis:alpine
          ports:
            - name: redis
              containerPort: 6379
          securityContext:
            capabilities:
              drop:
                - all
              add:
                - CHOWN
                - SETGID
                - SETUID
            readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
```

### Service

Next, we create a service manifest for the profiler database.

```yaml
# Suggested path: kubedim/services/profiler-db.yaml

apiVersion: v1
kind: Service
metadata:
  name: profiler-db
  labels:
    name: profiler-db
  namespace: my-app
spec:
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    name: profiler-db
```

## Scaffolding the Profiler

Kubedim's profiler is deployed as a separate service, so the appropriate
manifest files must first be scaffolded as follows.

### Deployment

First, we add a new deployment manifest for the profiler.

```yaml
# Suggested path: kubedim/deployments/profiler.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: profiler
  labels:
    name: profiler
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: profiler
  template:
    metadata:
      labels:
        name: profiler
    spec:
      containers:
        - name: profiler
          image: kcz17/profiler:0.0.15
          ports:
            # Necessary to expose the profiler to the main dimmer service.
            - containerPort: 8080
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            capabilities:
              drop:
                - all
              add:
                - NET_BIND_SERVICE
            readOnlyRootFilesystem: true
          volumeMounts:
            - name: config-volume
              mountPath: /app/config.yaml
              subPath: config.yaml
          env:
            # We do not want to commit a sensitive token to version control,
            # so we will instead set the InflxuDB token configuration value via
            # a Kubernetes secret.
            - name: CONNECTIONS_INFLUXDB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: profiler-influxdb
                  key: token
      volumes:
        - name: config-volume
          configMap:
            # Points to configuration for the profiler.
            name: profiler-config
      nodeSelector:
        beta.kubernetes.io/os: linux
```

Instead of adding the InfluxDB token in a configuration file, make it available
to the container using secret, by running the following:
``kubectl create secret generic profiler-influxdb --from-literal=token='[TOKEN]' -nmy-app``

### Service

```yaml
# Suggested path: kubedim/services/profiler.yaml

apiVersion: v1
kind: Service
metadata:
  name: profiler
  labels:
    name: profiler
  namespace: my-app
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    name: profiler
```

### ConfigMap

```yaml
# Suggested path: kubedim/configmaps/profiler.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: profiler-config
  namespace: my-app
data:
  config.yaml: |
    connections:
      redis:
        addr: "profiler-db:6379"
        pass: ""
        storeDB: 0
        queueDB: 1
      influxdb:
        addr: "http://[InfluxDB IP]:[InfluxDB Port]"
        # The token should be set as a secret, separate from the ConfigMap.
        org: "[Bucket Organisation]"
        sessionBucket: "session_history" # Adjust this if your bucket name is different.
        loggingBucket: "profiling_log" # Adjust htis if your bucket name is different.
    profiling:
      # interval between fetching sessions to be profiled in seconds.
      interval: 10
    rules:
      # We will fill this out later.
```

## Configuring Profiler Rules

An example of profiler rules in the above ConfigMap is as follows:

```yaml
# ...
    rules:
      - description: "User has checked out items in past"
        path: "/cart"
        method:
          shouldMatchAll: false
          method: "POST"
        occurrences: 1
        result: "high"
      - description: "User is browsing and unlikely to buy"
        path: "/category.html"
        method:
          shouldMatchAll: true
        occurrences: 5
        result: "low"
      - description: "User is browsing for delivery updates"
        path: "/news"
        method:
          shouldMatchAll: true
        occurrences: 1
        result: "low"
# ...
```

The profiler works by running through the rules in sequential order, counting
the number of times a session has encountered each path-method combination.
If the number of occurrences is met or exceeded when testing a rule, then the
resulting priority is assigned to the user and no further rules are tested.

## Connecting Kubedim to the Profiler

To connect Kubedim to the profiler, we update the Kubedim ConfigMap, adding
the following section nested under the `dimming` attribute.

```yaml
    dimming:
      enabled: true
      # ...
      profiler:
        enabled: true
        # Session cookie is the name of the cookie which identifies unique
        # user sessions. This should be changed to the session cookie key.
        sessionCookie: "md.sid"
        influxdb:
          addr: "http://[InfluxDB Host]:[InfluxDB Port]]"
          # Recall we have set the profiler token as an environment variable.
          org: "[InfluxDB organisation]"
          bucket: "session_history" # Change this where appropriate.
        redis:
          addr: "profiler-db:6379"
          password: "" # The default Redis image does not use a password.
          # We use different databases to store priorities and the queue.
          prioritiesDB: 0 
          queueDB: 1
```
