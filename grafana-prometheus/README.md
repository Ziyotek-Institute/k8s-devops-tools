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

* [eks_ctl](https://eksctl.io/) follow for more details [AWS_EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)

```bash
eksctl create cluster -f cluster.yaml
eksctl delete cluster -f cluster.yaml
```

* [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

**IMPORTANT:** Provision ingress-controller in your Kubernetes cluster.

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

```bash
EBS_ROLE_ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query Role.Arn --output text)
```

To deploy the CSI driver:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.13"
```

```bash
kubectl patch serviceaccount "ebs-csi-controller-sa" --namespace kube-system --patch \
 "{\"metadata\": { \"annotations\": { \"eks.amazonaws.com/role-arn\": \"$EBS_CSI_ROLE_ARN\" }}}"
kubectl patch serviceaccount "ebs-csi-node-sa" --namespace kube-system --patch \
 "{\"metadata\": { \"annotations\": { \"eks.amazonaws.com/role-arn\": \"$EBS_CSI_ROLE_ARN\" }}}"
```

**1.** Install Prometheus and Install Grafana

* We’ll start by adding the repository to our helm configuration:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```

* Once the repo is ready, we can install the provided charts by running the following commands:

```bash
kubectl create namespace prometheus
```

```bash
helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set alertmanager.persistentVolume.size="8Gi" \
    --set server.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.size="8Gi" \
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
    --set persistence.size="8Gi" \
    --values ./grafana.yaml \
    --set service.type=ClusterIP
```

**IMPORTANT** Validate the Status of persistentvolumeclaim, should be ```BOUND```

```bash
$ kubectl get sc,pvc -A 
NAME                                        PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  12h

NAMESPACE    NAME                                                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus   persistentvolumeclaim/prometheus-server                   Bound    pvc-b8e00e8d-20c9-41ad-9dd3-a95448a9df90   8Gi        RWO            gp2            6m53s
prometheus   persistentvolumeclaim/storage-prometheus-alertmanager-0   Bound    pvc-05519ffb-c2c7-4d32-b6ad-eba02d92ddd4   2Gi        RWO            gp2            6m53s

```

**2.** We can add an ```kind: Ingress``` object and configure a [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) value for the host key. The ingress controller will match incoming HTTP traffic, and route it to the appropriate backend service based on the host key.
First expose the services via type ClusterIP.

**NOTE!** Grafana and Prometheus Services where created via helm during creation of grafana . Similar resource manifest files can be viewed in this repo, apply in case services have not been created [service-prometheus.yaml](service-prometheus.yaml) [service-grafana.yaml](service-grafana.yaml)

* Create an ingress resource manifest files [ingress-prometheus.yaml](ingress-prometheus.yaml) [ingress-grafana.yaml](ingress-grafana.yaml)

```bash
kubectl apply -f ingress-prometheus.yaml
kubectl apply -f ingress-grafana.yaml
```

###Get external-dns pod logs ```kubectl logs deployment/external-dns```

```bash
time="2022-12-03T18:00:26Z" level=info msg="Desired change: CREATE prometheus.kubeshore.com A [Id: /hostedzone/Z0018530333ZM5MP7G8FQ]"
time="2022-12-03T18:00:26Z" level=info msg="Desired change: CREATE prometheus.kubeshore.com TXT [Id: /hostedzone/Z0018530333ZM5MP7G8FQ]"
time="2022-12-03T18:00:26Z" level=info msg="2 record(s) in zone kubeshore.com. [Id: /hostedzone/Z0018530333ZM5MP7G8FQ] were successfully updated"
...
...
...
time="2022-12-03T18:09:30Z" level=info msg="Desired change: CREATE grafana.kubeshore.com A [Id: /hostedzone/Z0018530333ZM5MP7G8FQ]"
time="2022-12-03T18:09:30Z" level=info msg="Desired change: CREATE grafana.kubeshore.com TXT [Id: /hostedzone/Z0018530333ZM5MP7G8FQ]"time="2022-12-03T18:09:30Z" level=info msg="2 record(s) in zone kubeshore.com. [Id: /hostedzone/Z0018530333ZM5MP7G8FQ] were successfully updated"
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
