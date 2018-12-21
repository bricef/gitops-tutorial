Kubecon Seattle 2018 Gitops Tutorial
====================================

This repository contains the hands-on Gitops tutorial given at Kubecon 2018 in Seattle. You can find the slides by clicking the image below. Follow the tutorial by reading through this README.

If you see errors, mistakes or typos in this tutorial, please feel free to submit a pull request.

[![](resources/presentation.png?raw=true)](https://docs.google.com/presentation/d/1ujRd4k2s8dG0-AMHIWMTyA8JoTUkXRQwXQ4izmDWeTI/edit?usp=sharing)

Prerequisites
-------------

In order to go through this tutorial, you'll need two things.

### 1. A Kubernetes cluster

In order to conplete this tutorial, you will need a test kubernetes cluster. You have several options to quickly get a development cluster to test out Gitops.

You can use [Katacoda's Kubernetes playground](https://www.katacoda.com/courses/kubernetes/playground). This will allow you to access a small Kubernetes cluster on demand. The `kubectl` command line is set up by default and _should be used from the master node_.

There are also plenty of other options, such as using [minikube](https://github.com/kubernetes/minikube), Using [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) or [EKS](https://eksctl.io/).

But you may have another way of getting a kubernetes cluster up and running. If that's the case, great! This tutorial should work with any valid kubernetes installation. 

Before you begin, you should make sure that the `kubectl` command works and you can query basic information from kubernetes. for example, run

```
➤ kubectl get pods --all-namespaces
```

To get a listing of all the running pod on your cluster, or

```
➤ kubectl cluster-info
```

To get a quick status check on your cluster. If these commands complete succesfully, you're ready to go onto the next step.

### 2. A Github or Bitbucket account
You'll want to create a new repository for this tutorial and make it available from your cluster through a git URL. If you're using Bitbucket, make sure you have [SSH keys enabled](https://confluence.atlassian.com/bitbucketserver/enabling-ssh-access-to-git-repositories-in-bitbucket-server-776640358.html) as we'll need these to configure the deploy operator access. 

Of course, you can also configure the Flux operator to [use a private git host](https://github.com/weaveworks/flux/blob/master/site/standalone-setup.md#using-a-private-git-host), but configuring this is outside the scope of this introductory tutorial.

Getting started with Gitops
---------------------------

### 1. Fork & Clone this repository 
In order to control the operation of your cluster using gitops, you'll need to have a control repository in which the state of your cluster can be defined. `github.com/bricef/gitops-tutorial` is already set up with all the files you'll need to follow along. Fork it to your Bitbucket or github account. When you have your own remote repository, clone it to your local workspace:

```
➤ git clone <url of your forked repository>
```


### 2. Install the flux operator on your cluster
Now that you have a repository set up, it's time to install the deployment operators on your cluster. For this we'll use Weavework's [Flux](https://github.com/weaveworks/flux). 

The needed deployment manifests are kept in this repository's `flux` directory. 

If you have `kubectl` configured correctly, you'll be able to install the Flux operator by applying the manifests to your cluster:

```
➤ cd gitops-tutorial
➤ kubectl apply -f ./flux/
```

You should see that this will create several objects in your cluster

```
serviceaccount/flux created
clusterrole.rbac.authorization.k8s.io/flux created
clusterrolebinding.rbac.authorization.k8s.io/flux created
deployment.apps/flux created
secret/flux-git-deploy created
deployment.apps/memcached created
service/memcached created
```

Alongside flux, we have also installed a memcache service for flux to use, and appropriate permissions for the flux operator to act on the cluster itself through a role and rolebinding. 

You should now be able to see the flux pod running in the cluster:

```
➤ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
flux-9d69f6fc4-5t5w6        1/1       Running   0          18m
memcached-dbb59cb58-phk9s   1/1       Running   0          18m
```

Now that the operator is running in your cluster, we'll need to configure it to have access to your repository, so that it can read changes you make, as well as create commits when a service is automatically deployed.

To do this, we'll need to do two things. Firstly, we'll need to allow the flux operator to act on the repository; secondly, we'll need the flux operator to be configured to look for cluster configuration in a particular repository.

### 3. Sharing the operator's public key with the remote git repository
To allow flux to operate on the repository, we'll need to copy its public SSH key to Github or Bitbucket. 

We can get the operator's public key using the [fluxctl](https://github.com/weaveworks/flux/blob/master/site/fluxctl.md) tool.

First, we'll need to install the `fluxctl` tool. There are [different ways of installing it](https://github.com/weaveworks/flux/blob/master/site/fluxctl.md#installing-fluxctl) depending on your platoform. For this tutorial, we can grab the binary release directly.

```
➤ wget https://github.com/weaveworks/flux/releases/download/1.8.1/fluxctl_linux_amd64
➤ chmod +x fluxctl_linux_amd64
```

Once installed and executable, we can use the fluxctl command line tool to get the Flux operator's public key:

```
➤ ./fluxctl_linux_amd64 identity
```

The command should provide you with a public key:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCl05 [...]
```
You now need to copy this key and add it to github or bitbucket as a R/W key for your remote repository.

See the documentation on [how to do this for Github and Bitbucket](docs/adding-write-keys.md). Bitbucket will require the keys to be added to your profile instead of your repository.

### 4. Configure the operator to read from your repository
Now that your repository will accept read and write requests from the flux operator, we should configure Flux to use your remote repository. To do this, we'll reconfigure the Flux deployment.

Edit the `flux/flux-deployment.yaml` file to reconfigure the Flux deployment with the url of your repository by changing the arguments. Make sure to add the `--git-path=deploy/kubernetes` argument as well to point the Flux operator to the right folder. 

```diff
diff --git a/flux/flux-deployment.yaml b/flux/flux-deployment.yaml
index 15d0f31..c57d3b0 100644
--- a/flux/flux-deployment.yaml
+++ b/flux/flux-deployment.yaml
@@ -100,8 +100,9 @@ spec:
         - --ssh-keygen-dir=/var/fluxd/keygen

         # replace or remove the following URL
-        - --git-url=git@github.com:weaveworks/flux-get-started
+        - --git-url=<GIT URL OF YOUR REPOSITORY>
+        - --git-path=deploy/kubernetes
         - --git-branch=master


         # include these next two to connect to an "upstream" service
         # (e.g., Weave Cloud). The token is particular to the service
```

Once you've edited and saved the file, reapply the configuration to your cluster. 

```
➤ kubectl apply -f flux/flux-deployment.yaml
```

### 5. Add a yaml file 
Now that the operator is set up to react to changes in your repository's `deploy/kubernetes` directory, we need to provide it with some configuration for it to synchronise.

We have a template deployment in the `examples` directory. Copy this example to the `deploy/kubernetes` directory that Flux is watching, then commit and push your changes to your remote repository.

```
➤ cp examples/podinfo-dep.yaml deploy/kubernetes
➤ git add deploy
➤ git commit -m "Add new deployment to cluster"
➤ git push origin master
```

If everything has been configured correctly so far, you should be able to watch for changes in your cluster and see the `podinfo` deployment running after a little while.

```
➤ watch kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
flux-6f9798f7f-6hcc9        1/1       Running   0          12m
memcached-dbb59cb58-gn8pf   1/1       Running   0          16m
```

Then after a few minutes

```
NAME                        READY     STATUS    RESTARTS   AGE
flux-6f9798f7f-6hcc9        1/1       Running   0          12m
memcached-dbb59cb58-gn8pf   1/1       Running   0          16m
podinfo-6466ffddb-l9nq2     1/1       Running   0          3m
```

### 6. Manually delete a deployment
Now that we know that the gitops operator is running in your cluster, we should expect it to return the system to a know good state if we make an operator error. Let's try this by manually deleting the `podinfo` deployment.

```
➤ kubectl delete deploy/podinfo
```

This will delete the deployment from the cluster. We can see this by looking at the deployments, where the `podinfo` deployment should be missing.

```
➤ kubectl get deployments
```

We haven't changed the state of the repository, however, so the Flux operator still expects the deployment to be running. Since it's missing from our cluster, Flux will attempt to recreate it. We can see this if we monitor the deployments:

```
➤ watch kubectl get deployments
```

After a few minutes (be patient, it will get there), a new deployment will be created to replace the missing `podinfo` one.

This is an example of how the Flux operator is acting as a control loop for your cluster, always bringing it back to the desired state if possible.

### 7. Modify the configuration and commit your changes
Now we know that the flux operator is working, let's see it act on changes to our repository.

We'll scale up our podinfo deployment to multiple replicas in our configuration repository, and we will see how Flux automatically applies this new configuration.

First, let's take a look at the podinfo deployment

```
➤ kubectl get deploy/podinfo
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
podinfo   1         1         1            1           11m
```

As we can see, the deployment currently only has one pod. Let's change this in the deployment yaml (`deploy/kubernetes/podinfo-dep.yaml`).

```diff
diff --git a/deploy/kubernetes/podinfo-dep.yaml b/deploy/kubernetes/podinfo-dep.yaml
index e79ce86..8cd6c13 100644
--- a/deploy/kubernetes/podinfo-dep.yaml
+++ b/deploy/kubernetes/podinfo-dep.yaml
@@ -4,7 +4,7 @@ kind: Deployment
 metadata:
   name: podinfo
 spec:
-  replicas: 1
+  replicas: 4
   template:
     metadata:
       labels:
```

Now save and commit the changes.

We can watch Flux automatically update the deployment. Be patient as this can take a few minutes.

```
➤ watch kubectl get deploy/podinfo
```

We've now seen Flux react to changes in the repository. Any new kubernetes configuration yaml placed in the `deploy/kubernetes` folder and any future commited changes will now be automatically be applied to our cluster. Goodbye `kubectl apply`!

### 8. Automate deployment
Making manual changes to the deployment yaml is tedious, especially if we always want to deploy the latest version of our application, ie, do continuous deployment. 

Instead, Flux can monitor the image repository and whenever a new image matches a specified filter, deploy it to our cluster automatically.

Let's take a look at the current running version of podinfo in our cluster:

```
➤ kubectl get deploy/podinfo -o jsonpath='{..spec.template.spec.containers[0].image}{ "\n"}'
```

Now let's tell Flux to automatically update to the latest version using an annotation in the deployment:

```diff
diff --git a/deploy/kubernetes/podinfo-dep.yaml b/deploy/kubernetes/podinfo-dep.yaml
index 8cd6c13..334dac9 100644
--- a/deploy/kubernetes/podinfo-dep.yaml
+++ b/deploy/kubernetes/podinfo-dep.yaml
@@ -3,6 +3,9 @@ apiVersion: apps/v1beta1 # for versions before 1.6.0 use extensions/v1beta1
 kind: Deployment
 metadata:
   name: podinfo
+  annotations:
+      flux.weave.works/tag.podinfo: glob:*
+      flux.weave.works/automated: 'true'
 spec:
   replicas: 4
   template:
```

These will tell flux to automatically deploy the latest available image. Once we have saved, committed and pushed these changes to our remote repository, flux will deploy the latest version automatically.

We can watch this as usual:

```
➤ watch kubectl get deploy/podinfo -o jsonpath='{..spec.template.spec.containers[0].image}{ "\n"}'
```

Flux will actually create a new commit in your control repository, with the new version of the podinfo. Once we have the new version deployed we can look at the effect on the repository by pulling the changes Flux made:

```
➤ git pull
➤ git log -p
```

Which will show the commit made by Flux to deploy the latest version.

```diff
diff --git a/deploy/kubernetes/podinfo-dep.yaml b/deploy/kubernetes/podinfo-dep.yaml
index 334dac9..b5b4e6d 100644
--- a/deploy/kubernetes/podinfo-dep.yaml
+++ b/deploy/kubernetes/podinfo-dep.yaml
@@ -15,6 +15,6 @@ spec:
     spec:
       containers:
       - name: podinfo
-        image: quay.io/stefanprodan/podinfo:1.0.1
+        image: quay.io/stefanprodan/podinfo:1.4.1
         ports:
         - containerPort: 3000
```

Now that our deployment is automated, every new version of podinfo will be commited to our control repository,  and then automatically deployed to our cluster.

We can specify the filter used by Flux to choose images to deploy automatically by changing the `flux.weave.works/tag.dotnet-load-tester` annotation value. For example, `glob:master-*` will only deploy images tagged with the prefix `master-`. 

Flux is also capable of filtering on semantic versions. for exmaple, setting `flux.weave.works/tag.dotnet-load-tester` to `semver:>0.1.1` will only deploy tagged versions with semantic version greater than 0.1.1.

You can also use regular expressions to filter out the images for even more power. Use the format `regexp:...` with a custom regexp instead of `...`.

<!--
### Bonus: Deploy monitoring in a gitops way
- copy the monitoring folder yamls
- Look at the exposed service for grafana
- modify the dashboard script to add a new dashboard
- commit the changes.

### Bonus: Deploy your own workload
-->

Closing remarks
---------------

**Polling delays**

You might find that the delays between changes to the repository and the changes to the cluster are too long. By default, Flux is configured to poll the Git repository every 5 minutes, so this is quite slow.

It's probably not a good idea to poll more often, as it's surprisingly easy to go over cloud provider API limits. However, we can speed up the process another way.

[Weave Cloud](https://www.weave.works/product/cloud/) comes with the option of adding webhooks to notify flux of changes in the git repository or the image registry, so that as soon as changes are made, the Flux daemon will update the cluster.

**Helm**

We didn't include a Helm demo as that's a little more involved. However, you can use the same approac with Helm, but you'll have to install a custom operator on the cluster that's able to communicate between Flux an Helm's Tiller. You can find out more in [the gitops-helm project](https://github.com/stefanprodan/gitops-helm).

**Gitops Flux**

Noticeably, we don't manage Flux using Flux itself. This could easily get us in a bind when updating and we'd loose the ability to control our cluster. We'll always need a service to stand outside of the Gitops to provide the ability to manage the cluster, and it won't be able to manage itself safely. That's ok. 

The commercial [Weave Cloud](https://www.weave.works/product/cloud/) agents will automatically update, by using a separate update path managed through a deployed service.

**Automatically deleting services**

Flux won't automatially delete services that are running on the cluster but don't exist in the control repository. While that might seem like a good idea at first, this would not allow Flux to be deployed to to existing clusters, as it would immediately delete all running deployments as soon as installed! Instead, you'll have to manually delete resources after removing them from the control repository.

**Custom Resource Definitions**

Flux will play well with CRDs. If you have a CRD definition or a custom resource yaml in the control repository, then they will be applied to the cluster like any standard resources. 

Further resources
-----------------

You can find much more about Gitops online.

- 📄 The [original Gitops blog post](https://www.weave.works/blog/gitops-operations-by-pull-request)
- 📄 A [more recent follow up blog post](https://www.weave.works/blog/what-is-gitops-really)
- 📄 A [comprehensive Gitops primer](https://www.weave.works/technologies/gitops/)
- 📄 The [Gitops FAQ](https://www.weave.works/technologies/gitops-frequently-asked-questions/)
- 🎥 A [recorded presentation on Gitops and its implementation at Weaveworks](https://vimeo.com/293138562/5aa199fa9e)




