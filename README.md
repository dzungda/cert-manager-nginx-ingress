# cert-manager-nginx-ingress
#### Securing nginx-ingress using cert-manager 
Example on EKS enviroment
#
## Deploy nginx ingress controller
Run the following command to create the nginx-ingress-controller ingress controller deployment, along with the Kubernetes RBAC roles and bindings:
```hcl
kubectl apply -f rbac.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress-controller.yaml
```

The service of ingress-controller that used `LoadBalancer` type will expose a ELB coresponding 

```hcl
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
```
We can verify by this command
```hcl
$kubectl get svc -n ingress-nginx
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.100.42.179   a9ca4c35476394e80b5b74948eb28c82-853bc8bd128406bc.elb.ap-southeast-1.amazonaws.com   80:32141/TCP,443:30935/TCP   14m
```
#

## Setting Up the Example Backend
We will define a hello-world backend service and deployment
```hcl
kubectl apply -f hello-deployment.yaml
```
#
## Assign a DNS name 
The external IP that is allocated to the ingress-controller is the ELB endpoint to which all incoming traffic should be routed. To enable this, add it to a DNS zone you control
In this example I have DNS named : `dungda.xyz`
#
## Deploy Cert-Manager
We need to install cert-manager to do the work with Kubernetes to request a certificate and respond to the challenge to validate it.
All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file
```hcl
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.yaml
```
#
## Configure Let’s Encrypt Issuer
An Issuer is the definition for where cert-manager will get request TLS certificates. An `Issuer` is specific to a single namespace in Kubernetes and `ClusterIssuer` resources apply across all Ingress resources in your cluster and don’t have this namespace-matching requirement

```hcl
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
       # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
    email: dungdaptit@gmail.com
       # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
       # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

Once edited, apply the custom resource:
```hcl
kubectl apply -f issuer.yaml
```

#
## Deploy a TLS Ingress Resource 
Add `cert-manager.io/issuer` on annotations and config as below

```hcl
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world-ing
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - elb.giaingay.io
    secretName: tls-secret
  rules:
  - host: dungda.xyz
    http:
      paths:
      - backend:
          serviceName: docker-hello-world-svc
          servicePort: 8088
```
and apply 
```hcl
kubectl apply -f ingress.yaml
```
Cert-manager will read these annotations and use them to create a certificate, which you can request and see:
```hcl
$ kubectl get certificate
NAME         READY   SECRET       AGE
tls-secret   True    tls-secret   125m
```

## Testing
```hcl
$ curl -k https://dungda.xyz
<h1>Hello webhook world from: docker-hello-world-65fb557b9f-7ks9w</h1>
```
