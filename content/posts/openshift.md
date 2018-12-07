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

I am going to set up OpenShift Enterprise 3.11 on a single compute instance on Oracle Cloud Infrastructure (OCI), so I am using the "all in one" 
configuration.  This means that the master and the worker node is on the 
same machine.  This is not a production configuration, but it is fine for
testing purposes.

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

**Creating the image**

You can download my sample code from GitHub by cloning 
[this repository](https://github.com/markxnelson/weblogic-on-openshift) as 
follows:

```
git clone https://github.com/markxnelson/weblogic-on-openshift
```

abc

```
terraform plan
```


This example provides a method to generate a RHEL 7.4 image for use by both VM and BM shapes.

Please consult the Changelog for the latest changes to this process!

There are several prerequisites:

1. You MUST setup a Dynamic Group for the Compartment in which you are going to run this process.  The
   Dynamic Group allows the ipxe instance itself to authenticate to OCI so that no user configuration is
   needed to create the image.

   Information on how to create a Dynamic Group can be found here:
   https://docs.us-phoenix-1.oraclecloud.com/Content/Identity/Tasks/managingdynamicgroups.htm?Highlight=Dynamic%20Group

   In short, from the console:
   - Get the Compartment OCID for the Compartment you will be using.
   - In Identity, select Dynamic Groups
   - Click on the Create Dynamic Group box
   - Specify a name for the group
   - Click on the link labeled "Launch Rule Builder"
   - Select 'in Compartment ID' as the Resource Attribute
   - Enter the Compartment OCID in the Value box
   - Click on the Add Rule button
   - Click on the Create Dynamic Group button.

   The compartment is now enabled for Instance Principals.  If you do not want this after the image is
   created, simply delete the Dynamic Group AFTER the image is created.  Deletion of the DG will not
   affect the usability of the image.

   You MUST create a policy that gives this dynamic group permission to use OCI APIs. The simplest way
   is to just let it call all APIs for the compartment you are using, by adding a policy to the root
   compartment with a rule like this:

```
Allow dynamic-group CertificationSubscriptionUsers to manage all-resources in compartment Certification
```


2. You MUST have a valid RedHat account with subscriptions available.  The TF template needs a
   RH Username and Password to allow you to temporarily subscribe the instance that is building the image
   and get access to the various RH repos.
2. The template expects pre-configured VCNs and Subnets.
3. You need to provide a URL that points to the RHEL 7.4 ISO.  This URL must contain the name of the ISO,
   with an '.iso' extension.  An OCI Pre-Authenticated Request (PAR) works well for this operation.  How to create
   OCI PARs can be found here: https://docs.us-phoenix-1.oraclecloud.com/Content/Object/Tasks/managingobjects.htm#par.
4. The template uses filters that expect unique Compartment, VCN and Subnet names.
   NOTE: The root compartment CANNOT be used for this process.
5. The following must be specified in your shell environment (prefixed with TF_VAR_ of course):
    - tenancy_ocid
    - user_ocid
    - fingerprint
    - private_key_path
    - private_key_password (if required)
    - ssh_public_key (the actual public key, not the file)
    - region
6. The subnet to be used must have the following configuration:
	- Port 80 TCP must be allowed on the subnet
	- All ICMP traffic must be allowed on the subnet (ICMP All)

NOTE: A template env-vars file is provided as part of this example.  Simply complete the items inside the template and source the result into your shell by using:

```
. ./env-vars
```

Using this template is simple:

1. Set your environment variables
2. Open the configuration.tf file and substitute the values in each of the sections appropriate to your environment
   NOTE: The AD is specified as either 'AD-x' or 'ad-x' where x is the AD number you wish to use for the process.
3. Execute 'terraform plan; terraform apply'
4. Get coffee or favorite beverage...
5. After your image is created, execute 'terraform destroy -force' (there will not be a resource to actually kill,
   so force is required).

What happens in the background:
The template generates a script that embeds all the configuration files needed to build the iPXE server, extract the ISO
boot the instance used to load RHEL, causes RHEL to load, builds the image, destroys the build instance, and finally destroys the iPXE server.  You are left with a custom image named "RHEL_74" in your environment.

NOTE: The source configuration files for the iPXE server are included here.  It is *STRONGLY* recommended that they not be
      altered.

Enjoy.



## Creating a compute instance

abc

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

