---
Categories:
- Kubernetes
Description: ""
Tags:
- Kubernetes
- Helm
- CI/CD
date: 2017-02-20T15:21:26+01:00
menu: main
title: release_mangement_with_helm
draft: true
---

In this post I will be showing how to setup CI/CD to a Kubernetes cluster using Helm and Jenkins.

#### Preface

What is Helm? Well, think of it as the apt-get / yum of Kubernetes, it's a package manager for Kubernetes
developed by the guys from Deis (https://deis.com/). If you deploy applications to Kubernetes, Helm makes
it incredibly easy to version those deployments, package it and easily deploy, un deploy and even rollback
those deployments as charts. Charts being the terminology that `helm` use for package.


#### Building a Helm chart

Let's see Helm in action, using a small little Go test api I created specifically for use cases like this, let's
build a helm chart of it.

```
git clone https://github.com/daemonza/testapi.git; cd testapi
```

Let's create a skeleton structure chart

```
helm create testapi_chart
```

This will create a `testapi_chart` directory. Inside this directory the two files we are the most interested in is
Chart.yaml and values.yaml

* Chart.yaml describes the chart, as in it's name, description and version.
* values.yaml is stores variables for the Kubernetes specification deployment.yaml Go template in the templates directory.
Keep that in mind when your editing the values.yaml file, as you might have to edit the deployment.yaml template if you
have a more complex deployment needed as provided by the default template.

Open Chart.yaml and fill out the details of the application your deploying. Using the `testapi` as a example, this is
how my Chart.yaml looks like :

```
apiVersion: v1
description: A simple api for testing and debugging
name: testapi_chart
version: 0.0.1
```

Now open `values.yaml` and edit it as needed. Again, using the `testapi` as a example this is how my `values.yaml` file
looks like. 

```
replicaCount: 2
image:
  repository: docker.io/daemonza/testapi
  tag: latest
  pullPolicy: IfNotPresent
service:
  name: testapi
  type: ClusterIP
  externalPort: 80
  internalPort: 80
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

 Run 
 ```
 helm lint
 ```
 in your testapi_chart directory to make sure everything is ok.
If everything is good, change to the root of the testapi repository and 
package the chart with
 
```
helm package testapi_chart --debug
```

I like to add the `--debug` flag to see the output of the packaged chart.
Output should look simialer to the following

```
Saved /Users/daemonza/testapi/testapi_chart-0.0.1.tgz to current directory
Saved /Users/daemonza/testapi/testapi_chart-0.0.1.tgz to /Users/daemonza/.helm/repository/local
```
From that we can see that the chart is placed in our current directory as well as in our local
helm repository.

#### Setting up Helm repository

https://github.com/daemonza/helmet
 

#### Setting up Jenkins 

Go to your Jenkins dashboard, if you don't have a running Jenkins or just want to try out this setup, you
can run Jenkins in a quick and easy way locally with:
```
docker run -p 8080:8080 -p 50000:50000 jenkins
```

Now we need a way for Jenkins to deploy our `Helm Charts`, luckily as with almost anything, there is a 
Jenkins plugin available to help us out.

In Jenkins go to `Manage Jenkins->Manage Plugins` and on the `Available` tab search and install the
`kubernetes-ci` plugin, which is this (https://github.com/jenkinsci/kubernetes-ci-plugin).

We now need to configure the plugin to know about our Kubernetes cluster. Go to
`http://<YOUR JENKINS SERVER>/configure` scroll down to the bottom `Cloud` section, and add your
Kubernetes cluster details. For the `Credentials` select `Token` and paste in your Kubernetes token. You can
get your token running the following command :

```
kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t'
```

Now configure the `Chart repository`

TODO - This is my caddy driven chart repo on github. Link na boonste section "Setting up Helm repository"
TODO - jy moet helm repo update run nadat jy iets na die helm repo ge-upload het 

#### Conclusion