# Prometheus and Grafana setup #

## Introduction ##

In this LAB you will deploy [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/) into your [Minikube](https://github.com/kubernetes/minikube) or Any other Kubernetes cluster using provided Helm charts.

* Prometheus will scrape all data about the state of Kubernetes Cluster and other resources running on it.

* Grafana will help you visualize metrics recorded by Prometheus and display them in fancy [dashboards](https://grafana.com/grafana/dashboards/).

## Prerequisites ##

 **A.** [Helm](https://helm.sh/) - to create prometheus and grafana using helm kubernetes package manager

* Install helm on centOS 7

```bash
mkdir -p ~/pre-reqs/; cd ~/pre-reqs/

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

helm --help
```

* Install helm on macOS

```bash
brew install helm
```

* Install helm on Windows

```bash
choco install kubernetes-helm
```

 **B.** Choose your Kubernetes Cluster

* [Minikube](https://github.com/kubernetes/minikube)

* [eks_ctl](https://eksctl.io/) follow for more details [AWS_EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)

* [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

**IMPORTANT:** Provision ingress-controller in your Kubernetes cluster.

* For Minikube create ingress-controller using available minikube addons

```bash
minikube addons enable ingress
```

* For any other kubernetes clusters Create nginx-ingress-controller from Web URL

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/aws/deploy.yaml
```

```bash
export EKS_CLUSTER_NAME=""
export EKS_CLUSTER_REGION=""
```

## Enable aws-ebs-csi addon for managing dynamic provisioning of aws ebs volume ##

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $EKS_CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

**1.** Install Prometheus and Install Grafana

* We’ll start by adding the repository to our helm configuration:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```

* Once the repo is ready, we can install the provided charts by running the following commands:

```bash
helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2" \
    --set service.type=ClusterIP
```

* Create namespace for grafana

```bash
kubectl create namespace grafana
```

**checkout** [grafana.yaml](grafana.yaml) consists of configuration settings of prometheus as a data_source for grafana.

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
```

```bash
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='EKS!sAWSome' \
    --values ./grafana.yaml \
    --set service.type=ClusterIP
```

**2.** We can add an ```kind: Ingress``` object and configure a [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) value for the host key. The ingress controller will match incoming HTTP traffic, and route it to the appropriate backend service based on the host key.
First expose the services via type ClusterIP.

**NOTE!** Grafana and Prometheus Services where created via helm during creation of grafana . Similar resource manifest files can be viewed in this repo, apply in case services have not been created [service-prometheus.yaml](service-prometheus.yaml) [service-grafana.yaml](service-grafana.yaml)

* Create an ingress resource manifest files [ingress-prometheus.yaml](ingress-prometheus.yaml) [ingress-grafana.yaml](ingress-grafana.yaml)

```bash
kubectl apply -f ingress-prometheus.yaml
kubectl apply -f ingress-grafana.yaml
```

**3.** Access Grafana Dashboard

* Get your 'admin' user password by running:

```bash
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

The Grafana server can be accessed via [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) value for the host key in the [ingress-grafana.yaml](ingress-grafana.yaml)

**4.** Configure Prometheus Datasource

* We need to navigate in Grafana dashboard to Configuration > Datasources and add a new Prometheus instance.
* The URL for our Prometheus instance is the name of the service `http://prometheus:80`

**5.** Kubernetes Dashboard bootstrap

* We head to Create (+) > Import section to Import via grafana.com and we set 6417 and 1860 into the id field and click Load.
* In the dashboard configuration we need to select the Prometheus Datasource we created in the earlier step.

**6.** Clean Up

```bash
# if you provisioned ingress-nginx-controller via Web URL

kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/aws/deploy.yaml

# clean up individual component

helm uninstall prometheus prometheus-community/prometheus  --namepsace prometheus
helm uninstall grafana grafana/grafana --namespace grafana

# If you provisioned minikube cluster
minikube delete

# If you provisioned EKS cluster
eksctl delete cluster
```
