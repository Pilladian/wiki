[Back to Kubernetes](../README.md)

# Templates
> Templates of applications deployed on a Kubernetes Cluster

## Content
- [Assumptions](#assumptions)
- [Common static Website](#common-static-website)

<p id="assumptions">

### Assumptions
- Ingress Controller up and running - [Guide](../setup/k3s/ingress-controller.md)
- Kubectl tool installed and configured - [Guide](tbd)

<p id="common-static-website">

## Common static Website

### Prerequisites
Create a new namespace `NEW-NAMESPACE` on the cluster
```bash
kd create ns NEW-NAMESPACE
```

### Nginx Configuration
Create a new nginx configuration file `WEB-APP.conf` inside a new created directory `conf.d`
```bash
$ ls
conf.d

$ ls conf.d/
WEB-APP.conf
```
Content of `WEB-APP.conf`. Important here is the `/` at the end of the path
```bash
server {
	root /PATH/INSIDE/THE/POD/WHERE/THE/HOST-DATA/WILL/BE/MOUNTED/;
}
```

Create a new configmap `WEB-APP-configmap` on the cluster. Note that here the directory `conf.d` is provided. Not the file inside the directory.
```bash
kd -n NEW-NAMESPACE create configmap WEB-APP-configmap --from-file=conf.d
```

### Web Application
Create new file called `web-app.yaml` with the following content. 
```yaml
# Deployment Object for the Web App
apiVersion: apps/v1
kind: Deployment
metadata:
  # name of the web application
  name: WEB-APP-deployment
  # namespace where the deployment will be deployed
  namespace: NEW-NAMESPACE
spec:
  # amount of pods that are spawned
  replicas: 2
  selector:
    matchLabels:
      app: WEB-APP-nginx
  template:
    metadata:
      labels:
        app: WEB-APP-nginx
    spec:
      containers:
        # name of the pod
        - name: WEB-APP-nginx
          # image that will be used
          image: nginx:stable
          imagePullPolicy: Always
          volumeMounts:
            # volume of the static web content located on the host system
            - name: data
              mountPath: /PATH/INSIDE/THE/POD/WHERE/THE/HOST-DATA/WILL/BE/MOUNTED
            # volume of the nginx configuration that needs to be passed to the pod
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
          ports:
            # exposed container port
            - containerPort: 80
              # name of the port-exposion
              name: WEB-APP-port
              protocol: TCP
          # spawns a new pod if the old one crashes
          readinessProbe: 
            httpGet:
              port: WEB-APP-port
              path: /
      volumes:
        # create the volume for the static website content
        - name: data
          hostPath: 
            path: /PATH/TO/CONTENT/ON/HOST-SYSTEM
        # create the volume for the nginx configmap
        - name: nginx-config
          configMap:
            name: WEB-APP-configmap
---
# Service Object for exposing the Web App in the Cluster
apiVersion: v1
kind: Service
metadata:
  name: WEB-APP-service
  namespace: NEW-NAMESPACE
spec:
  selector:
    app: WEB-APP-nginx
  type: ClusterIP
  ports:
    - name: http
      port: 80
---
# Ingress Object for exposing the Web App to traefik
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: WEB-APP-ingress
  namespace: NEW-NAMESPACE
  annotations:
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec: 
  rules:
    - host: "SUB.SUB.YOURDOMAIN.COM"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: 
                name: WEB-APP-service
                port: 
                  number: 80
  tls:
  - hosts:
    - SUB.SUB.YOURDOMAIN.COM
    secretName: SUB-SUB-YOURDOMAIN-COM-tls
```

Deploy the `web-app.yaml` on the cluster
```bash
kd -n NEW-NAMESPACE apply -f web-app.yaml
```

Visiting `http://SUB.SUB.YOURDOMAIN.COM` should now result in a redirect to `https://SUB.SUB.YOURDOMAIN.COM` which presents you your static content.