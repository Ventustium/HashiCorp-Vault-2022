# Dynamic Database Authentication
In this example I will use MySQL for the Database
<br>
Make Sure you have Dynamic Provisioning in Kubernetes. In this case I use Dynamic NFS Provisioning

## Lets get started

Deploy the MySQL PVC Claim
```
kubectl create -f pvc.yaml
kubectl get pvc
```

Let's run the MySQL Deployment
```
kubectl create -f deployment.yaml
kubectl create -f phpmyadmin.yaml
kubectl get pods
kubectl get svc
```

Here my `kubectul get pods`
```
NAME                                      READY   STATUS    RESTARTS         AGE
mysql-569df69f69-wdpc9                    1/1     Running   0                38m
nfs-client-provisioner-6fc5767b68-cpc7n   1/1     Running   10 (30m ago)     44h
phpmyadmin-deployment-794f57ddd6-lf24s    1/1     Running   0                22m
```

Here my `kubectul get svc`
```
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP                      15d
mysql                ClusterIP   10.102.184.235   <none>        3306/TCP                     40m
nfs-server           ClusterIP   10.110.192.234   <none>        2049/TCP,20048/TCP,111/TCP   9d
phpmyadmin-service   NodePort    10.104.237.130   <none>        80:31285/TCP                 24m
```

You can Access PHPMyAdmin using Cluster-IP <br>
User this credential to Login
```
server:     mysql
user:       root
password:   5up3ru53r!
```

Create a new database and user, here I will use CLI
```
kubectl exec -it mysql-569df69f69-wdpc9 -- mysql -u root -p

CREATE USER hcpvault IDENTIFIED BY 'h4sh1c0rp';
GRANT SELECT, INSERT, UPDATE, DELETE, DROP, CREATE, ALTER, CREATE USER ON *.* to hcpvault WITH GRANT OPTION;
FLUSH PRIVILEGES;
CREATE DATABASE db_example;
```

Enable database engine
```
kubectl -n vault exec -it vault-0 -- vault login
kubectl -n vault exec -it vault-0 -- vault secrets enable database
```

Configure DB Credential creation
```
kubectl -n vault exec -it vault-0 -- sh

vault write database/config/mysql plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(10.102.184.235)/" \
  allowed_roles="mysqlrole" \
  username="hcpvault" \
  password="h4sh1c0rp"

vault write database/roles/mysqlrole db_name=mysql \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; \
  GRANT SELECT, INSERT, DELETE, DROP, CREATE, ALTER ON db_example.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"

vault read database/creds/mysqlrole

Key                Value
---                -----
lease_id           database/creds/mysqlrole/pLKv4xh6Lye1jghmtOA2hfC0
lease_duration     1h
lease_renewable    true
password           7NzQTPjWJk95vcbK-vkJ
username           v-root-mysqlrole-e3NNK7AmR9zGeIB
```

Now you can try login using that account