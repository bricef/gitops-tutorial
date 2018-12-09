Kubecon Seattle 2018 Gitops Tutorial
====================================


XXX: about, slides, weaveworks, brice

Prerequisites
-------------

### A Kubernetes cluster

In order to conplete this tutorial, you will need a test kubernetes cluster. You have several options to quikcly get a development cluster to test out Gitops

1. Run [Minikube]() locally. This will require you to have [Virtualbox]() installed. This is great if you have virtualbox installed, but the download is over 2GB. See the [setup isntructions for Minikube](docs/minikube-install.md).
2. Use [Google Kubernetes Engine](). You'll need to have a google cloud instance set up to do this, and it will cost you a small amount to keep your cluster running. See the [setup instructions for GKE](docs/gke-install.md)
3. Set up a cluster on [DigitalOcean](). This will also cost you a little to keep your cluster running. You can follow the [setup instructions for Digital Ocean](docs/digitalocean-install.md) to do this.

You may have another way of getting a kubernetes cluster up and running. If that's the case, great! This tutorial should work with any valid kubernetes installation. 

Before you begin, you should make sure you have configured the `kubectl` command to point to your cluster and can query basic information. for example, run

```
➤ kubectl get pods --all-namespaces
```

To get a listing of all the running pod on your cluster. If this command completes succesfully, you're ready to go onto the next step.

### A github or bitbucket account


Getting started with Gitops
---------------------------

### 1. Clone this repository 
in order to control the operation of your cluster using gitops, you'll need to 

### 2. Install the flux operator on your cluster

### 3. Configure the operator to read from your repository
- Add deploy keys with github repo
- Configure the deploy operator to point to your cloned repository, to the `deploy/kubernetes` folder


### 4. Add a yaml file 
Now that the operator is set up to react to changes in your repository's `deploy/kubernetes` directory, we need to provide it with some configuration for it to synchronise.

### 5. Manually delete a deployment
Now that we know that the gitops operator is running in your cluster, let's check the control group 

### 6. Modify the configuration and commit your changes
- look at number of running pod for service
- scale up deployment in the yaml
- commit
- watch the cluster for the change

### 7. Automate deployment
- look at version of deployment
- add automation annotation
- watch to see the cluster update to latest version

### 6. Bonus: Deploy monitoring in a gitops way
- copy the monitoring folder yamls
- Look at the exposed service for grafana
- modify the dashboard script to add a new dashboard
- commit the changes.






Finally
-------

Now you're done with this tutorial, make sure you delete your cluster so you don't incur running costs. See the 

- [Teardown instructions for Minikube](docs/minikube-teardown.md)
- [Teardown instructions for GKE](docs/gke-teardown.md)
- [Teardown instructions for DigitalOcean](digitalocean-teardown.md)