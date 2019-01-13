---
title: "WebLogic on OpenShift"
date: 2019-01-11T08:30:16-05:00
draft: true
toc: true
tags: [weblogic,openshift]
---
In this post I am going to walk through setting up and using WebLogic on 
OpenShift, using the [Oracle WebLogic Server Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator).  My starting point is the OpenShift Container Platform server 
that I set up on OCI in [this earlier post](/posts/openshift).

## Overview

Here's an overview of the process I am going to walk through:

* Create a new project (namespace) where I will be deploying WebLogic,
* Prepare the project for the WebLogic Kubernetes Operator, 
* Install the operator, 
* View the operator logs in Kibana, 
* Prepare Docker images to run my domain, 
* Create the WebLogic domain, 
* Verify access to the WebLogic administration console and WLST, 
* Deploy a test application into the cluster, 
* Set up a route to expose the application publicly,
* Test scaling and load balancing, and
* Install the WebLogic Exporter to get metrics into Prometheus.

Before we get started, you should clone the WebLogic operator project 
from GitHub.  It contains many of the samples and helpers we will need.

```
git clone https://github.com/oracle/weblogic-kubernetes-operator
```


## Create a new project (namespace) 

In the OpenShift web user interface, create a new project.  If you already 
have other prjects, go to the Application Console, and then click on the 
project pulldown at the top and click on "View All Projects" and then the 
"Create Project" button.  If you don't have existing projects, OpenShift 
will take you right to the create project page when you log in.  I called
my project "weblogic" as you can see in the image below:

{{< figure src="/images/wls-on-os001.png" caption="Creating a new project" >}}

Then navigate into your project view.  Right now it will be empty, as shown
below:

{{< figure src="/images/wls-on-os002.png" caption="The new 'weblogic' project" >}}


## Prepare the project for the WebLogic Kubernetes Operator 

The easiest way to get the operator Docker image is to just pull it from the 
Docker Store.  You can review details of the image in the 
[Docker Store](https://hub.docker.com/r/oracle/weblogic-kubernetes-operator).

{{< figure src="/images/wls-on-os003.png" caption="The WebLogic Kubernetes Operator in the Docker Store" >}}

You can use the following command to pull the image:

```
docker pull oracle/weblogic-kubernetes-operator:2.0
```

Instead of pulling the image and manually copying it onto our OpenShift nodes, 
we could also just add an Image Pull Secret to our project (namespace) so 
that OpenShift will be able to pull the image for us.  We can do this
with the following commands (at this stage we are using a user with `cluster-admin`):

```
oc project weblogic
oc create secret docker-registry docker-store-secret \
   --docker-server=store.docker.com \
   --docker-username=DOCKER_USER \
   --docker-password=DOCKER_PASSWORD \
   --docker-email=DOCKER_EMAIL
```

In this command, replace `DOCKER_USER` with your Docker store userid, 
`DOCKER_PASSWORD` with your password, and `DOCKER_EMAIL` with the email
address associated with your Docker Store account. 

We also need to tell OpenShift to link this secret to our service account.
Assuming we want to use the `default` service account in our `weblogic` 
project (namespace), we can run this command:

```
oc secrets link default docker-store-secret --for=pull
```

## (Optional) Build the image yourself

It is also possible to build the image yourself, rather than pulling it 
from Docker Store.  If you want to do that, first go to Docker Store and 
accept the license for the [Server JRE image](https://hub.docker.com/_/oracle-serverjre-8), 
ensure you have the [listed 
prerequisites](https://github.com/oracle/weblogic-kubernetes-operator/blob/master/site/user-guide.md#prerequisites) installed, and then run these commands:

```
mvn clean install 
docker build -t weblogic-kubernetes-operator:2.0 --build-arg VERSION=2.0 .
```

## Install the ELK stack 

The operator can optionally send its logs to Elasticsearch and Kibana.
This provides a nice way to view the logs, and to search and filter and so on,
so let's install this too.  

A sample YAML file is provided in the project to install them:

```
kubernetes/samples/scripts/elasticsearch-and-kibana/elasticsearch_and_kibana.yaml
```

Edit this file to set the namespace to "weblogic" for each deployment and 
service (i.e. on lines 30, 55, 74 and 98 (just search for `namespace`) 
and then install them using this command: 

```
oc apply -f kubernetes/samples/scripts/elasticsearch-and-kibana/elasticsearch_and_kibana.yaml
```

After a few moments, you should see the pods running in our namespace:

```
oc get pods,services
NAME                                 READY     STATUS    RESTARTS   AGE
pod/elasticsearch-75b6f589cb-c9hbw   1/1       Running   0          10s
pod/kibana-746cc75444-nt8pr          1/1       Running   0          10s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/elasticsearch   ClusterIP   172.30.143.158   <none>        9200/TCP,9300/TCP   10s
service/kibana          NodePort    172.30.18.210    <none>        5601:32394/TCP      10s
```

So based on the service shown above and our project (namespace) named `weblogic`, 
the URL for Elasticsearch will be `elasticsearch.weblogic.svc.cluster.local:9200`.
We will need this URL later. 

## Install the operator 

Now we are ready to install the operator.  In the 2.0 release, we use Helm to
install the operator.  So first we need to download Helm and set up Tiller on 
our OpenShift cluster (if you have not already installed it). 

Helm provide [installation instructions](https://github.com/kubernetes/helm/blob/master/docs/install.md)
on their site.  I just downloaded the latest [release](https://github.com/helm/helm/archive/v2.12.1.zip),
unzipped it, and made `helm` executable.

Before we install tiller, let's create a cluster role binding to make sure the 
`default` service account in the `kube-system` namespace (which tiller will run
under) has the `cluster-admin` role, which it will need to install and manage
the operation. 

```
cat << EOF | oc apply -f -
apiVersion: authorization.openshift.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller-cluster-admin
roleRef:
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
userNames:
- system:serviceaccount:kube-system:default
EOF
```

Now we can execute `helm init` to install tiller on the OpenShift cluster.
Check it was successful with this command:

```
oc get deploy -n kube-system
NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy                1         1         1            1           18s
```

When you install the operator you can either pass the configuration parameters
into helm on the command line, or if you prefer, you can store them in a YAML
file and pass that file in.  I like to store them in a file.  There is a 
sample provided, so we can just make a copy and update it with our details.

```
cp kubernetes/charts/weblogic-operator/values.yaml my-operator-values.yaml
```

Here are the updates we need to make: 

* Set the `domainNamespaces` paraemeter to include just `weblogic`, i.e. the
  project (namespace) that we created to install WebLogic in. 

{{< highlight bash "linenos=table,hl_lines=2,linenostart=19" >}}
domainNamespaces:
  - "weblogic"
{{< / highlight >}}

* Set the `image` parameter to match the name of the image you pulled from 
  Docker Store or built yourself.  If you just create the image pull secret
  then use the value I have shown here: 

{{< highlight bash "linenos=table,hl_lines=2,linenostart=22" >}}
# image specifies the docker image containing the operator code.
image: "oracle/weblogic-kubernetes-operator:2.0"
{{< / highlight >}}

* Set the `imagePullSecrets` list to include the secret we created earlier.
  If you did not create the secret you can leave this commented out.

{{< highlight bash "linenos=table,hl_lines=2,linenostart=35" >}}
imagePullSecrets:
- name: "docker-store-secret"
{{< / highlight >}}

* Set the `elkIntegrationEnabled` parameter to `true`.

{{< highlight bash "linenos=table,hl_lines=2,linenostart=90" >}}
# elkIntegrationEnabled specifies whether or not ELK integration is enabled.
elkIntegrationEnabled: true
{{< / highlight >}}

* Set the `elasticSearchHost` to the address of the Elasticsearch server
  that we set up earlier.

{{< highlight bash "linenos=table,hl_lines=3,linenostart=97" >}}
# elasticSearchHost specifies the hostname of where elasticsearch is running.
# This parameter is ignored if 'elkIntegrationEnabled' is false.
elasticSearchHost: "elasticsearch.weblogic.svc.cluster.local"
{{< / highlight >}}

Now we can use helm to install the operator with this command, notice
that I pass in the name of my parameters YAML file in the `--values` option:

```
helm install kubernetes/charts/weblogic-operator \
     --name weblogic-operator \
     --namespace weblogic \
     --values my-operator-values.yaml \
     --wait
```

This command will wait until the operator starts up successfully.  If it has
to pull the image, that will obviously take a little while, but if this command
does not finish in say a minute or so, then it is probably stuck.  You can 
send it to the background and start looking around to see what went wrong.
Most often it will be a problem pulling the image.  If you see the pod has
status `ImagePullBackOff` then OpenShift was not able to pull the image. 

You can verify the pod was created with this command: 

```
oc get pods
NAME                                READY     STATUS    RESTARTS   AGE
elasticsearch-75b6f589cb-c9hbw      1/1       Running   0          2h
kibana-746cc75444-nt8pr             1/1       Running   0          2h
weblogic-operator-54d99679f-dkg65   1/1       Running   0          48s
```


## View the operator logs in Kibana 

Now we have the operator running, let's take a look at the logs in Kibana.
We installed Kibana earlier.  Let's expose Kibana outside our cluster:

```
oc expose service kibana
route.route.openshift.io/kibana exposed
```

You can check this worked with these commands:

```
oc get routes
NAME      HOST/PORT                                                      PATH      SERVICES   PORT      TERMINATION   WILDCARD
kibana    kibana-weblogic.sub11201828382.certificationvc.oraclevcn.com             kibana     5601                    None

oc get services
NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
elasticsearch                    ClusterIP   172.30.143.158   <none>        9200/TCP,9300/TCP   2h
internal-weblogic-operator-svc   ClusterIP   172.30.252.148   <none>        8082/TCP            7m
kibana                           NodePort    172.30.18.210    <none>        5601:32394/TCP      2h
```

Now you should be able to access Kibana using the OpenShift front-end address
and the node port for the Kibana service.  In my case the node port is `32394`
and my OpenShift server is accessible to me as `openshift`
so I would use the address `https://openshift:32394`.

You will see a page like this one:

{{< figure src="/images/wls-on-os004.png" caption="The initial Kibana page" >}}

Click on the "Create" button, then click on the "Discover" option in the menu 
on the left hand side.  Now hover over the entries for `level` and `log` in the
field list, and click on the "Add" button that appears next to each one. 

Now you should have a nice log screen like this:

{{< figure src="/images/wls-on-os005.png" caption="The operator logs in Kibana" >}}

Great! We have the operator installed.  Now we are ready to move on to create
some WebLogic domains.

## Prepare Docker images to run the domain 

Now we have some choices to make.  There are two main ways to run WebLogic 
in Docker - we can use a standard Docker image which contains the WebLogic
binaries but keep the domain configuration, applications, etc., outside the
image, for example in a persistent volume; *or* we can create Docker images
with both the WebLogic binaries and the domain burnt into them.  There are 
advantages and disadvantages to both approaches, so it really depends on 
how we want to treat our domain.  

The first approach is good if you just 
want to run WebLogic in Kubernetes but you still want to use the admin 
console and WLST and so on to manage it.  The second approach is better 
if you want to drive everything from a CI/CD pipeline where you do not 
mutate the running environment, but instead you update the source and then 
build new images and roll the environment to uptake them. A number of these
kinds of considerations are listed [here](https://github.com/oracle/weblogic-kubernetes-operator/blob/master/site/domains.md).  

For the sake of this post, let's use the "domain in image" option (the 
second approach).

So we will need a base WebLogic image with the necessary patches installed,
and then we will create our domain on top of that.  Let's create a domain
with a web application deployed in it, so that we have something to use
to test our load balancing configuration and scaling later on. 

The easiest way to get the base image is to grab it from Oracle using
this command:

```
docker pull waiting-for-details-from-monica
```

Of course, you could also take the standard 
[WebLogic Server 12.2.1.3.0](https://hub.docker.com/_/oracle-weblogic-server-12c) 
image from Docker Store and then install the patches on top of that.  This is 
worth knowing how to do, in case you need some additional one-off patches.
If you are not interested in that, skip forward to 
[here](#creating-the-image-with-the-domain-in-it).

### (Optional) Manually creating a patched WebLogic image

Here is an example `Dockerfile` that we can use to install the necessary
patches.  You can modify this to add any additional one-off patches that
you need.  Follow that pattern already there to copy the patch into the 
container, apply it, and then remove the temporary files after you are done.

```
# ---------------------------------------------
# Install patches to run WebLogic on Kubernetes
# ---------------------------------------------

# Start with the standard WebLogic 12.2.1.3.0 Docker image
FROM store/oracle/weblogic:12.2.1.3

MAINTAINER Mark Nelson <mark.x.nelson@oracle.com>

# We need patch 29135930 to run WebLogic on Kubernetes
# We will also also install the latest PSU which is 28298734
# That prereqs a newer version of OPatch, which is provided by 28186730

ENV PATCH_PKG0="p28186730_139400_Generic.zip"
ENV PATCH_PKG2="p28298734_122130_Generic.zip"
ENV PATCH_PKG3="p29135930_12213181016_Generic.zip"

# Copy the patches into the container
COPY $PATCH_PKG0 /u01/
COPY $PATCH_PKG2 /u01/
COPY $PATCH_PKG3 /u01/


# Install the psmisc package which is a prereq for 28186730
USER root
RUN yum -y install psmisc

# Install the three patches we need - do it all in one command to
# minimize the number of layers and the size of the resulting image.
# Also run opatch cleanup and remove temporary files.
USER oracle
RUN cd /u01 && \
    $JAVA_HOME/bin/jar xf /u01/$PATCH_PKG0 && \
    $JAVA_HOME/bin/java -jar /u01/6880880/opatch_generic.jar \
        -silent oracle_home=/u01/oracle -ignoreSysPrereqs && \
    echo "opatch updated" && \
    sleep 5 && \
    cd /u01 && \
    $JAVA_HOME/bin/jar xf /u01/$PATCH_PKG2 && \
    cd /u01/28298734 && \
    $ORACLE_HOME/OPatch/opatch apply -silent && \
    cd /u01 && \
    $JAVA_HOME/bin/jar xf /u01/$PATCH_PKG3 && \
    cd /u01/29135930 && \
    $ORACLE_HOME/OPatch/opatch apply -silent && \
    $ORACLE_HOME/OPatch/opatch util cleanup -silent && \
    rm /u01/$PATCH_PKG0 && \
    rm /u01/$PATCH_PKG2 && \
    rm /u01/$PATCH_PKG3 && \
    rm -rf /u01/6880880 && \
    rm -rf /u01/28298734 && \
    rm -rf /u01/29135930 && \
    rm -rf /u01/oracle/cfgtoollogs/opatch/*

WORKDIR ${ORACLE_HOME}

CMD ["/u01/oracle/createAndStartEmptyDomain.sh"]
```

This `Dockerfile` assumes the patch archives are available in the same
directory. You would need to download the patches from [My Oracle 
Support](https://support.oracle.com) and then you can build the image 
with this command:

```
docker build -t my-weblogic-base-image:12.2.1.3.0 .
```

Take special note of that last line: 

```
rm -rf /u01/oracle/cfgtoollogs/opatch/*
```

If you include that line, it will remove the backups created
by OPatch when applying the patches.  In the case of a PSU, these 
backups can consume a significant amount of space.  Removing them does two 
things - first, it will reduce the size of the resulting image, which is a 
good thing; but second, it will make the resulting image unpatchable - you 
won't be able to apply
any more patches to it, or remove any that are already there.  This is not 
a problem if you are going to build new images every time you want to 
change you selection of patches.  If you want to maintain the ability 
to patch that image, just take out that line.

### Creating the image with the domain in it

I am going to use the [WebLogic Deploy Tooling](https://github.com/oracle/weblogic-deploy-tooling)
to define my domain.  If you are not familiar with this tool, you might 
want to check it out!  It lets you define your domain declaratively instead
of writing custom WLST scripts.  For just one domain, maybe not such a big 
deal, but if you need to create a lot of domains it is pretty useful.  It 
also lets you paramaterize them, and it can introspect existing domains to 
create a model and associated artifacts.  You can also use it to "move" 
domains from place to place, say from an on-premises install to Kubernetes,
and you can change the version of WebLogic on the way without needing to worry
about differences in WLST from version to version - it takes care of all 
that for you.  Of course, we don't need all those features for what we need
to do here, but it is good to know they are there for when you might need
them!

I created a GitHub repository with my domain model 
[here](https://github.com/markxnelson/simple-sample-domain). You can just
clone this repository and then run the commands below to download
the WebLogic Deploy Tooling and then build the domain in a new Docker
image that we will tag `my-domain1-image:1.0`:


```
git clone https://github.com/markxnelson/simple-sample-domain
cd simple-sample-domain
curl -Lo weblogic-deploy.zip https://github.com/oracle/weblogic-deploy-tooling/releases/download/weblogic-deploy-tooling-0.14/weblogic-deploy.zip
./build-archive.sh
./quickBuild.sh
```

I won't go into all the nitty gritty details of how this works, that's a 
subject for another post, but take a look at the `simple-toplogy.yaml` file
to get a feel for what is happening:

{{< highlight bash "linenos=table" >}}
domainInfo:
    AdminUserName: '@@FILE:/u01/oracle/properties/adminuser.properties@@'
    AdminPassword: '@@FILE:/u01/oracle/properties/adminpass.properties@@'
topology:
    Name: '@@PROP:DOMAIN_NAME@@'
    AdminServerName: '@@PROP:ADMIN_NAME@@'
    ProductionModeEnabled: '@@PROP:PRODUCTION_MODE_ENABLED@@'
    Log:
        FileName: '@@PROP:DOMAIN_NAME@@.log'
    Cluster:
        '@@PROP:CLUSTER_NAME@@':
            DynamicServers:
                ServerTemplate: '@@PROP:CLUSTER_NAME@@-template'
                CalculatedListenPorts: false
                ServerNamePrefix: '@@PROP:MANAGED_SERVER_NAME_BASE@@'
                DynamicClusterSize: '@@PROP:CONFIGURED_MANAGED_SERVER_COUNT@@'
                MaxDynamicClusterSize: '@@PROP:CONFIGURED_MANAGED_SERVER_COUNT@@'
    Server:
        '@@PROP:ADMIN_NAME@@':
            ListenPort: '@@PROP:ADMIN_PORT@@'
            NetworkAccessPoint:
                T3Channel:
                    ListenPort: '@@PROP:T3_CHANNEL_PORT@@'
                    PublicAddress: '@@PROP:T3_PUBLIC_ADDRESS@@'
                    PublicPort: '@@PROP:T3_CHANNEL_PORT@@'
    ServerTemplate:
        '@@PROP:CLUSTER_NAME@@-template':
            ListenPort: '@@PROP:MANAGED_SERVER_PORT@@'
            Cluster: '@@PROP:CLUSTER_NAME@@'
appDeployments:
    Application:
        # Quote needed because of hyphen in string
        'test-webapp':
            SourcePath: 'wlsdeploy/applications/test-webapp.war'
            Target: '@@PROP:CLUSTER_NAME@@'
            ModuleType: war
            StagingMode: nostage
            PlanStagingMode: nostage
{{< / highlight >}}

As you can see it is all parameterized. Most of those properties are defined
in `properties/docker-build/domain.properties`: 

{{< highlight bash "linenos=table" >}}
# These variables are used for substitution in the WDT model file.
# Any port that will be exposed through Docker is put in this file.
# The sample Dockerfile will get the ports from this file and not the WDT model.
DOMAIN_NAME=domain1
ADMIN_PORT=7001
ADMIN_NAME=admin-server
ADMIN_HOST=domain1-admin-server
MANAGED_SERVER_PORT=8001
MANAGED_SERVER_NAME_BASE=managed-server-
CONFIGURED_MANAGED_SERVER_COUNT=2
CLUSTER_NAME=cluster-1
DEBUG_PORT=8453
DEBUG_FLAG=false
PRODUCTION_MODE_ENABLED=true
JAVA_OPTIONS=-Dweblogic.StdoutDebugEnabled=false
T3_CHANNEL_PORT=30012
T3_PUBLIC_ADDRESS=openshift
CLUSTER_ADMIN=cluster-1,admin-server
{{< / highlight >}}

On lines 10-17 we are defining a cluster named `cluster-1` with two dynamic 
servers in it. On 18-25 we are defining the admin server.  And on 30-38 we 
are defining an application that we want deployed.  This is a simple web 
application that prints out the IP address of the managed server it is running
on.  Here is the main page of that application:

{{< highlight java "linenos=table" >}}
<%@ page import="java.net.UnknownHostException" %>
<%@ page import="java.net.InetAddress" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <c:url value="/res/styles.css" var="stylesURL"/>
    <link rel="stylesheet" href="${stylesURL}" type="text/css">
    <title>Test WebApp</title>
  </head>
  <body>
    <%
      String hostname, serverAddress;
      hostname = "error";
      serverAddress = "error";
      try {
        InetAddress inetAddress;
        inetAddress = InetAddress.getLocalHost();
        hostname = inetAddress.getHostName();
        serverAddress = inetAddress.toString();
      } catch (UnknownHostException e) {

        e.printStackTrace();
      }
    %>

    <li>InetAddress: <%=serverAddress %>
    <li>InetAddress.hostname: <%=hostname %>

  </body>
</html>
{{< / highlight >}}

The source code for the web application is in the `test-webapp` directory.
We will use this application later to verify that scaling and load balancing
is working as we expect.

So now we have a Docker image with our custom domain in it, and the WebLogic
Server binaries and the patches we need.  So we are ready to deploy it!

## Create the WebLogic domain 


## Verify access to the WebLogic administration console and WLST 


## Deploy a test application into the cluster 


## Set up a route to expose the application publicly


## Test scaling and load balancing


## Install the WebLogic Exporter to get metrics into Prometheus



