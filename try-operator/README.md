# 201-3 Deploy a Kubernetes Oprator

Kubernetes Operator is getting really important to understand these days. An operator has the following two components.

- Custom Resource Definition
- Custom Controller

As we learnt before, Kubernetes uses a controller to manage each resource. e.g. Deployment resource is created/update/destroyed by Deployment controller

Custom Resource is one of the Kubernetes extension feature and it enables you to define your own resource definiton.

It is very useful when you want to do something like 

- Simplify middleware deployments that are used in many teams and deployed into your cluster but in the same purpose
- Simplify your manual operations

In practice, Operators are developed by many third-party vendors such as MinIO, MySQL, Cassandra and more.

[Operator Hub](https://operatorhub.io/) is the central place where you can find many of these operators.

This time, we will try to use mattermost operator to make it easy to install mattermost application; the Slack open source clone.

## Helm vs Operator

These two can sound similar when you compare them with "simplifying installation" perspective, but they have big difference for the use cases

- Helm is better when you want to install app just one time.
- Operator is better when you want to performance the process(not necessarily installation/initializing) multiple times.

In short, operators will work for you as your automated operator :)

## 1. Login to the bastion server

Using your terminal emulator such as WSL, putty or ConEmu, login to the server.

## 2. Let's try mattermost operator!

[mattermost](https://mattermost.com/) is a Slack-like open source chat service and they provide a Kubernetes operator to easily build your own mattermost envirnonment!

### 2-1. Check the prerequisites

In order to run mattermost in Kubernetes, you're required to have more than 6 nodes that have >4vCPU and >8GB RAM.

The mattermost operator depends on two other operators. They provision DB and Object Storage for mattermost functionality. Ingress is used to provide the external endpoint to the mattermostservice.

- MySQL operator
- MinIO operator
- Nginx Ingress

### 2-2. Install dependencies

1. Install MySQL and MinIO operator.

```shell
$ kubectl apply -f mysql-operator-namespace.yaml
$ kubectl apply -n mysql-operator -f https://raw.githubusercontent.com/mattermost/mattermost-operator/master/docs/mysql-operator/mysql-operator.yaml
$ kubectl apply -f minio-operator-namespace.yaml
$ kubectl apply -n minio-operator -f https://raw.githubusercontent.com/mattermost/mattermost-operator/master/docs/minio-operator/minio-operator.yaml
```

2. Install nginx ingress.

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/deploy.yaml
```

### 2-3 Now, install the mattermost operator

```shell
$ kubectl apply -f mattermost-operator-namespace.yaml
$ kubectl apply -n mattermost-operator -f https://raw.githubusercontent.com/mattermost/mattermost-operator/master/docs/mattermost-operator/mattermost-operator.yaml
```

Having a look at the operator yaml, you can see they define a CRD named ClusterInstallation.

This resource will control whatever you'd need to create mattermost components automatically!

### 3. Deploy a mattermost cluster

Execute the following the command and copy the `EXTERNAL-IP` of ingress-nginx-controller.

```shell
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                          PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.208.196   ab47036fd4cb74ef9b8884482ec2ab2d-904289beb63d5ab3.elb.ap-northeast-1.amazonaws.com   80:30528/TCP,443:30392/TCP   17m
```

Create a YAML file and apply it to your Kubernetes. Make sure you overrode the placeholder in `ingressName` with the above information.


```yaml
apiVersion: mattermost.com/v1alpha1
kind: ClusterInstallation
metadata:
  name: mm-example-full
spec:
  size: 5000users
  ingressName: <YOUR-LOADBALANCER-FQDN> #Override with the EXTERNAL-IP
  ingressAnnotations:
    kubernetes.io/ingress.class: nginx
  version: 5.14.0
  mattermostLicenseSecret: ""
  database:
    storageSize: 50Gi
  minio:
    storageSize: 50Gi
  elasticSearch:
    host: ""
    username: ""
    password: ""
```

This will create a bunch of Kubernetes resources including MySQL statefulset, MinIO statefulset and mattermost deployment.

You can see the progress of deployment by this operetor here;
```
$ watch -c kubectl get all
```

Note that StatefulSet resource does not handle deletion of Persistent Volumes as they try to keep the persistent storage.

### 4. Access to your mattermost

1. Get the ingress information

```shell
$ kubectl get ingress
NAME              HOSTS                            ADDRESS                                                                              PORTS   AGE
mm-example-full   ab47036fd4cb74ef9b8884482ec2ab2d-904289beb63d5ab3.elb.ap-northeast-1.amazonaws.com    a49d6f15b2d1f4fda867e632dc3e948a-6fb6b18b7c643e20.elb.ap-northeast-1.amazonaws.com   80      7h55m
```

2. Use your browser and access to the LoadBalancer FQDN :)

You should be able to see your mattermost chat now!
