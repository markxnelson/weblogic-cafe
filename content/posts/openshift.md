---
title: "Running OpenShift on OCI"
date: 2018-12-07T17:00:00-05:00
draft: false
toc: true
tags: [oci,openshift]
---
In this post, I want to share my experiences with running OpenShift on Oracle 
Cloud Infrastructure (OCI).  This is not strictly a "how to" because I did 
set this up as a test/development system, not a production one, so I did take a few 
small short cuts, but I hope that it is still helpful to share.

I am going to set up OpenShift Enterprise 3.11 on a single compute instance 
on Oracle Cloud Infrastructure (OCI), so I am using the "all in one" 
configuration.  This means that the master and the worker node is on the 
same machine.  This is not a production configuration, but it is fine for
testing and development purposes.

In my next post, I am going to talk about running WebLogic on OpenShift.

## Overview

Here's an overview of the process I will walk through in this post:

* Prepare the OCI tenancy to run OpenShift - set up compartments, groups,
  policies, instance principals, and so on
* Create a RHEL 7.4 custom image in OCI (since OCI does not provide any
  RHEL images)
* Create a compute instance using that custom image
* Install and configure OpenShift in that instance

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

## Conclusion

In my next post, I will share my experiences running WebLogic on OpenShift, 
which is why I wanted to set up OpenShift in the first place! 
