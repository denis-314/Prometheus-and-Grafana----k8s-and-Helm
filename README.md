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

  3. Install Prometheus and Grafana using teh Helm charts

    helm install stable prometheus-community/kube-prometheus-stack -n prometheus

  4. Edit the services to change the default service type to NodePort, and change the default ports, according to the yaml files from this repo:
     - stable-kube-prometheus-sta-prometheus.yml
     - stable-grafana.yml

    kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
    kubectl edit svc stable-grafana -n prometheus

  5. Access Prometheus on port 30007 and Grafana on port 30009 in browser. The default credentials for Grafana are
     - user: admin
     - pass: prom-operator
    
       Obs! In case the default credentials are not working, they can be found in the secret „stable-grafana” using the below command:
       
    kubectl get secret stable-grafana -oyaml -n prometheus
    echo "YWRtaW4=" | base64 --decode
    echo "cHJvbS1vcGVyYXRvcg==" | base64 --decode
