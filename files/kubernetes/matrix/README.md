[Back to Kubernetes](../README.md)

# Setup your own Matrix Synapse Server

## Content
- [Prerequisites](#pre)
- [Installation](#installation)
- [Backup](#backup)

<p id="pre">

## Prerequisites
- K3S Kubernetes Cluster - [Guide](../setup/k3s/README.md)
- Ingress Controller - [Guide](../setup/k3s/ingress-controller.md)
- Kubectl - [Guide](tbd)

<p id="installation">

## Installation
All configurations will be in one `matrix-synapse.yaml` file at the end.
The used aliases for `kubectl` can be found [here](../aliases.md).

### Prepare Cluster
```bash
# Create new namespace on cluster and make it our working namespace
kd create ns matrix-synapse
kn matrix-synapse
```

### Main Server Deployment
As our `image` we use the latest `python` container. 
We then install all requirements for matrix, create a python3 virtual environment and create the initial configuration files. Since these files are stored in the `/synapse` directory inside the container and this directory is a mounted volume, we can see all created configuration files on the host at `~/matrix-synapse`. Make sure to change `matrix.local` to your domain the server will be accessible in the future. Something like `matrix.yourdomain.com`.
```yaml
# Deployment of matrix-synapse
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: matrix-synapse
  name: matrix-synapse
  namespace: matrix-synapse
spec:
  replicas: 1
  selector:
    matchLabels:
      app: matrix-synapse
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: matrix-synapse
    spec:
      automountServiceAccountToken: false
      nodeSelector:
        kubernetes.io/hostname: cluster-master
      containers:
        - image: python:latest
          name: matrix-synapse
          # Create init files
          command: ["/bin/bash", "-c", "touch /tmp/healthy && apt update && apt install -y build-essential python3-dev libffi-dev python3-pip python3-setuptools sqlite3 libssl-dev virtualenv libjpeg-dev libxslt1-dev && mkdir /env && virtualenv -p python3 /env && source /env/bin/activate && pip install --upgrade pip && pip install --upgrade setuptools && pip install matrix-synapse && cd /synapse && python -m synapse.app.homeserver --server-name matrix.local --config-path homeserver.yaml --generate-config --report-stats=yes && sleep 86400"]
          # Normal deployment command
          #command: ["/bin/bash", "-c", "touch /tmp/healthy && apt update && apt install -y build-essential python3-dev libffi-dev python3-pip python3-setuptools sqlite3 libssl-dev virtualenv libjpeg-dev libxslt1-dev && mkdir /env && virtualenv -p python3 /env && source /env/bin/activate && pip install --upgrade pip && pip install --upgrade setuptools && pip install matrix-synapse && pip install matrix_client && cd /synapse && synctl start && sleep 90000"]
          volumeMounts:
            - name: data
              mountPath: /synapse
          ports:
            - containerPort: 8008
              name: matrix-web
              protocol: TCP
          livenessProbe:
            exec:
              command:
                - cat
                - /tmp/healthy
            initialDelaySeconds: 30
            periodSeconds: 5
      volumes:
        - name: data
          hostPath: 
            path: ~/matrix-synapse

```
We now deploy `matrix-synapse.yaml` the first time, creating all our needed configuration files.
```bash
kd apply -f matrix-synapse.yaml
```

In the host directory `~/matrix-synapser` we now see the initial files, one of them being `homeserver.yaml`.
We now need to change a configuration, such that the server handles request from outside of it's own subnet.
Therefor find the string `"bind_addresses: ['::1', '127.0.0.1']"` under the `port: 8008` section and comment it out. 
Save the file and exit the editor.
As last step change the `command` section in `matrix-synapse.yaml` such that it looks like the following:

```yaml
# Deployment of matrix-synapse
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: matrix-synapse
  name: matrix-synapse
  namespace: matrix-synapse
spec:
  replicas: 1
  selector:
    matchLabels:
      app: matrix-synapse
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: matrix-synapse
    spec:
      automountServiceAccountToken: false
      nodeSelector:
        kubernetes.io/hostname: cluster-master
      containers:
        - image: python:latest
          name: matrix-synapse
          # Create init files
          #command: ["/bin/bash", "-c", "touch /tmp/healthy && apt update && apt install -y build-essential python3-dev libffi-dev python3-pip python3-setuptools sqlite3 libssl-dev virtualenv libjpeg-dev libxslt1-dev && mkdir /env && virtualenv -p python3 /env && source /env/bin/activate && pip install --upgrade pip && pip install --upgrade setuptools && pip install matrix-synapse && cd /synapse && python -m synapse.app.homeserver --server-name matrix.local --config-path homeserver.yaml --generate-config --report-stats=yes && sleep 86400"]
          # Normal deployment command
          command: ["/bin/bash", "-c", "touch /tmp/healthy && apt update && apt install -y build-essential python3-dev libffi-dev python3-pip python3-setuptools sqlite3 libssl-dev virtualenv libjpeg-dev libxslt1-dev && mkdir /env && virtualenv -p python3 /env && source /env/bin/activate && pip install --upgrade pip && pip install --upgrade setuptools && pip install matrix-synapse && pip install matrix_client && cd /synapse && synctl start && sleep 90000"]
          volumeMounts:
            - name: data
              mountPath: /synapse
          ports:
            - containerPort: 8008
              name: matrix-web
              protocol: TCP
          livenessProbe:
            exec:
              command:
                - cat
                - /tmp/healthy
            initialDelaySeconds: 30
            periodSeconds: 5
      volumes:
        - name: data
          hostPath: 
            path: ~/matrix-synapse
```
Since the server exposes port `8008` to the local cluster, we need a service such that our ingress controller is able to reach it. 
We therefor create a `Service` in the same file:
```yaml
# Deployment stuff 
---
# Service for matrix-syanpse
apiVersion: v1
kind: Service
metadata:
  name: ms-web
  namespace: matrix-synapse
spec:
  selector:
    app: matrix-synapse
  type: ClusterIP
  ports:
    - name: matrix-web
      port: 8008
```

Since our matrix server should be able to be find by other instances, need a second deployment. 
This will be a nginx web server that presents only one file.
For this we need to create a new directory and place a nginx-configuration file inside:

```bash
mkdir config-dir
vim config-dir/matrix.yourdomain.com.conf
```

Now paste the following content inside that file and save it:
```nginx
server {
        root /delegation/;
}
```

We now need to create a `configmap` in our cluster. 
This is something like a volume but only containing one file. 
Making it perfect for mounting single files in pods.
```bash
kd -n matrix-synapse create configmap nginx-configmap --from-file=config-dir
```

After that we can now add the `Deployment` and the `Service` for our delegation pod to `matrix-synapse.yaml`
```yaml
# Matrix Deployment stuff
# Matrix Service stuff
---
# Deployment for delegation nginx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delegation
  namespace: matrix-synapse
  labels:
    app: delegation
spec:
  replicas: 1
  selector:
    matchLabels:
      app: delegation
  template:
    metadata:
      labels:
        app: delegation
    spec:
      automountServiceAccountToken: false
      nodeSelector:
        kubernetes.io/hostname: cluster-master
      containers:
        - name: delegation
          image: nginx:stable
          imagePullPolicy: Always
          volumeMounts:
            - name: delegation-files
              mountPath: /delegation
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
          ports:
            - containerPort: 80
              name: delegation-p
              protocol: TCP
          readinessProbe: 
            httpGet:
              port: delegation-p
              path: /.well-known/matrix/server
      volumes:
        - name: delegation-files
          hostPath:
            path: /root/cluster/cluster-config/matrix-synapse/delegation
        - name: nginx-config
          configMap:
            name: nginx-configmap