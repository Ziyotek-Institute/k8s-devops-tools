# devops-tools 

### In this Jenkins Kubernetes deployment we have used the following:

* securityContext' for Jenkins pod to be able to write to the local persistent volume.
* Liveness and readiness probe to monitor the health of the Jenkins pod.
* Local persistent volume based on local storage class that holds the Jenkins data path '/var/jenkins_home'.

**NOTE**: The deployment file uses local storage class persistent volume for Jenkins data. For production use cases, you should add a cloud-specific storage class persistent volume for your Jenkins data

With the pod name, you can get the logs as shown below. Replace the pod name with your pod name.

kubectl

```
kubectl logs jenkins-deployment-2539456353-j00w5 --namespace=devops-tools
```

Alternatively, you can run the exec command to get the password directly from the location as shown below.

```
kubectl exec -it jenkins-559d8cd85c-cfcgk cat /var/jenkins_home/secrets/initialAdminPassword -n devops-tools
```