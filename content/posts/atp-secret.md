---
title: "Storing ATP Wallets in a Kubernetes Secret"
date: 2019-04-08T09:36:43-04:00
draft: false
tags: [atp]
---

In [this previous post](/posts/atp-datasource), we talked about how to create a
WebLogic datasource for an ATP database.  In that example we put the ATP 
wallet into the domain directly, which is fine if your domain is on a secure
environment, but if we want to use ATP from a WebLogic domain running in 
Kubernetes, you might not want to burn the wallet into the Docker image. 
Doing so would enable anyone with access to the Docker image to retrieve 
the wallet.

A more reasonable thing to do in the Kubernetes environment would be to put
the ATP wallet into a Kubernetes secret and mount that secret into the
container. 

You will, of course need to decide where you are going to mount it and update
the sqlnet.ora with the right path, like we did in the previous post. Once
that is taken care of, you can create the secret from the wallet using a
small script like this:

```
#!/bin/bash
# Copyright 2019, Oracle Corporation and/or its affiliates. All rights reserved.

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: atp-secret
  namespace: default
type: Opaque
data:
  ojdbc.properties: `cat ojdbc.properties | base64 -w0`
  tnsnames.ora: `cat tnsnames.ora | base64 -w0`
  sqlnet.ora: `cat sqlnet.ora | base64 -w0`
  cwallet.sso: `cat cwallet.sso | base64 -w0`
  ewallet.p12: `cat ewallet.p12 | base64 -w0`
  keystore.jks: `cat keystore.jks | base64 -w0`
  truststore.jks: `cat truststore.jks | base64 -w0`
EOF
```

We need to base64 encode the data that we put into the secret.  When you
mount the secret on a container (in a pod), Kubernetes will decode it, 
so it appears to the container in its original form. 

Here is an example of how to mount the secret in a container:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-weblogic-server
  labels:
    app: my-weblogic-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-weblogic-server
  template:
    metadata:
      labels:
        app: my-weblogic-server
    spec:
      containers:
      - name: my-weblogic-server
        image: my-weblogic-server:1.2
        volumeMounts:
        - mountPath: /shared
          name: atp-secret
          readOnly: true
      volumes:
       - name: atp-secret
         secret:
           defaultMode: 420
           secretName: atp-secret
```

You will obvisously still need to control access to the secret and the 
running containers, but overall this approach does help to provide a better
security stance.



