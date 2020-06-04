# cert-manager-nginx-ingress
#### Securing nginx-ingress using cert-manager 
Example on EKS enviroment
git clone https://github.com/dzungda/cert-manager-nginx-ingress.git

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


## Setting Up the Example Backend
We will define a hello-world backend service and deployment
```hcl
kubectl apply -f hello-deployment.yaml
```
## Assign a DNS name 
The external IP that is allocated to the ingress-controller is the ELB endpoint to which all incoming traffic should be routed. To enable this, add it to a DNS zone you control
In this example I have DNS named : `elb.giaingay.io`

## Deploy Cert-Manager
We need to install cert-manager to do the work with Kubernetes to request a certificate and respond to the challenge to validate it.
All resources (the CustomResourceDefinitions, cert-manager, namespace, and the webhook component) are included in a single YAML manifest file
```hcl
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.yaml
```

## Configure Let’s Encrypt Issuer


Deploy a TLS Ingress Resource 
