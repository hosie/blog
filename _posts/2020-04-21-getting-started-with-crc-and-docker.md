---
layout: post
title:  "Getting Started with Code Ready Containers and Docker"
date:   2020-04-21
tags: [Kubernetes, OpenShift, Docker, Code Ready Containers]
---

Code Ready Containers is a version of Red Hat OpenShift Container platform that runs on your laptop. By following this guide, you will be able to set it up on your laptop, push images to its image registry from your existing local docker registry and run those images in pods on the CRC cluster.

When first moving to or evaluating OCP, you might want to take small steps.  If you are like me, you will like to start from what you know and try new things a little at a time.

I was using docker desktop for Mac and I would tend to write code, test it, build an image (using `docker build`), test the image on my local docker engine (using `docker run`), then deploy that image into my Docker Desktop Kubernetes cluster (using `kubectl run`).  

![Previous Workflow]({{ site.url }}{{ site.baseurl}}/assets/previous-workflow-local-k8s-workflow.png)

My eventual production environment is Red Hat OpenShift Container Platform so I wanted to try out Code Ready Containers as my local Kuberenetes cluster. I figured that I should probably swap `kubectl run` for `oc run`. I would probably need to do a `docker push` at some point given that CRC has its own image registry. If I could mimimise my first step of exploration to just those changes, then I would still be in my comfort zone and I would gradually be expanding that comfort zone.

![Baby Step Towards CRC Workflow]({{ site.url }}{{ site.baseurl}}/assets/baby-step-towards-crc-workflow.png)


OCP has a build system as part of the platform and some day I should learn and embrace that.  For now, I am just looking for the shortest path to get my docker images running on CRC. I want to stray as little as possible from my existing developer workflow.

CRC also comes with `podman` binary which has some advantages over the `docker` cli. I will probably learn that later.  For now I just want to run an image that is currently on my local docker registry.

## The Challenge

I couldn't find any quick start guide that got me to the point of being able to deploy an image that I had built myself. The main challenges I found was configuring my docker CLI to accept the CRC self signed certificate and getting the correct password to use on the `docker login` command.

I was getting errors like
```
x509: certificate signed by unknown authority
```
and
```
Error response from daemon: Get https://default-route-openshift-image-registry.apps-crc.testing/v2/: unauthorized: authentication required
```

After a fair bit of searching, I eventually figured it out. This guide lists all of the steps I had to follow along with some notes and some gotchas that I encountered along the way.

## The Solution

### Environment and Versions
I have tested the instructions in this guide with:
 - MacOS Catalina 10.15.3
 - Docker version 19.03.8, build `afacb8b`  (from Docker Desktop version 2.2.0.5)
 - OpenShift Container Platform Code Ready Containers version 1.8.0

### Steps

#### Install and Run CRC

The official [Getting Started Guide](https://code-ready.github.io/crc/#introducing-codeready-containers_gsg) sections 1 - 3.3.2 are ideal for this step.  I suggest that you follow those and then come back here for the next part.

If you have followed that guide, you have probably ran the following commands after [downloading the latest release of CRC](https://cloud.redhat.com/openshift/install/crc/installer-provisioned)
```bash
export PATH=$PATH:/Users/johnhosie/tools/ocp/crc-macos-1.8.0-amd64
crc setup
crc start
eval $(crc oc-env)
oc login -u developer -p developer https://api.crc.testing:6443
```

**NOTE**: during the `crc start` process, you will be asked for an image pull secret.  You can get this from [https://cloud.redhat.com/openshift/install/crc/installer-provisioned](https://cloud.redhat.com/openshift/install/crc/installer-provisioned) after logging in with your Red Hat credentials.
{: .notebox }


**GOTCHA**: I did hit one hiccup while doing this. The networking from the cluster seemed not to work if I was connected to my company's VPN.  I noticed that the cluster got into a bad state if I was connected to the VPM while either creating and or running the cluster. I never got to the bottom of this and it might be a red herring. It seems to work fine if I delete the cluster and then start from scratch while not connected to the VPN. See [this issue](https://github.com/code-ready/crc/issues/1162).
{: .gotcha }


####  Deploy hello world
Before you start deploying an image from your own registry, let's take a smaller step. Let's deploy one from Dockerhub. Hello World is a good place to start.

```yaml
oc apply -f - <<EOF
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  creationTimestamp: null
  labels:
    run: hello-world
  name: hello-world
spec:
  replicas: 1
  selector:
    run: hello-world
  strategy:
    resources: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: hello-world
    spec:
      containers:
      - image: hello-world:latest
        name: hello-world
        resources: {}
  test: false
  triggers: null
status:
  availableReplicas: 0
  latestVersion: 0
  observedGeneration: 0
  replicas: 0
  unavailableReplicas: 0
  updatedReplicas: 0
EOF
```

viewing the logs of the resulting pod
```
oc logs $(oc get pods -l deploymentconfig=hello-world -o jsonpath="{.items[0].metadata.name}")
```

should return
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```
####  Login to the OCP Image Registry from your Docker client
The image registry uses a self signed certificate for TLS. You need to fetch that certificate and add it to your keychain so that your docker CLI will trust the image registry server.

If you do not do this step, then you will see the following error:
```
x509: certificate signed by unknown authority
```


To do this next step, you need to be logged in as `kubeadmin`.  Run `crc console --credentials` to get the password and then use it with `oc login -u kubeadmin -p <password-copied-from-crc-console--credentials> https://api.crc.testing:6443`  or run this handy one liner
```bash
oc login -u kubeadmin -p $(crc console --credentials | grep kubeadmin | sed -E "s/.*-p (.*) .*/\1/g") https://api.crc.testing:6443
```
Then you can extract the certificate and add it to your keychain
```
oc extract secret/router-ca --keys=tls.crt -n openshift-ingress-operator
security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain tls.crt
```

Alternatively, you can use the keychain UI rather than the `security add-trusted-cert` command.

<div class="notebox" markdown="1">
**NOTE**: The above instructions apply to Mac OS. If you are using Linux, you would do something like
```
cp tls.crt /etc/docker/certs.d/default-route-openshift-image-registry.apps-crc.testing/
```
 instead of using the `security add-trusted-cert` command
</div>

Now, you should be able to run `docker login` to connect your docker CLI to the image registry on CRC
```bash
docker login -u kubeadmin -p $(oc whoami -t) default-route-openshift-image-registry.apps-crc.testing
```

<div class="gotcha" markdown="1">
**GOTCHA**: make sure you use the `token` returned from `oc whoami -t` rather than the developer password.
Otherise, you will see the following error
```
Error response from daemon: Get https://default-route-openshift-image-registry.apps-crc.testing/v2/: unauthorized: authentication required
```
</div>


####  Deploy an image from your local docker registry

Small steps again.  Before we start building a new image, let's pull the Hello World image and push that.
```bash
docker pull hello-world:latest
docker tag hello-world:latest default-route-openshift-image-registry.apps-crc.testing/default/hello-world:latest
docker push default-route-openshift-image-registry.apps-crc.testing/default/hello-world:latest
```

####  Run the image in a container on the cluster
This is similar to when we ran Hello World before.  The only difference here is the `image` property. Note that the repo is `image-registry.openshift-image-registry.svc:5000` even though we pushed it to `default-route-openshift-image-registry.apps-crc.testing`.  The former is how the image registry is referenced externally and the latter is how it is referenced from within the cluster.
```yaml
oc apply -f - <<EOF
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  creationTimestamp: null
  labels:
    run: hello-world
  name: hello-world
spec:
  replicas: 1
  selector:
    run: hello-world
  strategy:
    resources: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: hello-world
    spec:
      containers:
      - image: image-registry.openshift-image-registry.svc:5000/default/hello-world:latest
        name: hello-world
        resources: {}
  test: false
  triggers: null
status:
  availableReplicas: 0
  latestVersion: 0
  observedGeneration: 0
  replicas: 0
  unavailableReplicas: 0
  updatedReplicas: 0
EOF
```

####  Running your own image

Now you should know what to do to run your own image.  Simply tag the image using `docker tag` and then `docker push` it. Be sure to set the repository to `default-route-openshift-image-registry.apps-crc.testing` and set the namespace to match the OpenShift project ( kubernetes namespace) that you intend to deploy it to. In the examples above, we deployed to the `default` project so we tagged the image with `default-route-openshift-image-registry.apps-crc.testing/default/`


## Next Steps
Now you have taken your first step and not deviated much from you previous developer work flow, there are a few things that you should consider exploring to step further into the OpenShift way of doing things
 - Use the `developer` identity.  In this guide, we performed all actions as `kubeadmin`. As a good practice, it is better to get familiar with role based access and perform as much as possible with the `developer` user. You will need to learn about [Authentication](https://docs.openshift.com/container-platform/4.3/authentication/understanding-authentication.html) and configure the relevant access for the `developer` user on your CRC cluster.   
 - Create [BuildConfigs](https://docs.openshift.com/container-platform/4.3/builds/understanding-image-builds.html) to declaratively specify to CRC how your images should be built.
 - Explore some of the [operators available](https://operatorhub.io/) to simplify the management of the cluster and workloads deployed to it.
 - In particualr, explore the [Knative Serving Operator](https://operatorhub.io/operator/knative-serving-operator) and the tech preview of the [OpenShift Serverless operator](https://www.openshift.com/learn/topics/serverless) for a developer friendly way to deploy and manage your own workloads on OpenShift Container platform


 <script src="https://utteranc.es/client.js"
         repo="hosie/blog"
         issue-term="pathname"
         label="BlogComment"
         theme="github-light"
         crossorigin="anonymous"
         async>
 </script>
