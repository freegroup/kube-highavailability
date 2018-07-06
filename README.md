## High Availability and multi-AZ guidance on AWS

To build a more production-ready configuration, users should consider:

 - running Kubernetes in a High Availability mode (out of scope)
 - running across multiple Availability Zones
 

## Overview
In this tutorial I cover the scenario below. 
 - One Cluster
 - Multi tenants each in its own namespace
 - **multiple availability zones**
 - each pod of a Deployment/DtatefulSet running in a different zone
 
 
![Screen](/images/overview.png?raw=true "overview")

## Setup
To use multiple AZ we need to setup a cluster with a customized shoot YAML  and add the AWS availability
zones and notwork segments by hand. But don't worry - there's no magic behind it

### Get a kubeconfig.yaml for a technical user
Fist of all we need a kubeconfig.yaml to access the gardener via `kubectl`. I assume you already know how to use 
kubectl and make a kubeconfig available. Below is a shoot screencast how you can download a new kubeconfig.yaml 
in the gardener dashboard.
 
![Screen](/images/create_tech_user.gif?raw=true "create_user")


### Create a custom shoot.yaml
Download and edit the shoot YAML below to your needs.

where:
 - **&lt;GARDENER-NEW-CLUSTERNAME-TO-CREATE&gt;** is the name of the cluster to create
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

``` 
kubectl apply -f my-shoot.yaml
```

You can now go to your gardener dashboard and check the progress of your cluster creation.

