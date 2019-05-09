# Kubernetes MariaDB Galera

This cluster is not just a one command and the cluster is up and running.

Make sure to have all of your certs and keyfiles made and deployed before applying this.

Still do the normal 

`kubectl apply -f mariadb.yml,maxscale.yml`

But after all of the pods are labeled as running, comment out line 37 or mariadb.yml and uncomment line 38. This starts up the same galera pod but it does not make a new cluster because the cluster is already made with first database server. Then you need to increment the replica counter on line 107. Then run the command:

`kubectl apply -f mariadb.yml`

And then you just have to increment the replica counter and apply again until you get to the desired amount of pods. Then you're done! You are free to enjoy your highly available, secure, encrypted, Kubernetes based, MariaDB Galera cluster. It is expected for the joining pods to restart once during their first startup. But after their first start up, the servers should be very quick to come back up if they ever go down.