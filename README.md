## High Availability and multi-AZ guidance on AWS

To build a more production-ready configuration, users should consider:

 - running Kubernetes in a High Availability mode (out of scope)
 - running across multiple Availability Zones
 

## Overview
In this tutorial I cover the scenario below. 
 - One Cluster
 - Multi tenants each in its own namespace
 - multiple availability zones
 
 
![Screen](/images/overview.png?raw=true "overview")

## Setup
To use multiple AZ we need to setup a cluster with a customized shoot YAML  and add the AWS availability
zones by hand.

### get a kubeconfig.yaml for a technical user
 
 
![Screen](/images/create_tech_user.gif?raw=true "create_user")
