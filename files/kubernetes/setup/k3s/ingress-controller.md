[Back to k3s Setup](./README.md)

# Setup an Ingress Controller for k3s

## Installing cert-manager
Create a new namespace called `cert-manager` and make it your default working namespace
```bash
kd create ns cert-manager
kn cert-manager
```

Deploy the manifest 
```bash
kd apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
```

## Staging
Create a staging certificate issuer `letsencrypt-issuer-staging.yaml` with the following content
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: YOUR_EMAIL_GOES_HERE
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: traefik
```

Create a staging certificate `le-test-certificate.yaml` to test the current configuration
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-certificate
  namespace: default
spec:
  secretName: test-domain-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: YOUR_DOMAIN_GOES_HERE
  dnsNames:
  - YOUR_DOMAIN_GOES_HERE
```

Deploy the issuer and issue the certificate
```bash
kd apply -f letsencrypt-issuer-staging.yaml
kd apply -f le-test-certificate.yaml
```

Check for the status of the certificate, but note, that it can take a while to get it issued
```bash
kd -n default get certificates
kd -n default describe certivicate test-certificate
```

Clean up the staging issuer and certificate
```bash
kd -n default delete certificate test-certificate
kd -n default delete secret test-domain-tls
kd delete clusterissuer letsencrypt-staging
```

## Production
Create the production issuer `letsencrypt-issuer-prod.yaml`, which is very similar to the staging one
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: YOUR_EMAIL_GOES_HERE
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: traefik
```

Deploy the production issuer
```bash
kd apply -f letsencrypt-issuer-prod.yaml
```

The pods now need to be configured to automatically use the certificate issuer to request a valid certificate. A guide of how to deploy a web application using this ingress controller can be found [here](../../templates/README.md#common-static-website).