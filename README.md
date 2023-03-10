# Building Dapr-enabled Platforms using Kratix

In this tutorial we will use Kratix to configure our Dapr enabled development environment. 
The main objective is to request new development environments for our applications that comes with all the components needed already pre-installed and configured to work. 

## Installing Kratix 

We will be using a Kubernetes KinD Cluster to run Kratix and configure our development environments. You can create a KinD Cluster by running: 

```
kind create cluster --name platform --image kindest/node:v1.24.7
```

Then we install Kratix into the cluster by following their getting started guide: https://kratix.io/docs/main/quick-start

```
kubectl apply --filename https://raw.githubusercontent.com/syntasso/kratix/main/distribution/single-cluster/install-all-in-one.yaml
```

And then registering the same cluster as a worker cluster. If you are working in multiple clusters, this  

```
kubectl apply --filename https://raw.githubusercontent.com/syntasso/kratix/main/distribution/single-cluster/config-all-in-one.yaml
```
# Defining Dapr-enabled Development environments

We will create a new Kratix promise to represent the components that needs to be installed for each Development Environment that we want to create. 
For this example, we want a developer environment to have Dapr installed, Redis installed and a Dapr Statestore component ready to be used by our developers. 

For that we create a new Kratix promise that installs the Dapr and Redis promise and defines our Dapr Statestore Component configuration to point to the installed Redis instance.

```env-promise/promise.yaml
```

Then we apply this promise to the cluster.

```bash
kubectl apply -f env-promise/promise.yaml
```

This will install Dapr and Redis Promises, which themselves installed Dapr and
the Redis operator.


# Requesting new Dapr-enabled Development Environment

Now that we have our promise installed in Kratix we can create new development environments by sending the following resource:

```env-promise/resource-request.yaml
```

Then we then make this request to the Platform cluster:

```bash
kubectl apply -f env-promise/resource-request.yaml
```

Now if we inspect the cluster we will see we have a Redis instance running, and
the components configurated in Dapr:
```
kubectl get pods
```

```
kubectl get components
```

Lets test this out by deploying our example node app:
```
kubectl apply -f node-app.yaml
```

Port forward to it:
```
kubectl port-forward service/nodeapp 8080:80
```

Lets put some state into our app:
```
curl --request POST --data "@sample.json" --header Content-Type:application/json http://localhost:8080/neworder
```

And lets see that the app successfully stores the state for order using our Dapr
Statestore:
```
curl http://localhost:8080/order
```


