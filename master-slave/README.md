# Kubernetes MariaDB Master Slave

After you have made and deployed the secrets in `mariadb-secret.yml` then you can just run the normal command:

```
kubectl apply -f mariadb.yml,maxscale.yml
```

Then you will have a MariaDB Master Slave cluster. If you want true high availability then see the Galera cluster. 

The reason why we have to start the replication to the mariadb-0 pod is because MaxScale version 2.3 does not work for Kubernetes deployments, so there is no other way to make replication use SSL unless you script it on start up. If the pod ever restarts it remembers that we are using SSL and continues to replicate over SSL. 