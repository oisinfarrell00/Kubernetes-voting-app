# Kubernetes-voting-app Deployment Version
The voting application 

## Architecture
This system is similar to the pod version but uses deployments. This allows you to set define a number of desired pods and Kubernetes will ensure that this number of pods is maintained. This system contains 5 ``deployments`` and 4 ``services``

Deployments: vote, result, redis, db, worker

Services: vote-service, result-servie, redis-service, db-service


## Create Pods:
Command:
```
kubectl create -f .
```
Note: In production or ongoing development, you would typically use ``kubectl apply -f .`` instead of create, as apply updates resources declaratively.


Output:
```
deployment.apps/db-deployment created
service/db created
deployment.apps/redis-deployment created
service/redis created
deployment.apps/result-deployment created
service/result created
deployment.apps/vote-deployment created
service/vote created
deployment.apps/worker-deployment created
```

## Replica Sets
`Deployments` use replica sets under the hood to ensure that N pods are running at all times. Thus in this example the deployments have automatically creates replica sets, one for each deployment.

```
kubectl get rs
```

```
NAME                           DESIRED   CURRENT   READY   AGE
db-deployment-6dd96f5df5       1         1         1       2m28s
redis-deployment-664fbf775b    1         1         1       2m28s
result-deployment-7b7f69957f   3         3         3       2m28s
vote-deployment-6d8f9ddcdd     3         3         3       2m28s
worker-deployment-89f44c667    1         1         1       2m28s
```
3 pods are defined in the deployment defininition for both the vote and result elements. The rest have one. As you can see above 3 pods are currently up and running for these. More details about one of the replica sets in more detail using the following command: ``kubectl describe rs/vote-deployment-6d8f9ddcdd``

### Check pods running un the system:
```
kubectl get pods
```
Output:
```
NAME                                 READY   STATUS    RESTARTS   AGE
db-deployment-6dd96f5df5-mkcrn       1/1     Running   0          25m
redis-deployment-664fbf775b-l7drw    1/1     Running   0          25m
result-deployment-7b7f69957f-5vxzr   1/1     Running   0          25m
result-deployment-7b7f69957f-dn54r   1/1     Running   0          25m
result-deployment-7b7f69957f-p6vd5   1/1     Running   0          25m
vote-deployment-6d8f9ddcdd-cx6fz     1/1     Running   0          65s
vote-deployment-6d8f9ddcdd-fd5z8     1/1     Running   0          25m
vote-deployment-6d8f9ddcdd-gcr46     1/1     Running   0          25m
worker-deployment-89f44c667-p946w    1/1     Running   0          25m
```
You can see here that there is 3 instances of the vote and result pods as we defined three replicas needed in the deployment. 

#### Pod Crash Simulation
If one of the voter pods were to crash the replic set would automatically provision a new one. To demonstrate this we will add a watch to the vote pods:

```
kubectl get pods -l app=vote -w
```
```
NAME                               READY   STATUS    RESTARTS   AGE
vote-deployment-6d8f9ddcdd-cx6fz   1/1     Running   0          23s
vote-deployment-6d8f9ddcdd-fd5z8   1/1     Running   0          24m
vote-deployment-6d8f9ddcdd-gcr46   1/1     Running   0          24m
```

There are three vote pods running. To simulate one crashing we will delete one. 

```
kubectl delete pod vote-deployment-6d8f9ddcdd-cx6fz
```
Result:
```
NAME                               READY   STATUS               RESTARTS   AGE
vote-deployment-6d8f9ddcdd-cx6fz   1/1     Running              0          23s
vote-deployment-6d8f9ddcdd-cx6fz   1/1     Terminating          0          2m24s
vote-deployment-6d8f9ddcdd-gl95k   0/1     Pending              0          0s
vote-deployment-6d8f9ddcdd-cx6fz   1/1     Terminating          0          2m24s
vote-deployment-6d8f9ddcdd-gl95k   0/1     Pending              0          0s
vote-deployment-6d8f9ddcdd-gl95k   0/1     ContainerCreating    0          0s
vote-deployment-6d8f9ddcdd-cx6fz   0/1     Completed            0          2m24s
vote-deployment-6d8f9ddcdd-cx6fz   0/1     Completed            0          2m25s
vote-deployment-6d8f9ddcdd-cx6fz   0/1     Completed            0          2m25s
vote-deployment-6d8f9ddcdd-gl95k   1/1     Running              0          3s
```

You can see pod ``vote-deployment-6d8f9ddcdd-cx6fz`` goes from `Running` to `Terminating` and finally to `Completed`. Pod ``vote-deployment-6d8f9ddcdd-gl95k`` begins in the `Pending` state. Then enters the `ContainerCreating` and finally the `Running` state. If we check the current pods for the vote deployment: ``kubectl get pods -l app=vote``

```
NAME                               READY   STATUS    RESTARTS   AGE
vote-deployment-6d8f9ddcdd-fd5z8   1/1     Running   0          33m
vote-deployment-6d8f9ddcdd-gcr46   1/1     Running   0          33m
vote-deployment-6d8f9ddcdd-gl95k   1/1     Running   0          6m28s
```

You can see that there are 3 running, one which was created much more recently, as a result of the deletion and automatic provisioning. 

During the deletion you can also see changes in the replica set using `kubectl get rs -l app=vote`:
Before:
```
NAME                         DESIRED   CURRENT   READY   AGE
vote-deployment-6d8f9ddcdd   3         3         3       35m
```
During:
```
NAME                         DESIRED   CURRENT   READY   AGE
vote-deployment-6d8f9ddcdd   3         3         2       36m
```

After:
```
NAME                         DESIRED   CURRENT   READY   AGE
vote-deployment-6d8f9ddcdd   3         3         3       36m
```

NOTE: Similarly, if you create a new pod using the create command an old one will be removed to keep the replica set at 3 pods. 




## Accessing the UI (minikube)
Locally I am only running on `minikube` to keep setup easy. To access UI you need to set up a tunnel so the easiest was to see your UI is the following command:
```
minikube service vote
minikube service result
```
Note: They will need to be in two separate terminals

Then you will see the two tabs with both sections of the UI:
![alt text](../screenshots/image.png)
![alt text](../screenshots/image-1.png)

## Accessing the UI (production)
While I have not completed this step myself as I am only using `minikube` the setup for production for this voter application would be as follows:

In a production environment (e.g., AWS EKS, GCP GKE, or Azure AKS), your services will typically be of type ``LoadBalancer`` rather than ``NodePort``. This allows the cloud provider to automatically provision an external load balancer and assign a public IP that routes traffic into your cluster.

Command:
```
kubectl get services
```
Output:
```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
result       NodePort    10.100.109.235   <none>        8081:31001/TCP   3m54s
db           ClusterIP   10.109.82.42     <none>        5432/TCP         3m54s
vote         NodePort    10.103.33.210    <none>        8080:31000/TCP   3m54s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          47h
redis        ClusterIP   10.101.75.62     <none>        6379/TCP         3m54s
```
We can see here that frontend is of Type `NodePort`, which is necessary to be accessible from outside the cluster. The node is running on port ``31000`` and passes requests to port ``80`` on the ``pod``. 


Once the external IPs are assigned (which may take a few minutes), you can access the applications directly using those IPs:

```
http://34.122.18.95:8080     # Vote UI
http://34.122.18.94:8081     # Result UI
```

If your cluster uses NodePort instead of LoadBalancer, you can still access the app through the node’s public IP:

```
http://<NODE_PUBLIC_IP>:31000   # Vote UI
http://<NODE_PUBLIC_IP>:31001   # Result UI
```