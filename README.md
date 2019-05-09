# MariaDB and Kubernetes

MariaDB is an awesome SQL database and has some fantastic features that are all containerized and can be very secure. The combination of MariaDB and MaxScale is a great opportunity to move to the cloud and use Kubernetes as container orchestrating system. I just want to make the more secure way easier to find and build. I have made manifests for both clusters, Master slave and Galera, that are fully encrypted and only use secure, encrypted connections. It does this by using the built in MariaDB and MaxScale features for data at rest encryption and making SSL traffic required for all users. 

## The problem with the MariaDB helm cart and the Kubernetes operator
The helm cart and the Kubernetes operator both use something called a Statestore. The MariaDB Statestore, as of now, has 0 documentation on what it does or how to change it. Well, it basically runs a sidecar on your MariaDB and MaxScale deployments to update MaxScale on the fly if any of the pods role over. Which would be fine if it did not use just the default username and password to configure MaxScale. 

What this means: someone can look up the default username and password to configure your database proxy. Then configure the MaxScale instance to point to only a random IP address, or worse, another SQL server.

This is a major security hole.

The work around: **DO NOT USE STATESTORE**

These cluster manifests run without Statestore and are automatically updated over the built in functionality of MariaDB and MaxScale alone.

## How these manifests work

This manifests use the Kubernetes DNS to automatically update the clusters if something goes wrong and a pod roles. The default DNS structure is {pod-name}.{service-name}.{namspace}.svc.cluster.local

If you have a different namespace other than `mariadb`, you will need to go in and change each of these DNS names to fit your namepace. 

The MaxScale 2.3 does not work for the Master Slave cluster, so we are using the MaxScale 2.2 version. The current bug ticket for that is: https://jira.mariadb.org/browse/MDEV-19315

MariaDB has very good documentation on how this all works. The knowledge center is your best friend if you do not understand something.

Each cluster has a guide for how to get up and running. The Master slave is scripted failover, so it has a small chance of data loss and is not truly highly available but the Galera cluster is highly available with virtually no downtime. 

## How to set up the SSL certs and encryption keys for these deployments

We are going to make SSL certs and keys from openssl. 

Generate a new key:
```
$ openssl genrsa 2048 > ca-key.pem
Generating RSA private key, 2048 bit long modulus
.......................................................+++
.....+++
e is 65537 (0x10001)
```
Make the cert from the key, please do not use "root" as the Common Name. Also, you should probably fill out the rest of the fields too.
```
$ openssl req -new -x509 -nodes -days 3650 -key ca-key.pem -out ca.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Organizational Unit Name (eg, section) []:
Common Name (eg, fully qualified host name) []:root
Email Address []:
```
Generate a new key. Please do not use "server" as the common name here either. 
```
$ openssl req -newkey rsa:2048 -days 3650 -nodes -keyout server-key.pem -out server-req.pem
Generating a 2048 bit RSA private key
................................................+++
.............+++
writing new private key to 'server-key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Organizational Unit Name (eg, section) []:
Common Name (eg, fully qualified host name) []:server
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:

$ openssl rsa -in server-key.pem -out server-key.pem
writing RSA key

$ openssl x509 -req -in server-req.pem -days 3650 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem
Signature ok
subject=/CN=server
Getting CA Private Key
```

And you now have all you need for the SSL part. Go into each file that was just generated and remove the new line character at the end of the file. So the last like should be "-----END CERTIFICATE-----" not a empty line. 

Then run `$ cat server-cert.pem | base64` and put that output on line 12 of the mariadb-secret.yml file after `server.cert:` 
Then run `$ cat server-key.pem | base64` and put that output on line 13 of the mariadb-secret.yml file after `server.key:` 
Then run `$ cat ca.pem | base64` and put that output on line 14 of the mariadb-secret.yml file after `ca.cert:` 

### Getting encryption at rest working

We have to make a keyfile:
```
$ openssl rand -hex 32 >> keyfile
$ openssl rand -hex 32 >> keyfile
$ openssl rand -hex 32 >> keyfile
$ openssl rand -hex 32 >> keyfile
```

Then open the key file and edit it to look like this:
```
1;bjk4piuq34tgrqphg34q9hqg34punberqgh89qrgh89qrgh8g4389hh4g3h
2;07ht340hg4230g342g23h07grhiugrh07g9h78gr89hegrw8egrh78grehu
3;0743rgoh7g3h78grh87grh78g3h7g3r78hogrh78ogrho78grh78orgho78
4;h79q3h7o8gho7fho7fho7c7ho7ho43oh74wo7euifhougoalor7to7gwUEO
```
{number};{encryption key} is what your file should look like. You can put any numbers there. But the first one has to be 1.

Make a keyfile.key so we are not leaving the encryption keys in plain text on the server.

```
$ openssl rand -hex 128 > keyfile.key
```

Then make the keyfile encrypted with the keyfile.key

```
$ openssl enc -aes-256-cbc -md sha1 -pass file:keyfile.key -in keyfile -out keyfile.enc
```

Then run `cat keyfile.enc | base64` and put the output on line 3 after  `keyfile.enc:`

Then run `cat keyfile.key | base64` and put the output on line 4 after  `keyfile.key:`

You do need to go into the respect `mariadb.yml` files and change the `users.sql` file to require the correct Issuer and Subject (the Common Names from the SSL certs). Or you could just require SSL or x509, but requiring a specific Issuer and Subject are more secure. 

Now deploy the secrects by running: `$ kubectl apply -f mariadb-secrect.yml`