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
apiVersion: platform.kratix.io/v1alpha1
kind: Promise
metadata:
  name: env
  namespace: default
spec:
  workerClusterResources:
  - apiVersion: platform.kratix.io/v1alpha1
    kind: Promise
    metadata:
      creationTimestamp: null
      name: dapr
      namespace: default
    ...
  - apiVersion: platform.kratix.io/v1alpha1
    kind: Promise
    metadata:
      creationTimestamp: null
      name: redis
      namespace: default
    ...
  xaasCrd:
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: env.marketplace.kratix.io
    spec:
      group: marketplace.kratix.io
      names:
        kind: env
        plural: env
        singular: env
      scope: Namespaced
      versions:
      - name: v1alpha1
        schema:
          openAPIV3Schema:
            properties:
              spec:
                properties:
                  deployApp:
                    description: |
                      Deploy the Dapr Applications defined
                    type: boolean
                  database:
                    properties: 
                      statestoreName:
                        description: |
                          The name of the statestore component to create
                        type: string
                      enabled:
                        description: |
                          Deploy an instance of Redis or not
                        type: boolean
                    type: object
                type: object
            type: object
        served: true
        storage: true
  xaasRequestPipeline:
  - salaboy/environment-request-pipeline:v0.1.0

```

Then we apply this promise to the cluster.

```bash
kubectl apply -f env-promise/promise.yaml
```

This will install Dapr and Redis Promises, which themselves installed Dapr and
the Redis operator.


# Requesting new Dapr-enabled Development Environment

Now that we have our promise installed in Kratix we can create new development environments by sending the following resource:

```env-promise/my-dev-env.yaml
apiVersion: marketplace.kratix.io/v1alpha1
kind: env
metadata:
  name: my-dev-env
  namespace: default
spec:
  deployApp: true
  database:   
    enabled: true
    statestoreName: "statestore"

```

Then we then make a new Development Environment request to the Platform cluster:

```bash
kubectl apply -f env-promise/my-dev-env.yaml
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


# Cleaning up

You can delete your KinD Cluster to clean up all the installed components: 
```
kind delete clusters platform
```