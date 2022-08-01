# Kratix Multi-Cluster Install
_To install on a single cluster use the [Single Cluster Quick Start Guide](https://github.com/syntasso/kratix/blob/main/docs/single-cluster.md)._

### Install KinD

See [the KinD quick start guide](https://kind.sigs.k8s.io/docs/user/quick-start/) to install KinD.

### Clone Kratix
```
git clone https://github.com/syntasso/kratix.git
```

### Set up Platform Cluster

The below commands will create our platform cluster and install Kratix.

```
kind create cluster --name platform
kubectl apply -f distribution/kratix.yaml
kubectl apply -f hack/platform/minio-install.yaml
```

The Kratix API should now be available.

```
kubectl get crds
```

The above command will give an output similar to:
```
NAME                                     CREATED AT
clusters.platform.kratix.io   2022-05-10T11:10:57Z
promises.platform.kratix.io   2022-05-10T11:10:57Z
works.platform.kratix.io      2022-05-10T11:10:57Z
```

### Multi-Cluster Networking
Some KinD installations use non-standard networking. To ensure cross-cluster communication we need to run this script: 

```
PLATFORM_CLUSTER_IP=`docker inspect platform-control-plane | grep '"IPAddress": "172' | awk '{print $2}' | awk -F '"' '{print $2}'` 
sed -i'' -e "s/172.18.0.2/$PLATFORM_CLUSTER_IP/g" hack/worker/gitops-tk-resources.yaml
```

### Set up Worker Cluster
This will create a cluster for running the X-as-a-service workloads:

```
kind create cluster --name worker #Also switches kubectl context to worker
kubectl apply -f config/samples/platform_v1alpha1_worker_cluster.yaml --context kind-platform #register the worker cluster with the platform cluster
kubectl apply -f hack/worker/gitops-tk-install.yaml
kubectl apply -f hack/worker/gitops-tk-resources.yaml
```

Once Flux is installed and running (this may take a few minutes), the Kratix resources will be visible on the worker cluster.

```
kubectl get ns kratix-worker-system
```

The above command will give an output similar to:
```
NAME                   STATUS   AGE
kratix-worker-system   Active   4m2s
```

Congratulations! Kratix is now installed.

## Part 2: Install a Jenkins Promise 

For the purpose of this walkthrough let's install the provided Jenkins-as-a-service Promise.

```
kubectl config use-context kind-platform
kubectl apply -f samples/jenkins/jenkins-promise.yaml
```

On the platform cluster we can now see the ability to create Jenkins instances.

```
kubectl get crds jenkins.example.promise.syntasso.io
```

The above command will give an output similar to:
```
NAME                                     CREATED AT
jenkins.example.promise.syntasso.io   2021-09-03T12:02:20Z
```

On our worker cluster we should see that the Jenkins operator has been installed. 

```
kubectl get pods --namespace default --context kind-worker
```

The above command will give an output similar to:
```
NAME                                READY   STATUS    RESTARTS   AGE
jenkins-operator-7886c47f9c-zschr   1/1     Running   0          4m1s
```

Congratulations! You have now installed your first Promise. The machinery to issue Jenkins instances on demand by application teams has now been installed.

## Part 3: Request a Jenkins Instance

```
kubectl apply -f samples/jenkins/jenkins-resource-request.yaml
```

We can see the request on the platform cluster.

```
kubectl get jenkins.example.promise.syntasso.io
```

The above command will give an output similar to:
```
NAME                   AGE
my-jenkins   27s
```

### Review created Jenkins instance on the worker cluster

Once Kratix has applied the new configuration to the worker cluster (this will take a few minutes), the Jenkins instance will be created.

```
kubectl get pods --namespace default --context kind-worker
```

The above command will give an output similar to:
```
NAME                                READY   STATUS    RESTARTS   AGE
jenkins-example                     1/1     Running   0          113s
jenkins-operator-7886c47f9c-zschr   1/1     Running   0          19m
```

Congratulations! You have now created an instance of Jenkins.

### Using your Jenkins instance

We can see the Jenkins UI in our browsers (all commands on worker cluster):
1. Get the Jenkins username: `kubectl --context kind-worker get secret jenkins-operator-credentials-example -o 'jsonpath={.data.user}' | base64 -d`
2. Get the Jenkins password: `kubectl --context kind-worker get secret jenkins-operator-credentials-example -o 'jsonpath={.data.password}' | base64 -d`
3. `kubectl --context kind-worker port-forward jenkins-example 8080:8080` 
4. Navigate to http://localhost:8080 and login using the username and password captured in steps one and two. 
5. You should see a Seed Job in the Jenkins UI, and a corresponding Pod on your Worker cluster. 

## What have we learned?

1. We created an internal platform API, and a worker cluster to host workloads. 
2. We then decorated our platform API by Promising Jenkins-as-a-service.
3. We adopted the role of an application team member and requested a Jenkins instance from the platform. The Jenkins instance was created on the worker cluster.

## Feedback

Please help to improve Kratix. Give feedback [via email](mailto:feedback@syntasso.io?subject=Kratix%20Feedback) or [google form](https://forms.gle/WVXwVRJsqVFkHfJ79). Alternatively, open an issue or pull request.

## Challenge 
[Write your own Promise](./writing-a-promise.md), with a custom pipeline image, and share it with the world!