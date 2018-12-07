---
title: "Running WebLogic on OpenShift (on OCI)"
date: 2018-12-06T14:07:54-05:00
draft: true
toc: true
taxonomies:
  tag: oci, openshift
---
In this post, I want to share my experiences with running WebLogic on 
OpenShift.  This is not strictly a "how to" because I did take a few 
small short cuts, but I think that it is still (hopefully) pretty helpful.

I am going to set up OpenShift Enterprise 3.11 on a single compute instance 
on Oracle Cloud Infrastructure (OCI), so I am using the "all in one" 
configuration.  This means that the master and the worker node is on the 
same machine.  This is not a production configuration, but it is fine for
testing and development purposes.

## Overview

Here's an overview of the process I will walk through in this post:

* Prepare the OCI tenancy to run OpenShift - set up compartments, groups,
  policies, instance principals, and so on
* Create a RHEL 7.4 custom image in OCI (since OCI does not provide any
  RHEL images)
* Create a compute instance using that custom image
* Install and configure OpenShift in that instance
* Prepare my WebLogic Docker images
* Install the WebLogic Operator for Kubernetes
* Create some WebLogic domains
* Explore the various other integrations like load balancing WebLogic 
  clusters, exporting metric to Prometheus, and scaling clusters

To follow along, you will need a RedHat subscription which includes 
RHEL and OpenShift Enterprise, and an OCI tenancy.

## Preparing your OCI tenancy to run OpenShift

Before we start, we need to set up our tenancy to run OpenShift. Let's 
start by creating a *compartment*, I called mine `OpenShift`.  You can 
create a compartment in [the OCI Console](https://console.us-phoenix-1.oraclecloud.com)
in the "Identity" menu, then "Compartments".

{{< figure src="/images/openshift001.png" >}}

Notice the OCID on this page - you will need this value later, you may 
want to copy it and save it somewhere. 

**Groups**

Now we should set up some *groups* so we can control access to our 
compartment with *policies*.  In the "Indentity" menu, go to "Groups" and 
create two groups: 

* `OpenShiftAdministrators` - add yourself to this group, and
* `OpenShiftUsers`

{{< figure src="/images/openshift002.png" >}}

**Policies**

In the "Identity" menu, go to "Policies" and make sure you choose your root 
compartment, not the compartment you jsut created!  Create a new policy, I
called mine `OpenShiftAdministratorsPolicy` and add these five *policy statements*:

* `Allow group OpenShiftAdministrators to manage all-resources in compartment OpenShift`
* `Allow group OpenShiftAdministrators to inspect all-resources in tenancy`
* `Allow group OpenShiftAdministrators to read instances in tenancy`
* `Allow group OpenShiftAdministrators to read all-resources in tenancy`
* `Allow group OpenShiftAdministrators to manage repos in tenancy`

In the "policy versioning" section choose the option to "keep policy current".

{{< figure src="/images/openshift003.png" >}}

Create another policy called `OpenShiftUsersPolicy` for regular users.
Keep the policy current and add the following policy statements to this policy:

* `Allow group OpenShiftUsers to use all-resources in compartment OpenShift`
* `Allow group OpenShiftUsers to inspect all-resources in tenancy`
* `Allow group OpenShiftUsers to read instances in tenancy`
* `Allow group OpenShiftUsers to read audit-events in tenancy`
* `Allow group OpenShiftUsers to read all-resources in tenancy`

**Dynamic Group (Instance Principals)**

Now we need to create a *dynamic group* to enable "instance principals" for
this compartment.  "Instance principal" means that a compute instance can 
act as a "principal" and we can control which API calls it can make by 
writing policies for this dynamic group - just like we would normally write
policies for IAM users or groups - so in essence this lets us treat a compute
instance as an actor in its own right.  In the dynamic group itself, we 
set a rule to determine which instances in the compartment it will apply to.
In this case we will use this to control which instances will be able to 
use our RHEL subscription. 

In the "Identity" menu, under "Dynamic groups" create a new dynamic group 
called `OpenShiftSubscriptionUsers` and add this rule:

* `ALL {instance.compartment.id = 'xxx'}`

Replace the `xxx` with the OCID of your compartment - the one that you
saved in an earlier step.  This rule will match all compute instances in 
this compartment.

**Subscription Policy**

In the "Identity" menu, under "Policies" create a new policy called 
called `OpenShiftSubscriptionPolicy`, keep it current and add this policy 
statement:

* `Allow dynamic-group OpenShiftSubscriptionUsers to manage all-resources in compartment OpenShift`

The reason we need this is because we are going to run an Oracle Linux instance
with a ipxe boot server in it, and then it will create another instance, and
boot that instance and install RHEL into it by booting from that server running
in the first instance.  So that instance needs to be able to act as a principal
to call the necessary OCI APIs to create the second instance. (I hope all of this
instance principal stuff makes sense!)

**Create the Virtual Cloud Network**

You need to create your Virtual Cloud Network before we move on to the next 
steps.  You do this in the "Networks" menu, then "Virtual Cloud Networks". 
Make sure you are in the right compartment, then click on the big blue
"Create Virtual Cloud Network" button.  Just give it a name (I used `OpenShiftVCN`),
and check the "Create virtual cloud network plus related resources" option, 
leave everything else at the defaults and click on "Create Virtual Cloud Network".
When it is done, click on the "Close" button.

{{< figure src="/images/openshift009.png" >}}

You will see your new VCN, click on that to view it, and then find the subnet 
that matches the Availability Domain that you want to use, I used `AD-3`.

{{< figure src="/images/openshift010.png" >}}

Click on the security list for that subnet to view it, then click on the "Edit 
All Rules" button.  Find the rule for ICMP, and change it from `3, 4` to `All`:

{{< figure src="/images/openshift011.png" >}}

Click on the "+ Another Ingress Rule" button to create another rule, and set it 
up for source CIDR `0.0.0.0/0` and destination port `80, 8443`:

{{< figure src="/images/openshift012.png" >}}

Now click on the "Save security list rules" button to save them.  Now your 
security list should look like this:

{{< figure src="/images/openshift013.png" >}}

## Creating the RHEL 7.4 custom image

OCI does not provide a RHEL image, so we are going to need to create one. 
We need a RHEL 7.4 custom image to create
our compute instance from.  I used Terraform to create this image.  Before
starting, you will need to sign up for a RedHat subscription (if you don't 
already have one).

**Get the RHEL 7.4 ISO**

First create a bucket if you
don't already have one.  This is done in the "Object Storage" menu under "Object 
Storage".  Make sure you choose your compartment - don't create it in the 
root compartment.  I called my bucket `ISO-Images`.

{{< figure src="/images/openshift004.png" >}}

You will now need to download the RHEL 7.4 ISO from the RedHat subscription
download site and then upload it into your object store using the OCI CLI (because
it is too large to upload through the web UI).  Here is the command to 
upload to image into this bucket:

```
oci os object put --bucket-name ISO-Images --file rhel-server-7.4-x86_64-dvd.iso
```

If you don't have the OCI CLI installed, you can find out how to [install it
and set up the config file here](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliinstall.htm).

Getting those config files right can be a challenge, so here is what mine 
looks like, with the values changed, to give you a sample to work from:

```
$ cat ~/.oci/config
[DEFAULT]
user=ocid1.user.oc1..aaaaaaaawwn54sdf789sdfjkasdhhjksfd890adsfjkajasfd890asdfzmra
fingerprint=3d:5c:f9:7a:34:fd:e7:ab:4a:ba:34:52:d4:77:e5:e9
key_file=/Users/marnelso/.oci/oci_api_key.pem
pass_phrase=WebLogicCafe
tenancy=ocid1.tenancy.oc1..aaaaaaaafsdhjkfds789fdshjkfds789fdskjdsf789dfshkfds79sx63imq
region=us-phoenix-1
```

**Create Pre-authenticated Request**

In order for our terraforming to access the ISO image in the next step
without the need for any complex authentication work, we can create a 
*Pre-authenticated request* which will permit access to the ISO image 
without authentication for a short period of time - provided you happen
to know the magic URL! 

To create it, click on the "three dots" next to your ISO image and choose 
"Create Pre-Authenticated Request" from the menu:

{{< figure src="/images/openshift005.png" >}}

You need to give it a name, and you can keep the read-only option and
set a reasonable expiration time - a couple of hours will be plenty of time:

{{< figure src="/images/openshift006.png" >}}

It will give you a URL to access the ISO file - make sure you keep that URL, 
you will need it later:

{{< figure src="/images/openshift007.png" >}}

**Creating the image**

As I mentioned earlier, we will use Terraform to drive the image creation 
process.  It is going to work like this:

{{< figure src="/images/openshift008.png" >}}

We will start `Instance1` which will boot from the standard Oracle Linux
image provided by OCI.  It will start up a IPXE boot server and mount the
RHEL ISO using the pre-authenticated request URL.  Then we will start a 
second instance (`Instance2`) with no OS and we will boot it from the IPXE
server running in `Instance1` and drive the RHEL installation in this instance.
When we are done we will create a *custom image* from `Instance2`.  

You can download my sample code from GitHub by cloning 
[this repository](https://github.com/markxnelson/weblogic-on-openshift) as 
follows:

```
git clone https://github.com/markxnelson/weblogic-on-openshift
```

In this repository, you will see a `terraform` directory.  That is what we
will use in this step.  (We will use the others later)

To get ready to run this you will first of all need terraform installed
if you don't already have it.  You can get that from [here](https://www.terraform.io/downloads.html).
You will also need the OCI Terraform provider, which you can download
from [here](https://github.com/terraform-providers/terraform-provider-oci/releases).

To set up the OCI provider, follow the instructions [here](https://www.terraform.io/docs/providers/oci/guides/version-3-upgrade.html).  You will see this talks about the provider block:

```
provider "oci" {
  version          = ">= 3.0.0"
  region           = "${var.region}"
  tenancy_ocid     = "${var.tenancy_ocid}"
  user_ocid        = "${var.user_ocid}"
  fingerprint      = "${var.fingerprint}"
  private_key_path = "${var.private_key_path}"
}
```

This is located in the file `terraform/variable.tf` in my repository. It is
set up to take the values from the environment.  So you can just edit the
`env-vars` file and put the values in there:

{{< highlight bash "linenos=table,hl_lines=2-5 8 11 16" >}}
### Authentication details
export TF_VAR_tenancy_ocid="xxx"
export TF_VAR_user_ocid="xxx"
export TF_VAR_fingerprint="xxx"
export TF_VAR_private_key_path="xxx"

### Optional API Private key passphrase (if set)
export TF_VAR_private_key_password="xxx"

### Region
export TF_VAR_region="us-phoenix-1"

### Public keys used on the instance
## NOTE: These are not your api keys. More info on the right keys see
## https://docs.us-phoenix-1.oraclecloud.com/Content/Compute/Tasks/managingkeypairs.htm
export TF_VAR_ssh_public_key=$(cat xxx)
{{< / highlight >}}

Note the highlighted lines, these are the ones where you need to make 
updates.  The `TF_VAR_private_key_path` on line 5 is for your OCI API key
and the `TF_VAR_private_key_password` on line 8 is the password for that
same key.

The `TF_VAR_ssh_public_key` on line 16 is for the SSH key that you would 
use to SSH into an instance.  This is different to your API key! 

You will need to update `TF_VAR_region` to match the region where you have
capacity to run these instances.

Once you have all your updates in place, you can source this file into your
shell to make these variables available to terraform:

```
. ./env-vars
```

Next, let's look at our `configuration.tf`, again the lines you need to update
are highlighted:

{{< highlight bash "linenos=table,hl_lines=7 17-18 31-34" >}}
# Set your information here:

# iso_url - specify the URL for the RHEL ISO.  This can be any publically accessible URL.
# The URL must contain the name of the ISO image with an '.iso' extension.

variable "iso_url" {
	default = "<your pre-authenticated request url>"
}

# RHEL account variables:
# user_name - The user name of the account that holds the subscription
# password - password for said user name

variable "rhel_account" {
	type = "map"
	default = {
		user_name = "<your redhat account>"
		password = "<your password>"
	}
}

# Build environment variables:
# compartment - your compartment name
# ad - which Availability Domain to use. Format in either AD-x or ad-x where 'x' is the AD number
# vcn - the display name of the vcn to use
# subnet - display name of the subnet to use

variable "build_env" {
	type = "map"
	default = {
		compartment = "<your compartment>"
		ad = "<your AD>"
		vcn = "<your VCN>"
		subnet = "<your subnet>"
	}
}
{{< / highlight >}}

On line 7, you need to put in that pre-authenticated request URL we saved in an 
earlier step.

On lines 17 and 18, you put in your RedHat credentials.  The `user_name` in this 
case is probably your email address (not the short userid).

On lines 31 to 34, you will put in your OCI details. `compartment` is the name 
of your compartment (not the OCID), `ad` will be a string like `AD-3`, `vcn` will
be the name of your VCN (not the OCID) and `subnet` will be the name (not the OCID)
of the subnet you want to use (it has to be a public one), it will probably look
something like `Public Subnet VPGL:PHX-AD-3`.

Next, you need to edit the `variable.tf` file to set the correct shape to use. 
This will depend on what shapes you have available.  You can use either a VM or
BM shape.  I used a `VM.Standard2.8` which is actually much, much bigger than 
we need, but that is just what I had free.  Pretty much any shape should work 
for you.  Set the shape on line 24:

{{< highlight bash "linenos=table,hl_lines=6,linenostart=19" >}}
variable "ipxe_instance" {
	type = "map"
	default = {
		name = "ipxe-rhel74"
		hostname = "ipxe-rhel74"
		shape = "VM.Standard2.8"
	}
}
{{< / highlight >}}

Now we are ready to run it!  First, make sure terraform is ready by running this
command:

```
terraform init
```

This will check all your conifguration is correct and the OCI provider is installed
correctly.  If you get any failures here, you will have to go and fix them before
you can continue.

Once that has completed successfully, you want to run the "plan" command:

```
terraform plan
```

This will look at your OCI environment and work out what needs to actually be done.
It will not take any action, but it will print out details of what it *would* do
if you ran it.  You should read through this output carefully to make sure you
know what it is going to do before moving on.  Check again it is pointing to the
right compartment! 

When you are ready to continue, run the "apply" command:

```
terraform apply
```

Now be aware that this is going to take some time to complete, maybe 30 to 60 minutes. 
The following things are going to happen:

* The template generates a script that embeds all the configuration files needed 
  to build the iPXE server, extract the ISO boot the instance used to load RHEL, 
* It starts up an instance and runs this script,
* It starts a new instace, causes RHEL to load, builds the image, and destroys 
  the build instance, and then
* Destroys the iPXE server.  

You are left with a custom image named "RHEL_74" in your environment.  You can 
watch all of this happening in the OCI console, and if you are curious you can 
also SSH into the instances and run `journalctl -f` to watch the logs.

After your image is created, you should execute this command to 
clean up (there will not be a resource to actually kill, so force is required):

```
terraform destroy -force
```

You should be able to see your new custom image in the OCI Console under "Compute"
and then "Custom Images":

{{< figure src="/images/openshift014.png" >}}

Now that we have our custom RHEL 7.4 image in OCI, we can use that to create 
compute instances running RHEL! So now we are ready to go ahead and set up
OpenShift.

## Creating a compute instance

Now we can create our compute instance where we will install OpenShift.  To do 
this, we go to the "Compute" menu, then "Instances" and click on the "Create
Instance" button.  This will bring up the following page:

{{< figure src="/images/openshift015.png" >}}

You will need to do the following: 

* Give the image a name, I used `openshift`,
* Click on the "Change Image Source" button and choose "Custom Images" and 
  select the "RHEL 7.4" image we just created,
* Choose an instance type (VM or BM), 
* Choose the instance shape that you want to use, you probably want a fairly
  large one so you can run a lot of domains and clusters.  I used a `VM.Standard2.16`
  while has 16 OCPUs and 240GB of RAM.  You can see the definitions of the 
  various shapes [here](https://cloud.oracle.com/compute/pricing).
* Choose a custom boot volume size, I used 200GB.  If you are just doing testing
  and development, it is easiest to just put all your data on this volume.  In 
  production you might want to use NFS or something else instead, so you might 
  not need quite as large a boot volume.
* Add a SSH public key - this is the key that you will use to SSH into your 
  new instance.
* The networking section defaults should be fine.

Once you have all this set up, click on the "Create" button to create the 
instance. 

{{< figure src="/images/openshift016.png" >}}

If you open your instance in the OCI console you can get the public IP address
and you should now be able to SSH into your instance and become the superuser:

```
[Fri Dec 07 11:17:04] marnelso@MARNELSO-mac weblogic-on-openshift (master) 
$ ssh -i ~/.ssh/openshift opc@129.146.53.62
The authenticity of host '129.146.53.62 (129.146.53.62)' can't be established.
ECDSA key fingerprint is SHA256:C/YRPeEpsdf789sdf7d89sJYpIt43ZhaKHheaLMG7vM.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '129.146.53.62' (ECDSA) to the list of known hosts.
[opc@openshift ~]$ sudo su -
[root@openshift ~]# 
```

Now we are ready to set up the prerequisites.

## Setting up the prerequisties for OpenShift

I followed through the very detailed installation instructions on the [OpenShift
documentation site](https://docs.openshift.com/container-platform/3.11/install/index.html)
and I will just include the highlights here.  You will want to refer to that
site in parallel as you follow along.

On your machine, run these commands as root: 

```
subscription-manager refresh
subscription-manager list --available --matches '*OpenShift*'
```

In the output, find the pool ID and use it in the next command:

```
subscription-manager attach --pool=8a85789cd87978f9fe673b578d7c1de0
subscription-manager repos --disable="*"
subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"
```

Now install the prerequisites, the restart the machine:

```
yum install wget git net-tools bind-utils yum-utils iptables-services \
    bridge-utils bash-completion kexec-tools sos psacct
yum update
reboot
```

## Install OpenShift binaries

Let's chose the RPM-based install and so we run these commands:

```
yum install openshift-ansible
yum install docker-1.13.1
systemctl enable docker
systemctl start docker
```

Then we verify docker is using `overlay2` by checking `/etc/sysconfig/docker-storage`.
If it is not using overlay, then you probably want to go and configure that.

We can skip over some of the next steps since we are setting up for development
and testing here:

* do not enable image signature support,
* do not worry about log file size limit, and
* do not worry about blocking local volume use

## Glusterfs setup

Next we set up `glusterfs` using these commands: 

```
yum install glusterfs-fuse
subscription-manager repos --enable=rh-gluster-3-client-for-rhel-7-server-rpms
yum update glusterfs-fuse
```

## Prepare to run Ansible

To prepare for the setup, we need to create the file `/etc/ansible/hosts` with 
the following content (this is also in the repository that you cloned as 
`openshift/ansible.hosts`):

{{< highlight bash "linenos=table,hl_lines=17-18 21 24-28 35 39 43" >}}
# Create an OSEv3 group that contains the masters, nodes, and etcd groups
[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root

# If ansible_ssh_user is not root, ansible_become must be set to true
#ansible_become=true

openshift_deployment_type=openshift-enterprise
#oreg_url=example.com/openshift3/ose-${component}:${version}
oreg_auth_user=<your_userid>
oreg_auth_password=<your_password>
openshift_examples_modify_imagestreams=true

openshift_master_default_subdomain=<your subdomain>
openshift_docker_additional_registries=phx.ocir.io

openshift_hostname=<your hostname>
openshift_ip=<your private ip address>
openshift_public_ip=<your public ip address>
openshift_master_cluster_hostname=<your fqdn>
openshift_master_cluster_ip=<your private ip address>

# uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
#openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

# host group for masters
[masters]
<your fqdn> openshift_ip=<your private ip> openshift_hostname=<your hostname> openshift_master_cluster_hostname=<your fqdn> openshift_master_cluster_ip=<your private ip> openshift_public_ip=<your public ip> openshift_node_group_name='node-config-all-in-one'

# host group for etcd
[etcd]
<your fqdn> openshift_ip=<your private ip>

# host group for nodes, includes region info
[nodes]
<your fqdn> openshift_node_group_name='node-config-all-in-one'
{{< / highlight >}}

There are quite a few things to update in this file:

* On lines 17 and 18, replace `<your_userid>` and `<your_password>` with your RedHat
  credentials - for this one the short userid is needed, not your email address,
* On line 21, replace `<your subdomain>` with the subdomain for your subnet, this 
  will look like `sub12071436402.openshiftvcn.oraclevcn.com`, and you can get it 
  from the Subnet in the Virtual Cloud Network page (that we looked at earlier),
* On line 24, replace `<your hostname>` with the hostname of your machine, if you
  followed the example above, this will be `openshift`,
* On line 25, replace `<your private ip address>` with the private IP address of 
  the machine, you can get this from the instance page in the OCI console (see image
  below),
* On line 26, replace `<your public ip address>` with the public IP address of 
  the machine, which you can get from the same page, 
* On line 27, replace `<your fqdn>` with the internal FQDN (fully qualified domain
  name) from the same page,
* On line 28, replace `<your private ip address>` with the private IP address (the
  same as the one on line 25), and
* On lines 35, 39 and 43 there are several of these same fields again, replace them all with 
  the same values as you did earlier.

{{< figure src="/images/openshift017.png" >}}

Let's walk through this file and make sure we understand what it is defining, because
this is a pretty important file in the overall process.

On lines 3 to 5 we are defining three types of hosts (roles) called `masters`, `nodes` and 
`etcd`.  The `masters` will be used to run the Kubernetes/OpenShift masters, i.e.
the API servers and so on.  The `nodes` are the worker nodes, i.e. `kubelet`s where
our pods and services will run.  And the `etcd` ones are where etcd will run!

On line 10 we set the user that ansible will use to SSH to the other nodes.  Note
that since we are doing the "all in one" install, the same machine will take all three 
of the roles (`master`, `node`, and `etcd`) and ansible will only SSH to this one
machine.

On line 15 we set the version that we are dpeloying: `openshift-enterprise`.

Lines 17 and 18 set the credentials to use to access the RedHat Docker registries.
Line 22 lists additional Docker registries that we want OpenShift to search for
images in. 

Lines 21 and 24 to 28 set the names and IP addresses of the major components, these
are used between the various components when they need to talk to each other, so it
is very important that these are all correct and they all resolve correctly.

On line 34 we define the `master` role.  Here it is again, reformatted to make it 
a little easier to read:

```
<your fqdn> 
openshift_ip=<your private ip> 
openshift_hostname=<your hostname> 
openshift_master_cluster_hostname=<your fqdn> 
openshift_master_cluster_ip=<your private ip> 
openshift_public_ip=<your public ip> 
openshift_node_group_name='node-config-all-in-one'
```

Here we are passing in the network details that are required by various components
when they are configured, e.g. the catalog, so they can contact other services.
Also we identify at the end that we are using the "all-in-one" method of 
installation.

Similarly, line 39 sets up the `etcd` role, and line 43 sets up the `node` role.

## Set up DNS 

We need to add a line to `/etc/hosts` with the hostname and FQDN so that ansible 
and also the various OpenShift components can connect to each other using the 
hostnames, here is mine for example:

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.0.2.13   openshift openshift.sub11201828382.certificationvc.oraclevcn.com
```

It is important that you put the hostname and FQDN on the private IP address, not
the loopback address, or various OpenShift components will fail. 

## Additional (optional) components

There are several option components/configurations that we don't need in a development/test
environment.  So we will skip over these:

* glusterfs,
* integrated registry,
* global proxy,
* firewall - take the defaults (iptables),
* session options - use the defaults, and using a single master anyway, so not needed,
* custom certificates,
* cluster logging - take the default (none),
* service broker, 
* openshift ansible broker,
* template service broker,
* web console customization,
* cluster console customization,
* operator lifecycle manager,

For most of these the defaults are fine for what we need.

## Create SSH keys

We need to create a SSH key pair and add it to the authorized hosts file (as root)
so that ansible will be able to SSH into this machine, make sure you also SSH in
to this machine once (from itself, as root) to accept the certificate so that 
ansible will be able to connect when needed:

```
ssh-keygen # (hit enter for all)
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
ssh openshift.sub11201828382.certificationvc.oraclevcn.com # (to verify)
```

## Installing OpenShift (at last!!!)

Finally we come to the actual installation of OpenShift! (yay!) which is done with
ansible.  First we need to run these commands to check all the prerequisites are met:

```
cd /usr/share/ansible/openshift-ansible
ansible-playbook playbooks/prerequisites.yml
```

If you get any errors you will need to go back and fix them before continuing, but
if you have been following along the examples, this should complete with no 
errors.

Next we can deploy the OpenShift cluster - this is where the bulk of the work is 
done:

```
ansible-playbook playbooks/deploy_cluster.yml
```

Lots of things could go wrong during this step, and if they do, you will probably
find that the error messages that ansible prints out are actually fairly helpful 
(at least I did), and they also tell you how you can rerun the failing component
without going through the whole thing again.  

There is one interesting comment on the OpenShift documentation site that you 
should be aware of though: 

> On failure of the Ansible installer, you must start from a clean operating system installation. If you are using virtual machines, start from a fresh image. If you are using bare metal machines, run the following on all hosts:

```
# yum -y remove openshift openshift-* etcd docker docker-common

# rm -rf /etc/origin /var/lib/openshift /etc/etcd \
     /var/lib/etcd /etc/sysconfig/atomic-openshift* /etc/sysconfig/docker* \
     /root/.kube/config /etc/ansible/facts.d /usr/share/openshift
```

I did have to use these a couple of times until I got everything right!

Once the `deploy_cluster` complets successfully, we can  move on to verification.
Here is the end of my log, just for reference:

```
PLAY RECAP ***********************************************************************************************************************
localhost                  : ok=11   changed=0    unreachable=0    failed=0
openshift.sub11201828382.certificationvc.oraclevcn.com : ok=672  changed=287  unreachable=0    failed=0


INSTALLER STATUS *****************************************************************************************************************
Initialization               : Complete (0:00:10)
Health Check                 : Complete (0:00:22)
Node Bootstrap Preparation   : Complete (0:03:15)
etcd Install                 : Complete (0:01:39)
Master Install               : Complete (0:04:58)
Master Additional Install    : Complete (0:04:11)
Node Join                    : Complete (0:00:08)
Hosted Install               : Complete (0:00:37)
Cluster Monitoring Operator  : Complete (0:00:41)
Web Console Install          : Complete (0:00:19)
Console Install              : Complete (0:00:13)
metrics-server Install       : Complete (0:00:01)
Service Catalog Install      : Complete (0:02:14)
Monday 26 November 2018  16:37:22 -0800 (0:00:00.029)       0:19:06.579 *******
===============================================================================
cockpit : Install cockpit-ws -------------------------------------------------------------------------------------------- 216.56s
openshift_control_plane : Wait for all control plane pods to become ready ------------------------------------------------ 71.58s
etcd : Install etcd ------------------------------------------------------------------------------------------------------ 62.14s
openshift_node : Install node, clients, and conntrack packages ----------------------------------------------------------- 48.51s
openshift_node : install needed rpm(s) ----------------------------------------------------------------------------------- 43.98s
openshift_control_plane : Wait for control plane pods to appear ---------------------------------------------------------- 43.13s
template_service_broker : Verify that TSB is running --------------------------------------------------------------------- 34.79s
openshift_service_catalog : Verify that the catalog api server is running ------------------------------------------------ 34.08s
openshift_cluster_monitoring_operator : Wait for the ServiceMonitor CRD to be created ------------------------------------ 32.85s
openshift_ca : Install the base package for admin tooling ---------------------------------------------------------------- 31.16s
Run health checks (install) - EL ----------------------------------------------------------------------------------------- 21.47s
openshift_excluder : Install docker excluder - yum ----------------------------------------------------------------------- 13.95s
openshift_cli : Install clients ------------------------------------------------------------------------------------------ 13.02s
openshift_excluder : Install openshift excluder - yum -------------------------------------------------------------------- 12.01s
openshift_service_catalog : oc_process ----------------------------------------------------------------------------------- 11.10s
openshift_web_console : Verify that the console is running --------------------------------------------------------------- 11.02s
openshift_node_group : Wait for the sync daemonset to become ready and available ----------------------------------------- 10.95s
openshift_control_plane : Wait for APIs to become available --------------------------------------------------------------- 9.32s
openshift_hosted : Create OpenShift router -------------------------------------------------------------------------------- 7.77s
openshift_node : Install iSCSI storage plugin dependencies ---------------------------------------------------------------- 7.62s
```

## Verifying the OpenShift install

We can follow the verification instructions from the OpenShift documentation and
run the following commands (I am showing the output too for reference):

```
# oc get nodes
NAME        STATUS    ROLES                  AGE       VERSION
openshift   Ready     compute,infra,master   12m       v1.11.0+d4cacc0
```

You may want to add an entry to your host file similar to this (put your machine's
public IP address in there):

```
129.146.64.69 openshift.sub11201828382.certificationvc.oraclevcn.com openshift
```

Then you should be able to hit the OpenShift [console](http://openshift:8443/console)
in your browser.  It looks like this:

{{< figure src="/images/openshift018.png" >}}

You wont be able to log in yet, since we have not set up an identity provider and 
created some users, so let's do that now!

## Setting up an identity provider

The simplest useful identity provider is `htpasswd`, so let's set that up.  Again, 
it is probably not what you would choose for a production environment, but it is
fine for development and testing.  Run this command to install the software:

```
yum install httpd-tools
```

Then run this command to create both the password database and our first user:

```
htpasswd -c /etc/origin/master/users.htpasswd mark
(enter password)
```

Note that the `-c` is to create the file (first time), when creating additional 
users later, the `-c` is *not* required.

Also, note that the file *must* be in a directory that is mounted into the relevant
pods on master(s), so please put it in `/etc/origin/master`.

Next, we need to update the master configuration file, which is located at 
`/etc/origin/master/master-config.yaml` to tell it about our identity provider.
We need to update the `oauthConfig` section to remove the provider that it already
there (`DenyAll`) and add the `htpasswd` provider, copy the configuration shown 
in the example below, on lines 6 to 13:

{{< highlight bash "linenos=table,hl_lines=6-13" >}}
oauthConfig:
  assetPublicURL: https://openshift.sub11201828382.certificationvc.oraclevcn.com:8443/console/
  grantConfig:
    method: auto
  identityProviders:
  - name: htpasswd_auth
    challenge: true
    login: true
    mappingMethod: "claim"
    provider:
      apiVersion: v1
      kind: HTPasswdPasswordIdentityProvider
      file: /etc/origin/master/users.htpasswd
{{< / highlight >}}

Now we want to restart the master (this will shut it down and then bring it back 
up again):

```
oc cluster down
```

You might want to go ahead and create some more users too!

```
htpasswd /etc/origin/master/users.htpasswd monica
```

Now you can log in to the OpenShift web console with any of these users.

## WebLogic!

Now we have OpenShift up and running and we can log in successfully, we are ready
to start running some applications on there. Let's start with WebLogic! 

TODO TODO TODO

could not work out how to pull from docker hub on the openshift machine - they take over all 
your regisry defs :( - so i pulled the image and `docker save`d it on my machine and then 
scp'd and `docker load`ed it.

then i needed to get the token, log in to internal registry and tag and push the image:

```
oc whoami -t
oc registry info
docker login -u mark -p <token> docker-registry.default.svc:5000
docker tag c6bb22ff0ea8 docker-registry.default.svc:5000/test2/weblogic:12.2.1.3
docker push docker-registry.default.svc:5000/test2/weblogic:12.2.1.3
```

created an application from this image, added the env vars `WLUSER` and `WLPASSWORD` 
for user and password, could not see how to set command, etc, in the UI so I did
the "edit YAML" and added them in.  The yaml is save in [this file](weblogic.yml).

did a manual deploy, the pod crashed, log shows this:

```
/bin/bash: /u01/oracle/createAndStartEmptyDomain.sh: Permission denied
```

investigating...

need to create the `domain.properties` file and mount it in `/u01/oracle/properties`

created a file and a configmap from it:

```
cat domain.properties
username=weblogic
password=welcome1

oc create configmap weblogic --from-file=domain.properties
configmap/weblogic created
```

then did "add to application" in the web ui, chose volume, and provide path...

updated the security context `restricted` to change `RUNASUSER` from `MustRunAsRange` to `RunAsAny`

```
oc --config=/etc/origin/master/admin.kubeconfig edit scc restricted
```

it is coming up!!!!   and, it works!!!

now for the operator...

# installing the operator

cloned the repo, checked out `v1.1` tag, did a `mvn clean install` and built the image:

```
docker build -t weblogic-kubernetes-operator:openshift --no-cache=true .
```

had to fix the `Dockerfile` to put the right version in there.   then did a `docker save`,
scp'd to openshift box and `docker load`ed it.

then added to the openshift repo (tag and push as per notes above for weblogic)

image name is `weblogic-kubernetes-operator:openshift`.

created the `create-weblogic-operator-inputs.yaml` file, see [here](operator/create-weblogic-operator-inputs.yaml). 

ran the script with "generate only" option to create the files:

```
./create-weblogic-operator.sh -i ./create-weblogic-operator-inputs.yaml -o /root/output -g
```

ran the security yml:

```
KUBECONFIG=/etc/origin/master/admin.kubeconfig oc apply -f weblogic-operator-security.yaml
namespace/weblogic-operator created
serviceaccount/weblogic-operator created
clusterrole.rbac.authorization.k8s.io/weblogic-operator-cluster-role created
clusterrole.rbac.authorization.k8s.io/weblogic-operator-cluster-role-nonresource created
clusterrolebinding.rbac.authorization.k8s.io/weblogic-operator-operator-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/weblogic-operator-operator-rolebinding-nonresource created
clusterrolebinding.rbac.authorization.k8s.io/weblogic-operator-operator-rolebinding-discovery created
clusterrolebinding.rbac.authorization.k8s.io/weblogic-operator-operator-rolebinding-auth-delegator created
clusterrole.rbac.authorization.k8s.io/weblogic-operator-namespace-role created
rolebinding.rbac.authorization.k8s.io/weblogic-operator-rolebinding created
rolebinding.rbac.authorization.k8s.io/weblogic-operator-rolebinding created
```

ran the operator yaml:

```
KUBECONFIG=/etc/origin/master/admin.kubeconfig oc apply -f weblogic-operator.yaml
configmap/weblogic-operator-cm created
secret/weblogic-operator-secrets created
deployment.apps/weblogic-operator created
service/internal-weblogic-operator-svc created
```

# verifying operator install

checking operator is present:

```
KUBECONFIG=/etc/origin/master/admin.kubeconfig oc get crd
NAME                                    CREATED AT
alertmanagers.monitoring.coreos.com     2018-11-27T00:34:10Z
bundlebindings.automationbroker.io      2018-11-27T00:36:29Z
bundleinstances.automationbroker.io     2018-11-27T00:36:29Z
bundles.automationbroker.io             2018-11-27T00:36:30Z
domains.weblogic.oracle                 2018-11-28T17:02:20Z
prometheuses.monitoring.coreos.com      2018-11-27T00:34:10Z
prometheusrules.monitoring.coreos.com   2018-11-27T00:34:10Z
servicemonitors.monitoring.coreos.com   2018-11-27T00:34:10Z

KUBECONFIG=/etc/origin/master/admin.kubeconfig oc get domains
No resources found.

KUBECONFIG=/etc/origin/master/admin.kubeconfig oc -n weblogic-operator get pods,services
NAME                                     READY     STATUS    RESTARTS   AGE
pod/weblogic-operator-7d64b556d7-hzz88   1/1       Running   0          46s

NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/internal-weblogic-operator-svc   ClusterIP   172.30.74.165   <none>        8082/TCP   46s

KUBECONFIG=/etc/origin/master/admin.kubeconfig oc -n weblogic-operator logs weblogic-operator-7d64b556d7-hzz88
Launching Oracle WebLogic Server Kubernetes Operator...
{"timestamp":"11-28-2018T17:02:19.483+0000","thread":1,"level":"INFO","class":"oracle.kubernetes.operator.TuningParametersImpl","method":"update","timeInMillis":1543424539483,"message":"Reloading tuning parameters from Operator's config map","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:19.686+0000","thread":1,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"main","timeInMillis":1543424539686,"message":"Oracle WebLogic Server Kubernetes Operator, version: 1.1, implementation: 1e7cc89f6a838b9611cc7c86778f8af5aee717a9.1e7cc89, build time: 2018-11-27T15:52:00-0500","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:19.691+0000","thread":1,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"startLivenessThread","timeInMillis":1543424539691,"message":"Starting Operator Liveness Thread","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:19.701+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"begin","timeInMillis":1543424539701,"message":"Operator namespace is: weblogic-operator","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:19.703+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"begin","timeInMillis":1543424539703,"message":"Operator target namespaces are: default, test2","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:19.706+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"begin","timeInMillis":1543424539706,"message":"Operator service account is: weblogic-operator","exception":"","code":"","headers":{},"body":""}
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
{"timestamp":"11-28-2018T17:02:19.971+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424539971,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.147+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.CRDHelper","method":"checkAndCreateCustomResourceDefinition","timeInMillis":1543424540147,"message":"Create Custom Resource Definition: class V1beta1CustomResourceDefinition {\n    apiVersion: apiextensions.k8s.io/v1beta1\n    kind: CustomResourceDefinition\n    metadata: class V1ObjectMeta {\n        annotations: null\n        clusterName: null\n        creationTimestamp: null\n        deletionGracePeriodSeconds: null\n        deletionTimestamp: null\n        finalizers: null\n        generateName: null\n        generation: null\n        initializers: null\n        labels: null\n        name: domains.weblogic.oracle\n        namespace: null\n        ownerReferences: null\n        resourceVersion: null\n        selfLink: null\n        uid: null\n    }\n    spec: class V1beta1CustomResourceDefinitionSpec {\n        group: weblogic.oracle\n        names: class V1beta1CustomResourceDefinitionNames {\n            categories: null\n            kind: Domain\n            listKind: null\n            plural: domains\n            shortNames: [dom]\n            singular: domain\n        }\n        scope: Namespaced\n        subresources: null\n        validation: null\n        version: v1\n    }\n    status: null\n}","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.248+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.HealthCheckHelper","method":"performK8sVersionCheck","timeInMillis":1543424540248,"message":"Verifying Kubernetes minimum version","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.264+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.HealthCheckHelper","method":"performK8sVersionCheck","timeInMillis":1543424540264,"message":"Kubernetes version is: v1.11.0+d4cacc0","exception":"","code":"","headers":{},"body":""}
loading from jar:file:/operator/weblogic-kubernetes-operator.jar!/scripts
{"timestamp":"11-28-2018T17:02:20.299+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ConfigMapHelper$ConfigMapContext","method":"loadScriptsFromClasspath","timeInMillis":1543424540299,"message":"Loading scripts into domain control config map for namespace: null","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.304+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"readExistingDomains","timeInMillis":1543424540304,"message":"Listing WebLogic Domains","exception":"","code":"","headers":{},"body":""}
loading from jar:file:/operator/weblogic-kubernetes-operator.jar!/scripts
{"timestamp":"11-28-2018T17:02:20.309+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ConfigMapHelper$ConfigMapContext","method":"loadScriptsFromClasspath","timeInMillis":1543424540309,"message":"Loading scripts into domain control config map for namespace: null","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.311+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.Main","method":"readExistingDomains","timeInMillis":1543424540311,"message":"Listing WebLogic Domains","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.315+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.HealthCheckHelper","method":"performSecurityChecks","timeInMillis":1543424540315,"message":"Verifying that operator service account can access required operations on required resources","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.315+0000","thread":25,"level":"INFO","class":"oracle.kubernetes.operator.helpers.HealthCheckHelper","method":"performSecurityChecks","timeInMillis":1543424540315,"message":"Verifying that operator service account can access required operations on required resources","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.321+0000","thread":25,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540321,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.331+0000","thread":22,"level":"WARNING","class":"oracle.kubernetes.operator.helpers.AuthorizationProxy","method":"review","timeInMillis":1543424540331,"message":"Exception thrown: {0}","exception":"\nio.kubernetes.client.ApiException: Internal Server Error\n\tat io.kubernetes.client.ApiClient.handleResponse(ApiClient.java:882)\n\tat io.kubernetes.client.ApiClient.execute(ApiClient.java:798)\n\tat io.kubernetes.client.apis.AuthorizationV1Api.createSelfSubjectRulesReviewWithHttpInfo(AuthorizationV1Api.java:429)\n\tat io.kubernetes.client.apis.AuthorizationV1Api.createSelfSubjectRulesReview(AuthorizationV1Api.java:414)\n\tat oracle.kubernetes.operator.helpers.CallBuilder$SynchronousCallFactoryImpl.createSelfSubjectRulesReview(CallBuilder.java:1521)\n\tat oracle.kubernetes.operator.helpers.CallBuilder.createSelfSubjectRulesReview(CallBuilder.java:1210)\n\tat oracle.kubernetes.operator.helpers.AuthorizationProxy.review(AuthorizationProxy.java:300)\n\tat oracle.kubernetes.operator.helpers.HealthCheckHelper.performSecurityChecks(HealthCheckHelper.java:126)\n\tat oracle.kubernetes.operator.Main$StartNamespaceBeforeStep.apply(Main.java:285)\n\tat oracle.kubernetes.operator.work.Fiber._doRun(Fiber.java:621)\n\tat oracle.kubernetes.operator.work.Fiber.doRun(Fiber.java:548)\n\tat oracle.kubernetes.operator.work.Fiber.run(Fiber.java:487)\n\tat oracle.kubernetes.operator.work.ThreadLocalContainerResolver.lambda$null$0(ThreadLocalContainerResolver.java:87)\n\tat java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)\n\tat java.util.concurrent.FutureTask.run(FutureTask.java:266)\n\tat java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180)\n\tat java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)\n\tat java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)\n\tat java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)\n\tat java.lang.Thread.run(Thread.java:748)\n","code":"500","headers":{"Cache-Control":["no-store"],"Content-Type":["text/plain; charset=utf-8"],"X-Content-Type-Options":["nosniff"],"Date":["Wed, 28 Nov 2018 17:02:20 GMT"],"Content-Length":["70"],"OkHttp-Sent-Millis":["1543424540325"],"OkHttp-Received-Millis":["1543424540330"]},"body":"This request caused apiserver to panic. Look in the logs for details.\n"}
{"timestamp":"11-28-2018T17:02:20.381+0000","thread":25,"level":"WARNING","class":"oracle.kubernetes.operator.helpers.AuthorizationProxy","method":"review","timeInMillis":1543424540381,"message":"Exception thrown: {0}","exception":"\nio.kubernetes.client.ApiException: Internal Server Error\n\tat io.kubernetes.client.ApiClient.handleResponse(ApiClient.java:882)\n\tat io.kubernetes.client.ApiClient.execute(ApiClient.java:798)\n\tat io.kubernetes.client.apis.AuthorizationV1Api.createSelfSubjectRulesReviewWithHttpInfo(AuthorizationV1Api.java:429)\n\tat io.kubernetes.client.apis.AuthorizationV1Api.createSelfSubjectRulesReview(AuthorizationV1Api.java:414)\n\tat oracle.kubernetes.operator.helpers.CallBuilder$SynchronousCallFactoryImpl.createSelfSubjectRulesReview(CallBuilder.java:1521)\n\tat oracle.kubernetes.operator.helpers.CallBuilder.createSelfSubjectRulesReview(CallBuilder.java:1210)\n\tat oracle.kubernetes.operator.helpers.AuthorizationProxy.review(AuthorizationProxy.java:300)\n\tat oracle.kubernetes.operator.helpers.HealthCheckHelper.performSecurityChecks(HealthCheckHelper.java:126)\n\tat oracle.kubernetes.operator.Main$StartNamespaceBeforeStep.apply(Main.java:285)\n\tat oracle.kubernetes.operator.work.Fiber._doRun(Fiber.java:621)\n\tat oracle.kubernetes.operator.work.Fiber.doRun(Fiber.java:548)\n\tat oracle.kubernetes.operator.work.Fiber.run(Fiber.java:487)\n\tat oracle.kubernetes.operator.work.ThreadLocalContainerResolver.lambda$null$0(ThreadLocalContainerResolver.java:87)\n\tat java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)\n\tat java.util.concurrent.FutureTask.run(FutureTask.java:266)\n\tat java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180)\n\tat java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)\n\tat java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)\n\tat java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)\n\tat java.lang.Thread.run(Thread.java:748)\n","code":"500","headers":{"Cache-Control":["no-store"],"Content-Type":["text/plain; charset=utf-8"],"X-Content-Type-Options":["nosniff"],"Date":["Wed, 28 Nov 2018 17:02:20 GMT"],"Content-Length":["70"],"OkHttp-Sent-Millis":["1543424540380"],"OkHttp-Received-Millis":["1543424540381"]},"body":"This request caused apiserver to panic. Look in the logs for details.\n"}
{"timestamp":"11-28-2018T17:02:20.792+0000","thread":30,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ConfigMapHelper$ConfigMapContext$CreateResponseStep","method":"onSuccess","timeInMillis":1543424540792,"message":"Creating domain config map, test2, for namespace: {1}.","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.792+0000","thread":22,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ConfigMapHelper$ConfigMapContext$CreateResponseStep","method":"onSuccess","timeInMillis":1543424540792,"message":"Creating domain config map, default, for namespace: {1}.","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.872+0000","thread":39,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540872,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.872+0000","thread":38,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540872,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.879+0000","thread":40,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540879,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.879+0000","thread":41,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540879,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.896+0000","thread":42,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540896,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.896+0000","thread":43,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540896,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.919+0000","thread":45,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540919,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.920+0000","thread":44,"level":"INFO","class":"oracle.kubernetes.operator.helpers.ClientPool","method":"getApiClient","timeInMillis":1543424540920,"message":"The Kuberenetes Master URL is set to https://172.30.0.1:443","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:20.970+0000","thread":34,"level":"INFO","class":"oracle.kubernetes.operator.rest.RestServer","method":"start","timeInMillis":1543424540970,"message":"Did not start the external ssl REST server because external ssl has not been configured.","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:21.927+0000","thread":34,"level":"INFO","class":"oracle.kubernetes.operator.rest.RestServer","method":"start","timeInMillis":1543424541927,"message":"Started the internal ssl REST server on https://0.0.0.0:8082/operator","exception":"","code":"","headers":{},"body":""}
{"timestamp":"11-28-2018T17:02:22.483+0000","thread":20,"level":"INFO","class":"oracle.kubernetes.operator.TuningParametersImpl","method":"update","timeInMillis":1543424542483,"message":"Reloading tuning parameters from Operator's config map","exception":"","code":"","headers":{},"body":""}
```

analyzing that now :) 


# creating a domain

set up a secret for admin creds:

```
oc -n test2 create secret generic domain1-weblogic-credentials --from-literal=username=weblogic --from-literal=password=welcome1
```

create directory for host persistent volume:

```
mkdir ~opc/persistentVolume1
chown opc:opc ~opc/persistentVolume1
```

created the `create-weblogic-domain-inputs.yaml` file, see [here](operator/create-weblogic-domain-inputs.yaml). 

ran the script with "generate only" option to create the files:

```
./create-weblogic-domain.sh -i ./create-weblogic-domain-inputs.yaml -o /root/output -g
```

created persistent volume:

```
kubectl apply -f weblogic-domain-pv.yaml
persistentvolume/domain1-weblogic-domain-pv created

oc get pv
NAME                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS                            REASON    AGE
domain1-weblogic-domain-pv   10Gi       RWX            Retain           Available             domain1-weblogic-domain-storage-class             4s
```

create perstistent volume claim:

```
kubectl apply -f weblogic-domain-pvc.yaml
persistentvolumeclaim/domain1-weblogic-domain-pvc created

oc get pvc -n test2
NAME                          STATUS    VOLUME                       CAPACITY   ACCESS MODES   STORAGECLASS                            AGE
domain1-weblogic-domain-pvc   Bound     domain1-weblogic-domain-pv   10Gi       RWX            domain1-weblogic-domain-storage-class   7s
```

created the job to create domain:

```
kubectl apply -f create-weblogic-domain-job.yaml
configmap/domain1-create-weblogic-domain-job-cm created
job.batch/domain1-create-weblogic-domain-job created
```

looks like it failed:

```
oc get pods --show-all -n test2
Flag --show-all has been deprecated, will be removed in an upcoming release
NAME                                       READY     STATUS    RESTARTS   AGE
domain1-create-weblogic-domain-job-64zbd   0/1       Error     0          28s
domain1-create-weblogic-domain-job-sr878   0/1       Error     0          18s
domain1-create-weblogic-domain-job-tctn7   0/1       Error     0          31s
mysql-1-kspmr                              1/1       Running   0          1d
weblogic-5-bcfq6                           1/1       Running   0          23h
```

ok, got perm denied on the pv:

```
oc logs domain1-create-weblogic-domain-job-64zbd -n test2
The domain will be created using the script /u01/weblogic/create-domain-script.sh
mkdir: cannot create directory '/shared/domain': Permission denied
ERROR: Unable to create folder /shared/domain
```

forgot to make it 777, changed it and trying again... still the same error :(

looking at the pod description appears to have all the right volumes and mounts
and the pv's look fine.  going to go back and check, might be another security
contest issue??

scc `restricted` has `FSGroup Strategy: MustRunAs` - i think changing this to 
`RunAsAny` will unblock me, but i think ultimately, we are going to need to define
our own scc instead of using the built-in one, to avoid unintended consequences.

updated the scc, but still getting same error... investigating more...

updated `create-weblogic-domain-job.yaml` to include a `securityContext` with an `fsGroup` 
of `1000` (which is the `opc` group's GID) and trying again:

still same error.... I think it might be selinux fscontext.. researching that...

testing this change:

```
chcon -u system_u -r object_r -t svirt_sandbox_file_t -l s0 ~opc/persistentVolume1
```

still the same error. :( 

tried:

```
chcon -Rt svirt_sandbox_file_t ~opc
```

still the same... thinking about trying privileged mode to see if that gets me around
the issue, then i can go back and solve it properly later... 

```
oc adm policy add-scc-to-user privileged mark
```

and update the job to include this in the `container` section:

```
securityContext:
  allowPrivilegeEscalation: false
```

trying again... nope.. still seemed to run with restricted scc. 

trying `allowHostDirVolumePlugin: true` in restricted scc... nope..

added `serviceAccount: weblogic-operator` to the pod spec, and gave that SA access
to the privileged scc:

```
oc adm policy add-scc-to-user privileged -n test2 -z weblogic-operator
```

pods are still running with scc=restricted, must be missing something...  investigating...

added an annotation to the pod spec to set scc:

```
annotations:
  openshift.io/scc: privileged
```

sad face :(

interesting, the job shows the privileged annotation in the pod spec, but the 
pods are not picking up that scc...

```
oc -n test2 describe job
Name:           domain1-create-weblogic-domain-job
Namespace:      test2
Selector:       controller-uid=1c4c23c3-f350-11e8-9659-000017009a15
Labels:         app=domain1-create-weblogic-domain-job
                controller-uid=1c4c23c3-f350-11e8-9659-000017009a15
                job-name=domain1-create-weblogic-domain-job
                weblogic.domainName=base_domain
                weblogic.domainUID=domain1
                weblogic.resourceVersion=domain-v1
Annotations:    kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"domain1-create-weblogic-domain-job","namespace":"test2"},"spec":{"template":...
Parallelism:    1
Completions:    1
Start Time:     Wed, 28 Nov 2018 12:56:43 -0800
Pods Statuses:  0 Running / 0 Succeeded / 5 Failed
Pod Template:
  Labels:       app=domain1-create-weblogic-domain-job
                controller-uid=1c4c23c3-f350-11e8-9659-000017009a15
                job-name=domain1-create-weblogic-domain-job
                weblogic.domainName=base_domain
                weblogic.domainUID=domain1
                weblogic.resourceVersion=domain-v1
  Annotations:  openshift.io/scc=privileged
  Containers:
```

today, i can't ssh into the machine, i guess the selinux changed to ~opc has upset my 
ability to read the `/ssh` directory :(  i guess i have to start over, at least that
lets me test my notes work...

finished the reinstall, went pretty smoothly. 
missed one thing in the doc - need to tag the images to import them into an 
openshift image stream: 

```
docker images | grep weblogic # get the hash
docker tag <hash> weblogic:12.2.1.3  # need to do this because oc tag does not 
                                     # accept qualifiers "store/oracle/weblogic"
oc tag weblogic:12.2.1.3 test2/weblogic:12.2.1.3

oc tag weblogic-kubernetes-operator:openshift test2/weblogic-kubernetes-operator:openshift
```

# nfs

since hostPath PVs are proving so challenging, I decided to try nfs instead.

created an `/etc/exports`: 

```
/data/domain1 10.0.2.13(rw,sync)
```

updated my `weblogic-domain-pv.yaml` as follows:

```
# Copyright 2017, 2018, Oracle Corporation and/or its affiliates.  All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.

apiVersion: v1
kind: PersistentVolume
metadata:
  name: domain1-weblogic-domain-pv
  labels:
    weblogic.resourceVersion: domain-v1
    weblogic.domainUID: domain1
    weblogic.domainName: base_domain
spec:
  storageClassName: domain1-weblogic-domain-storage-class
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  # Valid values are Retain, Delete or Recycle
  persistentVolumeReclaimPolicy: Retain
  # hostPath:
  nfs:
    server: 10.0.2.13
    path: "/data/domain1"
```

created pv, pvc, ran the domain job, storage working well, but got a new error to debug:

```
oc logs domain1-create-weblogic-domain-job-gzbkc
The domain will be created using the script /u01/weblogic/create-domain-script.sh
/u01/weblogic/create-domain-script.sh: line 10: wlst.sh: command not found
Successfully Completed
```

i think i might have cracked it.. the container is running as some special openshift user..

i updated the `create-domain-script.sh` (in the config map) to include this:

```
  create-domain-script.sh: |-
    #!/bin/bash
    #

    # Include common utility functions
    source /u01/weblogic/utility.sh

    export DOMAIN_HOME=${SHARED_PATH}/domain/base_domain

    # mark debugging
    whoami
    echo $PATH
    find /u01 -name wlst.sh
```

output i get is this: 

```
The domain will be created using the script /u01/weblogic/create-domain-script.sh
whoami: cannot find name for user ID 1000080000
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/java/default/bin:/u01/oracle/oracle_common/common/bin:/u01/oracle/wlserver/common/bin
find: '/u01/oracle': Permission denied
find: failed to restore initial working directory: Permission denied
/u01/weblogic/create-domain-script.sh: line 15: wlst.sh: command not found
Successfully Completed
```

So i think i need to work out how to get it to run as `oracle` user (1000).

first part i think is a policy:

```
oc adm policy add-scc-to-user anyuid -z mark -n test2
```

but that alone is not enough, i think i also need to tell it somehow which user
to use.  researching that now...  i think this is also why i could not write to 
the hostPath PV - will be able to validate that later.

worked out how to give permission to all authenticated users to run as the user
specified in the Dockerfile:

```
oc adm policy add-scc-to-group anyuid system:authenticated
```

this gets me unblocked, but will need to work out later how to make this scoped
to just the required user, not everyone. 

# domain crd

created the domain crd and validate:

```
oc apply -f domain-custom-resource.yaml
domain.weblogic.oracle/domain1 created
[root@openshift domain1]# oc get domains
NAME      AGE
domain1   2s
[root@openshift domain1]# oc describe domain domain1
Name:         domain1
Namespace:    test2
Labels:       weblogic.domainName=base_domain
              weblogic.domainUID=domain1
              weblogic.resourceVersion=domain-v1
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"weblogic.oracle/v1","kind":"Domain","metadata":{"annotations":{},"labels":{"weblogic.domainName":"base_domain","weblogic.domainUID":"dom...
API Version:  weblogic.oracle/v1
Kind:         Domain
Metadata:
  Creation Timestamp:  2018-11-30T17:47:55Z
  Generation:          1
  Resource Version:    187463
  Self Link:           /apis/weblogic.oracle/v1/namespaces/test2/domains/domain1
  UID:                 11654ec5-f4c8-11e8-9280-000017007294
Spec:
  Admin Secret:
    Name:   domain1-weblogic-credentials
  As Name:  admin-server
  As Port:  7001
  Cluster Startup:
    Cluster Name:   cluster-1
    Desired State:  RUNNING
    Env:
      Name:     JAVA_OPTIONS
      Value:    -Dweblogic.StdoutDebugEnabled=false
      Name:     USER_MEM_ARGS
      Value:    -Xms64m -Xmx256m
    Replicas:   2
  Domain Name:  base_domain
  Domain UID:   domain1
  Export T 3 Channels:
    T3Channel
  Image:              weblogic:12.2.1.3
  Image Pull Policy:  IfNotPresent
  Image Pull Secret:
    Name:  <nil>
  Server Startup:
    Desired State:  RUNNING
    Env:
      Name:         JAVA_OPTIONS
      Value:        -Dweblogic.StdoutDebugEnabled=false
      Name:         USER_MEM_ARGS
      Value:        -Xms64m -Xmx256m
    Node Port:      30701
    Server Name:    admin-server
  Startup Control:  AUTO
Events:             <none>
[root@openshift domain1]#
```

ok this looks good.  now to get the oeprator working!!

# back to the operator

changed the logging level on the operator to get more info (thanks ryan)

set `JAVA_LOGGING_LEVEL` env var in operator deployment to `FINER`

restarted the operator, it came up with more logging - although it also 
noticed the domain CR and started the domain!  i can hit the admin console
through an ssh tunnel !!! :woot: 

let's try sclaing!

edit the domain:

```
oc edit domain domain1
```

changed replicas to 1, and it did shut down a managed server as expected. 

changed replicas to 2, and it started up the managed server again as expected. 

changed replicas to 6, and i see an error in the log that says 6 is greater than
the configured number of servers in the cluster (2), which is the expected
behavior. 

# RBAC analysis

I took a look at the userland option - which would allow a regular user to 
install the operator into their namespace/project, e.g. on openshift online, 
where they do not have admin roles. 

Created [this spreadsheet](RBAC.xls) which lists the differences.  Cols A-D show
what a normal user has.  Col E shows what the operator wants: 

* green means the user has everything that the operator needs,
* orange means the user has some of what the operator needs,
* red means the user has none of what the operator needs

So we will need to work out a plan for that. 

The operator cod ealso assumes it can get service accounts, persistent volumes, 
etc., so I think we would need to actually change the code to have a "userland"
mode that is aware of these limits.




abc

## Installing OpenShift

abc

## Getting ready to run WebLogic

abc

## Getting the WebLogic Docker image

abc

## Setting up the WebLogic Operator for Kubernetes

abc
