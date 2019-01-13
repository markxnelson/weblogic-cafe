---
title: "Relocate the /var/lib/docker storage on an OpenShift system"
date: 2019-01-13T18:02:10-05:00
draft: false
tags: [openshift,docker]
---

Today I ran out of space in the filesystem containing the `/var/lib/docker`
directory on my OpenShift system, which meant I could not schedule any new 
pods (you get an error about disk pressure).  So I decided I needed some 
more space. 

I created a new partition with `parted` and then I updated my `/etc/fstab`
to include that partition, with it mounted at `/var/lib/docker`.  Then I 
made a new XFS filesystem on the with this command:

```
mkfs.xfs /dev/sda4
```

One good thing about OpenShift is that I don't really have to worry about 
keeping the old filesystem.  I can pretty much delete it all and let OpenShift
pull all the images again, which is nice.  There is one little trick though
to make sure it has the right SELinux context.

First, shut down OpenShift:

```
oc adm manage-node openshift --schedulable=false
systemctl stop docker atomic-openshift-node
```

Now delete the old filesystem:

```
rm -rf /var/lib/docker
```

Next, repeat the Docker storage setup (you may get some warnings if you are
not using Logical Volumes, you can ignore those):

```
docker-storage-setup --reset
docker-storage-setup
```

Now make sure that the directory has the right SELinux context.  I did this 
both before and after mounting the filesystem, just to make sure: 

```
chcon -Rt svirt_sandbox_file_t /var/lib/docker
mount /var/lib/docker
chcon -Rt svirt_sandbox_file_t /var/lib/docker
```

You can `touch` a file in there and do an `ls -Z` to check it gets the right
context. 

Now we can start up OpenShift again: 

```
systemctl start docker atomic-openshift-node
oc adm manage-node openshift --schedulable=true
```

And watch it pull all those images and start right back up with much more
space!

