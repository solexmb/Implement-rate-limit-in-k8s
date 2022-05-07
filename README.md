# Protect Kubernetes APIs by Implementing Rate Limiting

This project demonstrates how to use multiple NGINX Ingress Controllers combined with rate limiting to prevent apps and APIs from getting overwhelmed. ( If the API receives a high volume of HTTP requests – a possibility with brute‑force password guessing or DDoS attacks – then both the API and app might be overwhelmed or even crash.)

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on aws.

### Prerequisites

This tutorial uses these technologies:

```
Kubernetes Cluster
```
```
NGINX Ingress Controller (based on NGINX Open Source)
```
```
Helm
```
```
Locust
```
```
Podinfo 
```

### This project includes three challenges:

```
Deploy an App, API, and Ingress Controller 
```
```
Overwhelm Your App and API 
```
```
Save Your App and API with Dual Ingress Controller and Rate Limiting 
```

## Challenge 1: Deploy the App, API, and Ingress Controller

Podinfo is a “web application made with Go that showcases best practices of running microservices in Kubernetes”. We’re using it as a sample app and API because of its small footprint.
https://github.com/stefanprodan/podinfo

```
Clone the repo
```
```
Deploy the app and API
$ kubectl apply -f 1-podinfo-apps.yaml
```
```
Confirm that the Podinfo pods deployed, as indicated by the value Running in the STATUS column
$ kubectl get pods
```

### Install NGINX Ingress Controller in a separate namespace (“nginx”) using Helm

```
Create the namespace
$ kubectl create namespace nginx
```
```
Add the NGINX repository to Helm
$ helm repo add nginx-stable https://helm.nginx.com/stable 
```
```
Download and install NGINX Ingress Controller in your cluster:
$ helm install main nginx-stable/nginx-ingress \ 
 --set controller.watchIngressWithoutClass=true \ 
 --set controller.ingressClass=nginx \ 
 --set controller.service.type=NodePort \ 
 --set controller.service.httpPort.nodePort=30010 \ 
 --set controller.enablePreviewPolicies=true \ 
 --namespace nginx
```
```
Confirm that the NGINX Ingress Controller pod deployed, as indicated by the value Running in the STATUS column
$ kubectl get pods –namespace nginx 
```

### Route Traffic to Your App

```
Deploy the Ingress resource:
$ kubectl apply -f 2-nginx-ingress.yaml
```

### Test the Ingress Configuration

Create a temporary pod
```
$ kubectl run -ti --rm=true busybox --image=busybox
```
Test the Podinfo API:

```
/ # wget --header="Host: api.example.com" -qO- main-nginx-ingress.nginx
```

Test Podinfo Frontend

```
/ # wget --header="Host: example.com" --header="User-Agent: Mozilla" -qO- main-nginx-ingress.nginx
```

### Port-forward the podinfo service 

```
$ kubectl port-forward [podinfo service] [hostPort:servicePort] &
copy and paste the the address to a browser or if you are using a loadbalancer service type, just copy the loadbalancer ip to your browser
```


## Challenge 2: Overwhelm Your App and API

Install Locust which serves as the traffic generator

```
$ kubectl apply -f  3-locust.yaml 
```

Confirm Locust deployment
```
$ kubectl get pod
```

### Simulate a Traffic Surge and Observe the Effect on Performance

```
$ kubectl port-forward [locust service] [hostPort:servicePort] &
copy and paste the the address to a browser or if you are using a loadbalancer service type, just copy the loadbalancer ip to your browser
```

Enter the following values in the fields:

```
Number of users – 1000
Spawn rate – 30
Host – http://main-nginx-ingress
Click the Start swarming button to send traffic to the Podinfo app and API 
```

Charts Tab: This tab provides a graphical depiction of the traffic. As API requests increase, watch your Podinfo API response times worsen.

Failures Tab: Because your web app and API share an Ingress controller, notice the web app soon returns errors and fails as a result of the increased API requests.


### Challenge 3: Save Your App and API with Dual Ingress Controller and Rate Limiting

Rate limiting restricts the number of requests a user can make in a given time period. When under a DDoS attack, for example, you can use rate limiting to limit the incoming request rate to a value typical for real users. When rate limiting is implemented with NGINX, any clients who submit too many requests will get redirected to an error page so they cannot negatively impact the API

An API gateway routes API requests from a client to the appropriate services

### Reconfigure the Cluster

```
Delete the Ingress definition:
$ kubectl delete -f 2-ingress.yaml
```

```
Create two namespaces:
$ kubectl create namespace nginx-web
$ kubectl create namespace nginx-api
```

### Install the nginx-web NGINX Ingress Controller

```
helm install web nginx-stable/nginx-ingress  
  --set controller.ingressClass=nginx-web \ 
  --set controller.service.type=NodePort \ 
  --set controller.service.httpPort.nodePort=30020 \ 
  --namespace nginx-web 
```

```
Deploy the new manifest:
$ kubectl apply -f 4-ingress-web.yaml
```

### Install the nginx-api NGINX Ingress Controller

```
helm install api nginx-stable/nginx-ingress  
  --set controller.ingressClass=nginx-api \ 
  --set controller.service.type=NodePort \ 
  --set controller.service.httpPort.nodePort=30030 \ 
  --set controller.enablePreviewPolicies=true \ 
  --namespace nginx-api 
```

```
Deploy the new manifest:
$ kubectl apply -f 5-ingress-api.yaml
```

### Reconfigure Locust
Change the Locust script so that:

```
All the requests to the web app go through nginx-web (http://web-nginx-ingress.nginx-web)
All API requests go to nginx-api (http://web-nginx-ingress.nginx-web)
```

```
Implement the script change:
$ kubectl apply -f 6-locust.yaml
```
### Force a reload:

Delete the Locust pods to force a reload of the new config map. The following command simultaneously retrieves your pods and deletes the Locust pod

```
$ kubectl delete pod `kubectl get pods | grep locust | awk {'print $1'}`
```

### Test Rate Limiting

Return to Locust and change the parameters:

```
Number of users – 400
Spawn rate – 10
Host – http://main-nginx-ingress
```

Click the Start swarming button to send traffic to the Podinfo app and API.

You will see that as the number of users starts to climb, so does the error rate. However, when you click on failures in the Locust UI, you will see that these are no longer the Web frontend, but are coming from the API service. Also notice that NGINX is serving back a ‘Service Temporarily Unavailable’ message. This is part of the rate limiting feature and it can be customized. The API is rate limited, and the web application is always available. 


## Acknowledgments

* Journey to DevOps/SRE 
* Nginx Microservice March

