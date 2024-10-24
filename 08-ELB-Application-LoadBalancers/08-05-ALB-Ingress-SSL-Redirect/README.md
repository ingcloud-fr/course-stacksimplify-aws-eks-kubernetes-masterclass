---
title: AWS Load Balancer - Ingress SSL HTTP to HTTPS Redirect
description: Learn AWS Load Balancer - Ingress SSL HTTP to HTTPS Redirect
---

## Step-01: Add annotations related to SSL Redirect
- **File Name:** 04-ALB-Ingress-SSL-Redirect.yml

On ajoute aux annotations :

- Redirect from HTTP to HTTPS
```yaml
    # SSL Redirect Setting
    alb.ingress.kubernetes.io/ssl-redirect: '443'   
```

## Step-02: Deploy all manifests and test

### Deploy and Verify

```
$ cd ../08-05-ALB-Ingress-SSL-Redirect/
```

Si rien n'est déjà déployé :

```t
# Deploy kube-manifests
$ kubectl apply -f kube-manifests/
```

Sinon :

```t
# Deploy ONLY 04-ALB-Ingress-SSL-Redirect.yml
$ kubectl apply -f kube-manifests/04-ALB-Ingress-SSL-Redirect.yml
ingress.networking.k8s.io/ingress-ssl-demo configured
```
On vérifie :

```t
# Verify Ingress Resource
$ kubectl get ingress
NAME               CLASS                  HOSTS   ADDRESS                                             PORTS   AGE
ingress-ssl-demo   my-aws-ingress-class   *       ssl-ingress-606772891.eu-west-3.elb.amazonaws.com   80      80m

$ kubectl describe ingress ingress-ssl-demo
Annotations:  ...
              alb.ingress.kubernetes.io/ssl-redirect: 443
            ...
```

On vérifie les pods de l'app :

```t

# Verify Apps
$ kubectl get deploy


$ kubectl get pods

# Verify NodePort Services
kubectl get svc
```
### Verify Load Balancer & Target Groups
- Load Balancer -  Listeneres (Verify both 80 & 443) 
- Load Balancer - Rules (Verify both 80 & 443 listeners) 
- Target Groups - Group Details (Verify Health check path)
- Target Groups - Targets (Verify all 3 targets are healthy)
 
## Step-03: Access Application using newly registered DNS Name
- **Access Application**
```t
# HTTP URLs (Should Redirect to HTTPS)
http://my-app-test.aws.ingcloud.eu/app1/index.html
http://my-app-test.aws.ingcloud.eu/app2/index.html
http://my-app-test.aws.ingcloud.eu

# HTTPS URLs
https://my-app-test.aws.ingcloud.eu/app1/index.html
https://my-app-test.aws.ingcloud.eu/app2/index.html
https://my-app-test.aws.ingcloud.eu
```

Les redirections vers HTTPS sont ok !


## Step-04: Clean Up
```t
# Delete Manifests
$ kubectl delete -f kube-manifests/04-ALB-Ingress-SSL-Redirect.yml 
ingress.networking.k8s.io "ingress-ssl-demo" deleted

$ kubectl delete -f kube-manifests/
ingressclass.networking.k8s.io "my-aws-ingress-class" deleted
deployment.apps "app1-nginx-deployment" deleted
service "app1-nginx-nodeport-service" deleted
deployment.apps "app2-nginx-deployment" deleted
service "app2-nginx-nodeport-service" deleted
deployment.apps "app3-nginx-deployment" deleted
service "app3-nginx-nodeport-service" deleted
Error from server (NotFound): error when deleting "kube-manifests/04-ALB-Ingress-SSL-Redirect.yml": ingresses.networking.k8s.io "ingress-ssl-demo" not found

## Delete Route53 Record Set
- Delete Route53 Record we created 
```

## Annotation Reference
- [AWS Load Balancer Controller Annotation Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)



