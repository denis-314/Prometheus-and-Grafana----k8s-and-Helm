Follow the below steps to deploy Prometheus and Grafana Apps in Kubernetes using Helm charts.

PREREQUISITES:

    l. Kubernetes Cluster
    2. Helm chart
    3. Open communication on ports 30007, 30008 and 30009 on the Router (or other ports, but they need to be changed in the serice).

DEPLOYMENT STEPS:

--METALLB--
1. Install Metallb as Load Balancer: https://www.youtube.com/watch?v=k8bxtsWe9qw
   
   1.1 Use Netplan to add additional IP addresses to the Network Adapter of the Master kubernetes node
   
 - Change directory to netplan location
 - create a config file 01_config.yaml
 - fill in the configuration to add additional IP addressess on the specified network adapder
 - apply the netplan
 - check if the IP addresses have been added

       cd /etc/netplan
       vi 01_config.yaml

       ---
       network:
         version: 2
         renderer: networkd
         ethernets:
           ens160:
             addresses:
             - 10.40.0.132/24
             - 10.40.0.135/24
             - 10.40.0.136/24
             gateway4: 10.40.0.254
             nameservers:
               addresses: [10.40.0.8, 1.1.1.1]
       ---

       sudo netplan apply
       ip addr show ens160

   
   1.2 Edit the kube-proxy Config Map in the kube-system namespace in order to enable strict ARP.

 - Change data.config.conf.ipvs.strictARP from „false” to „true” 

       kubectl edit configmap -n kube-system kube-proxy


   1.3 Install Metallb

 - check if the deployment was successfull

       kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
       k get all -n metallb-system


   1.4 Create an IPAddressPool object:

 - Create a directory to store the metallb manifest files
 - Create the IPAddressPool yaml file
 - specify the range of external IPs which was previously defined on the Network interface
 - apply the manifest file
 - check if the IPAddressPool was created

       mkdir metallb-files
       cd metallb-files
       vi pool-1.yaml

       ---
       apiVersion: metallb.io/v1beta1
	   kind: IPAddressPool
	   metadata:
	     name: first-pool
	     namespace: metallb-system
       spec:
	     addresses:
	     - 10.40.0.135-10.40.0.136
       ---

       kubectl apply -f pool-1.yaml -n metallb-system
       kubectl get IPAddressPool -n metallb-system


   1.5 Create an L2Advertisement object:
   
 - Create the L2Advertisement yaml file
 - Specify the IP Address Pool advertised
 - apply the manifest file
 - check if the L2Advertisement was created

       vi l2-advertisement.yaml

       ---
       apiVersion: metallb.io/v1beta1
	   kind: L2Advertisement
	   metadata:
	     name: demo
	     namespace: metallb-system
	   spec:
	     ipAddressPools:
	     - first-pool
       ---

       kubectl apply -f l2-advertisement.yaml
       kubectl get l2advertisement -n metallb-system


--INGRESS SERVICE--

2. Install NGINX as Ingress Controller:

   2.1 Install Nginx
 - apply the manifest file
 - check if the deployment was successfull and if the Ingress Controller Service has picked up an external IP

       kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
       kubectl get all -n ingress-nginx


--PROMETHEUS & GRAFANA--

3. Install Prometheus & Grafana

   3.1 Add the helm official repository for Prometheus and Grafana 

       helm repo add stable https://charts.helm.sh/stable
       helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    
   3.2 Create namespace
     
       kubectl create namespace prometheus

   3.3 Download the dafault values.yaml file from https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
     The default file can be seen also with the below command, but it cannot be copied if there are too many lines. I have attached it here.

       helm show values prometheus-community/kube-prometheus-stack

   3.4 Edit the content of the default values.yaml and add the Ingress path and domain, based on section "How to serve Grafana with a path prefix (/grafana)" from the official documentation https://github.com/grafana/helm-charts/tree/main/charts/grafana. The complete modified values.yaml file is attached here (lines 979 - 992).
     Obs ! Can be modified for testing reason, because the following warnings were received:

    W0507 15:00:23.905092 2862610 warnings.go:70] annotation "kubernetes.io/ingress.class" is deprecated, please use 'spec.ingressClassName' instead
    W0507 15:00:23.905160 2862610 warnings.go:70] path /grafana/?(.*) cannot be used with pathType Prefix

   3.5 Copy the values.yaml file locally.
     
   3.6 Install Prometheus and Grafana using teh Helm charts:
     
   Option 1: With the Grafana Service exposed over NodePort:
     
       helm install stable prometheus-community/kube-prometheus-stack -n prometheus
  
   Option 2: With the Grafana Service exposed over LoadBalancer & Ingress. Run the command from the same directory where values.yaml is located

       helm install stable prometheus-community/kube-prometheus-stack -n prometheus --values=values.yaml

   3.7 (! Optional - for access via NodePort. Skip for access via LoadBalancer & Ingress !):
     Edit the services to change the default service type to NodePort, and change the default ports, according to the yaml files from this repo:
     - stable-kube-prometheus-sta-prometheus.yml
     - stable-grafana.yml

    kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
    kubectl edit svc stable-grafana -n prometheus

   3.8 (! Optional - for access via NodePort. Skip for access via LoadBalancer & Ingress !):
     Access Prometheus on port 30007 and Grafana on port 30009 in browser. The default credentials for Grafana are:
     - user: admin
     - pass: prom-operator

---
Additional info:

* Obs! In case the default credentials are not working, they can be found in the secret „stable-grafana” using the below command:
       
      kubectl get secret stable-grafana -oyaml -n prometheus
    
      echo "YWRtaW4=" | base64 --decode
      echo "cHJvbS1vcGVyYXRvcg==" | base64 --decode
 
* It can be checked if the ingress information was passed to the Grafana pod by executing inside the container and checking the Grafana initial configuration file:

      k exec -it stable-grafana-54d9f9d99f-skslj -n prometheus -- /bin/bash
      cd /etc/grafana
      vi grafana.ini

---
Bibliografy: 
- https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- https://www.youtube.com/watch?v=k8bxtsWe9qw
