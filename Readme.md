

# README
## Service Design
Dockerized and used Java and Go applications with the names/tags of

- **Java**: mert/wr:java
- **Go**: mert/wr:go

Since we're focused on Canary Deployment by specifying weights for applications, I decided to use one service and two deployments. Components are as listed
- **1 Service**: wr
- **2 Deployments(Subsets)**: wr-go and wr-java
- **1 Virtual Service**: wr
- **1 Destination Rule**: wr
- **1 Ingress Gateway**: wr-gateway

I'd like to point out that Istio also supports weighted routing between two different services. But this is not the case here.
 
## Setup
This step includes the environment to implement the solution.
### Environment
- Windows 10 Host
- Ubuntu 18.04 LTS Guest running on Oracle VMBox with 
  - 4 Processors
  - 12 GiB Memory
  - 20 GiB VDI Storage
### Installed Tools on Ubuntu
- Minikube (v1.5.2)
- kubectl (Client: v1.16.3, Server: v1.15.2)
- Istio (v1.4.0)
### Operations
#### 1-) Initiliazing Minikube and Single Node Kubernetes Cluster

    $ sudo minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.15.2 --vm-driver none
    $ sudo mv /home/\$USER/.kube /home/\$USER/.minikube \$HOME
    $ sudo chown -R \$USER /home/\$USER/.kube /home/\$USER/.minikube

#### 2-) Installing Istio

    $ curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.4.0 sh -
    $ cd istio-1.4.0
    $ for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
    $ kubectl apply -f install/kubernetes/istio-demo.yaml
    $ watch -n1 kubectl get pods -n istio-system #Wait pods to be Running
    $ kubectl label namespace default istio-injection=enabled #Enable sidecar injection

#### 3-) Deploying the Service and two Deployments
- After ensuring that you've the Java and Go Docker images, extract the Kubernetes manifests from .zip, deploy service.yml and deployment_go.yml, deployment_java.yml by 

       $ kubectl apply -f service.yml
       $ kubectl apply -f deployment_go.yml
       $ kubectl apply -f deployment_java.yml
- (Optional) Added two HPA files for both deployments. If you'd like to provision pod autoscaling, you can execute
 
       $ kubectl apply -f hpa_go.yml
       $ kubectl apply -f hpa_java.yml

 - To test whether the deployments are working or not, port-forward the deployments by

       $ kubectl port-forward deployment/wr-go 8080:8080 (curl http://localhost:8080)
       $ kubectl port-forward deployment/wr-java 8081:8080 (curl http://localhost:8081)
       $ # OR
       $ curl $(kubectl get service -n default |grep wr | awk '{print $3}'):8080

- After ensuring that the deployments are working as expected, create an Ingress Gateway, Virtual Service and Destination Rule by

      $ kubectl apply -f ingress_gateway.yml 
      $ kubectl apply -f virtual_service.yml 
      $ kubectl apply -f destination_rule.yml

- Modify /etc/hosts file to map Ingress Gateway IP with wr.com by

      $ echo "$(kubectl get service -n istio-system | grep ingress | awk '{print $3}') wr.com" | sudo tee -a /etc/hosts

- To test the connection for Ingress Gateway and make requests for Kiali monitoring later, execute

      - for ((i=1;i<=100;i++)); do curl -s -o /dev/null -v "wr.com"; sleep 0.2; done

- After making 100 requests, we can now see that the Go subset gets approximately two out of three requests and Java subset gets one of three.

      - xdg-open "https://$(minikube ip):$(kubectl -n istio-system get service kiali -o jsonpath='{.spec.ports[?(@.port==20001)].nodePort}')/kiali/console"
