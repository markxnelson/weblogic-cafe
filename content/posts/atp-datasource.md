---
title: "Configuring a WebLogic Data Source to use ATP"
date: 2019-02-04T12:16:44-05:00
draft: false
tags: [weblogic,atp]
---
In this post I am going to share details about how to configure a WebLogic 
data source to use ATP.  

If you are not familiar with ATP, it is the new [Autonomous Transaction 
Processing](https://cloud.oracle.com/atp) service on Oracle Cloud.  It provides a fully managed autonomous
database.  You can create a new database in the OCI console in the Database
menu under "Autonomous Transaction Processing" by clicking on that big
blue button:

{{< figure src="/images/atp001.png" >}}

You need to give it a name, choose the number of cores and set an admin
password:

{{< figure src="/images/atp002.png" >}}

It will take a few minutes to provision the database.  Once it is ready, 
click on the database to view details.  

{{< figure src="/images/atp003.png" >}}

Then click on the "DB Connection"
button to download the wallet that we will need to connect to the database.

{{< figure src="/images/atp004.png" >}}

You need to provide a password for the wallet, and then you can download it:

{{< figure src="/images/atp005.png" >}}

Copy the wallet to your WebLogic server and unzip it.  You will see the
following files:

```
[oracle@domain1-admin-server atp]$ ls -l
total 40
-rw-rw-r--. 1 oracle oracle 6661 Feb  4 17:40 cwallet.sso
-rw-rw-r--. 1 oracle oracle 6616 Feb  4 17:40 ewallet.p12
-rw-rw-r--. 1 oracle oracle 3241 Feb  4 17:40 keystore.jks
-rw-rw-r--. 1 oracle oracle   87 Feb  4 17:40 ojdbc.properties
-rw-rw-r--. 1 oracle oracle  114 Feb  4 17:40 sqlnet.ora
-rw-rw-r--. 1 oracle oracle 6409 Feb  4 17:40 tnsnames.ora
-rw-rw-r--. 1 oracle oracle 3336 Feb  4 17:40 truststore.jks
```

I put these in a directory called `/shared/atp`.  You need to update the
`sqlnet.ora` to have the correct location as shown below: 

```
WALLET_LOCATION = (SOURCE = (METHOD = file) (METHOD_DATA = (DIRECTORY="/shared/atp")))
SSL_SERVER_DN_MATCH=yes
```

You will need to grab the hostname, port and service name from the `tnsnames.ora`
to create the data source, here is an example:

```
productiondb_high = (description= (address=(protocol=tcps)(port=1522)(host=adb.us-phoenix-1.oraclecloud.com))(connect_data=(service_name=feqamosccwtl3ac_productiondb_high.atp.oraclecloud.com))(security=(ssl_server_cert_dn=
        "CN=adwc.uscom-east-1.oraclecloud.com,OU=Oracle BMCS US,O=Oracle Corporation,L=Redwood City,ST=California,C=US"))   )
```

You can now log in to the WebLogic console and create a data source, give it
a name on the first page:

{{< figure src="/images/atp006.png" >}}

You can take the defaults on the second page: 

{{< figure src="/images/atp007.png" >}}

And the third:

{{< figure src="/images/atp008.png" >}}

On the next page, you need set the database name, hostname and port to the 
values from the `tnsnames.ora`:

{{< figure src="/images/atp009.png" >}}

On the next page you can provide the username and password.  In this example
I am just using the `admin` user.  In a real life scenario you would probably
go and create a "normal" user and use that.  You can find details about how
to set up [SQLPLUS here](https://docs.oracle.com/en/cloud/paas/atp-cloud/atpug/connect-sqlplus.html#GUID-A3005A6E-9ECF-40CB-8EFC-D1CFF664EC5A).

You also need to set up a set of properties that are required for ATP as
shown below, you can find more details in the [ATP documentation](https://docs.oracle.com/en/cloud/paas/atp-cloud/atpug/connect-jdbc-thin-wallet.html#GUID-6D16D2FB-C001-4272-B3E0-1348F5A3EA1B):

```
oracle.net.tns_admin=/shared/atp
oracle.net.ssl_version=1.2
javax.net.ssl.trustStore=/shared/atp/truststore.jks
oracle.net.ssl_server_dn_match=true
user=admin
javax.net.ssl.keyStoreType=JKS
javax.net.ssl.trustStoreType=JKS
javax.net.ssl.keyStore=/shared/atp/keystore.jks
javax.net.ssl.keyStorePassword=WebLogicCafe1
javax.net.ssl.trustStorePassword=WebLogicCafe1
oracle.jdbc.fanEnabled=false
```

Also notice the the URL format is `jdbc:oracle:thin:@cafedatabase_high`, you
just need to put the name in there from the `tnsnames.ora` file:

{{< figure src="/images/atp010.png" >}}

On the next page you can target the data source to the appropriate servers,
and we are done!  Click on the "Finish" button and then you can activate 
changes if you are in production mode. 

{{< figure src="/images/atp011.png" >}}

You can now go and test the data source (in the "Monitoring" tab and then 
"Testing", select the data source and click on the "Test Data Source" button.

{{< figure src="/images/atp012.png" >}}

You will see the success message:

{{< figure src="/images/atp013.png" >}}

Enjoy!

