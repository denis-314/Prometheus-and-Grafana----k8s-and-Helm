Follow the below steps to deploy Prometheus and Grafana Apps in Kubernetes using Helm charts.

PREREQUISITES:

    l. Kubernetes Cluster
    2. Helm chart
    3. Open communication on ports 30007, 30008 and 30009 on the Router (or other ports, but they need to be changed in the serice).

DEPLOYMENT STEPS:

  l. Add the helm official repository for Prometheus and Grafana 

    helm repo add stable https://charts.helm.sh/stable
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    
  2. Create namespace
     
    kubectl create namespace prometheus

  3. Download the dafault values.yaml file from https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
     The default file can be seen also with the below command, but it cannot be copied if there are too many lines

    helm show values prometheus-community/kube-prometheus-stack

  4. Edit the content of the default values.yaml and add the Ingress path and domain, based on section "How to serve Grafana with a path prefix (/grafana)" from the official documentation https://github.com/grafana/helm-charts/tree/main/charts/grafana. The complete modified values.yaml file is attached here (lines 979 - 992).
     Obs ! Can be modified for testing reason, because the following warnings were received:

    W0507 15:00:23.905092 2862610 warnings.go:70] annotation "kubernetes.io/ingress.class" is deprecated, please use 'spec.ingressClassName' instead
    W0507 15:00:23.905160 2862610 warnings.go:70] path /grafana/?(.*) cannot be used with pathType Prefix

  5. Copy the values.yaml file locally.
     
  6. Install Prometheus and Grafana using teh Helm charts:
     
 With the Grafana Service exposed over NodePort:
     
      helm install stable prometheus-community/kube-prometheus-stack -n prometheus
  
 With the Grafana Service exposed over LoadBalancer & Ingress. Run the command from the same directory where values.yaml is located

    helm install stable prometheus-community/kube-prometheus-stack -n prometheus --values=values.yaml

  7. (! Optional - for access via NodePort. Skip for access via LoadBalancer & Ingress !) Edit the services to change the default service type to NodePort, and change the default ports, according to the yaml files from this repo:
     - stable-kube-prometheus-sta-prometheus.yml
     - stable-grafana.yml

    kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
    kubectl edit svc stable-grafana -n prometheus

  8. (! Optional - for access via NodePort. Skip for access via LoadBalancer & Ingress !) Access Prometheus on port 30007 and Grafana on port 30009 in browser. The default credentials for Grafana are:
     - user: admin
     - pass: prom-operator
    
       Obs! In case the default credentials are not working, they can be found in the secret „stable-grafana” using the below command:
       
    kubectl get secret stable-grafana -oyaml -n prometheus
    
    echo "YWRtaW4=" | base64 --decode
    echo "cHJvbS1vcGVyYXRvcg==" | base64 --decode
 


---
Bibliografy: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
