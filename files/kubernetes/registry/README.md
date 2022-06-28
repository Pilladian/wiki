[Back to Kubernetes](../README.md)

# Setup your personal Registry

`Note:` This registry will not store your images permanent. When the docker container, the registry is running in, gets terminated, the content will be gone. 

## Prerequisites
- K3S Kubernetes Cluster - [Guide](../setup/k3s/README.md)
- Kubectl - [Guide](tbd)
- Docker installed on your local desktop

## Create Registry
We will create a local registry on our desktop.
```bash
docker run -d -p 5000:5000 --restart=always --name registry registry
```
Since the registry exposes port `5000`, this needs to be added to your local running firewall, since your cluster needs to access it. In my case I use `ufw` to set and manage the firewall rules:
```bash
ufw allow from <CLUSTER-MASTER> to any port 5000
ufw allow from <CLUSTER-WORKER-01> to any port 5000
...
```

## Build Images using Registry
On your local machine you can now build images using the following command: 
```bash
docker build -t localhost:5000/<image>:<version> -f Dockerfile .
```

## Push Images to Registry
To push your locally built image use:
```bash
docker push localhost:5000/<image>:<version>
```

If you want to push your image that was built on another machine, We first need to tell docker that it is okay to push to our insecure registry (insecure, since we don't use https). We therefor need to edit/create `/etc/docker/daemon.json` on our machine and put the following content:
```json
{
  "insecure-registries" : ["<DESKTOP-IP>:5000"]
}
```
After saving the file restart the docker service and you're good to go!
```bash
# restart docker
systemctl restart docker.service

# push image to registry
docker push <DESKTOP-IP>:5000/<image>:<version>
```

## Configure K3S to use Registry
We assume that you don't have any registry registered yet. We therefor need to create a new file at `/etc/rancher/k3s/registries.yaml` and put in the following content:
```yaml
mirrors: 
  "<DESKTOP-IP>":
    endpoint:
      - "http://<DESKTOP-IP>:5000"
```
When we assume that our local registry is running on our desktop and has the IP `10.1.1.10` then our file would look like this:
```yaml
mirrors: 
  "10.1.1.10":
    endpoint:
      - "http://10.1.1.10:5000"
```
We now need to restart `k3s.service` to apply the changes:
```bash
systemctl restart k3s.service
```

## Use Personal Registry in .yaml Files
We now can access our personal registry and pull images from it. Please have a look at this example:
```yaml
# Deployment for example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
  namespace: example
  labels:
    app: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      automountServiceAccountToken: false
      containers:
        - name: example
          image: 10.1.1.10/example:latest 
```