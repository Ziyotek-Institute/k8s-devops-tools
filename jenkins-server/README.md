# Jenkins server for EKS or any Kubernetes Cluster, for example minikube

## Step by step Kubernetes Jenkins Deployment


**Step 1:** Create a Namespace for Jenkins. It is good to categorize all the DevOps tools as a separate namespace from other applications

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-tools
```

```
kubectl apply -f namespace.yaml
```

**Step 2:** Create a 'service-account.yaml' file and copy the following admin service account manifest.

The 'service-account.yaml' creates a 'jenkins-admin' clusterRole, 'jenkins-admin' ServiceAccount and binds the 'clusterRole' to the service account.

The 'jenkins-admin' cluster role has all the permissions to manage the cluster components. You can also restrict access by specifying individual resource actions.


```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: devops-tools
```

```
kubectl apply -f  service-account.yaml
```

**Step 3:** for clusters on cloud provider  Create 'volume.yaml' and copy the following persistent volume manifest. For minikube volume check step 3.1

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage
  claimRef:
    name: jenkins-pv-claim
    namespace: devops-tools
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  namespace: devops-tools
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

**Important Note:** Replace 'worker-node01' with any one of your cluster worker nodes hostname.

You can get the worker node hostname using the kubectl.

```
kubectl get nodes
```

For volume, we are using the 'local' storage class for the purpose of demonstration. Meaning, it creates a 'PersistentVolume' volume in a specific node under the '/mnt' location.

As the 'local' storage class requires the node selector, you need to specify the worker node name correctly for the Jenkins pod to get scheduled in the specific node.

If the pod gets deleted or restarted, the data will get persisted in the node volume. However, if the node gets deleted, you will lose all the data.

Ideally, you should use a persistent volume using the available storage class with the cloud provider, or the one provided by the cluster administrator to persist data on node failures.

Let’s create the volume using kubectl

```
kubectl apply -f volume.yaml
```

**Step 3.1:**  We want to create a persistent volume for our Jenkins controller pod. This will prevent us from losing our whole configuration of the Jenkins controller and our jobs when we reboot our minikube. This official minikube doc explains which directories we can use to mount or data. In a multi-node Kubernetes cluster, you’ll need some solution like NFS to make the mount directory available in the whole cluster. But because we use minikube which is a one-node cluster we don’t have to bother about it.

We choose to use the ```/data``` directory. This directory will contain our Jenkins controller configuration.

1. Copy the code block and create a volume which is called jenkins-pv:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
  namespace: jenkins
spec:
  storageClassName: jenkins-pv
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/jenkins-volume/
```

2. Run the following command to apply the spec:
```
kubectl apply -f jenkins-volume.yaml
```

It’s worth noting that, in the above spec, hostPath uses the /data/jenkins-volume/ of your node to emulate network-attached storage. This approach is only suited for development and testing purposes. For production, you should provide a network resource like a Google Compute Engine persistent disk, or an Amazon Elastic Block Store volume.

Minikube configured for hostPath sets the permissions on /data to the root account only. Once the volume is created you will need to manually change the permissions to allow the jenkins account to write its data.

```
minikube ssh
sudo chown -R 1000:1000 /data/jenkins-volume
```


**Step 4:** Create a Deployment file named 'deployment.yaml' and copy the following deployment manifest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
            fsGroup: 1000
            runAsUser: 1000
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
              claimName: jenkins-pv-claim
```

In this Jenkins Kubernetes deployment we have used the following:

* securityContext' for Jenkins pod to be able to write to the local persistent volume.
* Liveness and readiness probe to monitor the health of the Jenkins pod.
* Local persistent volume based on local storage class that holds the Jenkins data path '/var/jenkins_home'.

**NOTE**: The deployment file uses local storage class persistent volume for Jenkins data. For production use cases, you should add a cloud-specific storage class persistent volume for your Jenkins data

If you don’t want the local storage persistent volume, you can replace the volume definition in the deployment with the host directory as shown below.

```yaml
volumes:
- name: jenkins-data
emptyDir: \{}
```

Create the deployment using kubectl.

```
kubectl apply -f deployment.yaml
```

Check the deployment status.

```
kubectl get deployments -n devops-tools
```

Now, you can get the deployment details using the following command.

```
kubectl describe deployments --namespace=devops-tools
```

## Accessing Jenkins Using Kubernetes Service

We have now created a deployment. However, it is not accessible to the outside world. For accessing the Jenkins deployment from the outside world, we need to create a service and map it to the deployment.

Create 'service.yaml' and copy the following service manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: devops-tools
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '8080'
spec:
  selector:
    app: jenkins-server
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
```

**NOTE** Here, we are using the type as 'NodePort' which will expose Jenkins on all kubernetes node IPs on port 32000. If you have an ingress setup, you can create an ingress rule to access Jenkins. Also, you can expose the Jenkins service as a Loadbalancer if you are running the cluster on AWS, Google, or Azure cloud. 

Create the Jenkins service using kubectl.

```
kubectl apply -f service.yaml
```

Now, when browsing to any one of the Node IPs on port 32000, you will be able to access the Jenkins dashboard.

```
http://<node-ip>:32000
```

For minikube use

```
minikube service jenkins 
```

Jenkins will ask for the initial Admin password when you access the dashboard for the first time.

You can get that from the pod logs either from the Kubernetes dashboard or CLI. You can get the pod details using the following CLI command.
```
kubectl get pods --namespace=devops-tools
```
With the pod name, you can get the logs as shown below. Replace the pod name with your pod name.
```
kubectl logs jenkins-deployment-2539456353-j00w5 --namespace=devops-tools
```
The password can be found at the end of the log.

Alternatively, you can run the exec command to get the password directly from the location as shown below.
```
kubectl exec -it jenkins-559d8cd85c-cfcgk cat /var/jenkins_home/secrets/initialAdminPassword -n devops-tools
```
Once you enter the password, proceed to install the suggested plugin and create an admin user. All of these steps are self-explanatory from the Jenkins dashboard.

```log
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

6c72d05887564dd8a45d5288b644b1ee

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

### Apply all files inside the folder 

```
kubectl apply -f .
```