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
up for source CIDR `0.0.0.0/0` and destination port `80`:

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

abc

## Installing OpenShift

abc

## Getting ready to run WebLogic

abc

## Getting the WebLogic Docker image

abc

## Setting up the WebLogic Operator for Kubernetes

abc

