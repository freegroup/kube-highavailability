## High Availability and multi-AZ guidance on AWS

To build a more production-ready configuration, users should consider:

 - running Kubernetes in a High Availability mode (out of scope)
 - running across multiple Availability Zones
 

## Overview
In this tutorial I cover the scenario below. 
 - One Cluster
 - Multi tenants, each in its own namespace
 - **multiple availability zones**
 - each pod of a Deployment/StatefulSet running in a different zone
 
 
![Screen](/images/overview.png?raw=true "overview")

## Setup
To use multiple AZ we need to setup a cluster with a customized shoot YAML  and add the AWS availability
zones and network segments by hand. But don't worry - there's no magic behind it

### Get access to the gardener API
First of all we need a kubeconfig.yaml to access the gardener via `kubectl`. I assume you already know how to use 
kubectl and make a kubeconfig available. Below is a screencast how you can create and download a new kubeconfig.yaml 
from the gardener dashboard.
 
![Screen](/images/create_tech_user.gif?raw=true "create_user")


### Create a shoot.yaml
You can create a new cluster either from the UI or from the command line. For our purposes we have to create the 
cluster via the command line. The reason for this is that the UI does not offer all possibilities to edit the 
zones and the networks.

Below is a template of a shoot.yaml. Please assign the named placeholders with the corresponding values 
and deploy the YAML with `kubectl`. (Keep in mind to use the `kubeconfig.yaml`of your technical user)


Where:
 - **&lt;GARDENER-NEW-CLUSTERNAME-TO-CREATE&gt;** is the name of the new cluster to create
 - **&lt;GARDENER-EXISTING-PROJECT&gt;** is the gardener project which belongs to the technical user
 - **&lt;YOUR-AWS-SECRET-NAME&gt;** the name of the AWS secret to use

```YAML
kind: Shoot
apiVersion: garden.sapcloud.io/v1beta1
metadata:
  name: <GARDENER-NEW-CLUSTERNAME-TO-CREATE>
  namespace: garden-<GARDENER-EXISTING-PROJECT>
  annotations:
    garden.sapcloud.io/createdBy: user@domain.de
    garden.sapcloud.io/purpose: evaluation
  finalizers:
    - gardener
spec:
  addons:
    cluster-autoscaler:
      enabled: true
    heapster:
      enabled: true
    kubernetes-dashboard:
      enabled: true
    nginx-ingress:
      enabled: true
    monocular:
      enabled: false
  backup:
    schedule: '*/5 * * * *'
    maximum: 7
  cloud:
    profile: aws
    region: eu-central-1
    secretBindingRef:
      name: <YOUR-AWS-SECRET-NAME>
    seed: aws-eu1
    aws:
      machineImage:
        name: CoreOS
        ami: ami-d0dcef3b
      networks:
        nodes: 10.250.0.0/16
        pods: 100.96.0.0/11
        services: 100.64.0.0/13
        vpc:
          cidr: 10.250.0.0/16
        internal:
          - 10.250.112.0/26
          - 10.250.112.64/26
          - 10.250.112.128/26
        public:
          - 10.250.96.0/26
          - 10.250.96.64/26
          - 10.250.96.128/26
        workers:
          - 10.250.0.0/26
          - 10.250.0.64/26
          - 10.250.0.128/26
      workers:
        - name: worker-c92jz
          machineType: m4.large
          autoScalerMin: 3
          autoScalerMax: 12
          volumeType: gp2
          volumeSize: 50Gi
      zones:
        - eu-central-1a
        - eu-central-1b
        - eu-central-1c
  dns:
    provider: aws-route53
    hostedZoneID: Z1ZC826MD6OQ78
    domain: <GARDENER-NEW-CLUSTERNAME-TO-CREATE>.<GARDENER-EXISTING-PROJECT>.shoot.canary.k8s-hana.ondemand.com
  kubernetes:
    allowPrivilegedContainers: true
    version: 1.10.5
  maintenance:
    autoUpdate:
      kubernetesVersion: true
    timeWindow:
      begin: 230000+0000
      end: 000000+0000
```

After replacing all values you can now create a new cluster using kubectl.

```bash 
kubectl apply -f my-shoot.yaml
```

You can now go to your gardener dashboard and check the progress of your cluster creation.

### Deploy demo application

Now we deploy the simple demo application below and get the desired schedule of the pods in its very own zone.


![Screen](/images/deployment.png?raw=true "create_user")



```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
        ns:
          fieldRef:
            fieldPath: metadata.namespace
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [ "web-store"]
            topologyKey: "failure-domain.beta.kubernetes.io/zone"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

The important part is the `podAntiAffinity` and especially the `topologyKey`. Without the topology a schedule 
of the pod in the layout below ist possible. **This is then guaranteed not the desired result.**


![Screen](/images/deployment_wrong.png?raw=true "create_user")
