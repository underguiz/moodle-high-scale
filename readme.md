## About this project

This project deploys a robust infrastructure on Azure to handle a high scale moodle installation, this environment is able - and tested - to handle **200k concurrent users**.

It uses Azure Kubernetes Service to run Moodle, Azure Storage Account to host course content, Azure Database for PostgreSQL Flexible Server as its database and Azure Front Door to expose the application to the public as well as caching common used assets.

![Architecture](moodle-high-scale.png)

## Create the infrastructure using terraform

Provision the infrastructure.

```
$ cd infra/
$ az login
$ terraform init
$ terraform plan -var moodle-environment=production
$ terraform apply -var moodle-environment=production
$ az aks get-credentials --name moodle-high-scale --resource-group <resource-group>
```

Provision the NFS server to host moodle data (you will need a bastion instance for this to work)

_Add your username as a Virtual Machine Admin User to moodle's data vm_

```
$ az network bastion ssh --name bastion --resource-group <resource-group> --target-resource-id <moodle-data-vm-id> --auth-type "AAD"
$ # add your ~/.ssh/id_rsa.pub content to .ssh/authorized_keys
$ cd ../ansible
$ az network bastion tunnel --name bastion --resource-group <resource-group> --target-resource-id <moodle-data-vm-id> --resource-port 22 --port 2200
$ ansible-playbook -u "<aad-user>" --ssh-extra-args "-p 2200" -i localhost, nfs-provisioning.yaml
```

Provision the Redis Cluster.

```
$ cd ../manifests/redis-cluster
$ kubectl apply -f redis-configmap.yaml
$ kubectl apply -f redis-cluster.yaml
$ kubectl apply -f redis-service.yaml
```

Wait for all the replicas to be running.

```
$ ./init.sh
```

Deploy Moodle and its services.

_Change image in moodle-service.yaml_

```
$ cd ../../images/moodle
$ az acr build --registry moodlehighscale<suffix> -t moodle:v0.1 --file Dockerfile .
$ cd ../../manifests
$ kubectl apply -f pgbouncer-deployment.yaml
$ kubectl apply -f moodle-service.yaml
$ kubectl -n moodle get svc --watch
```

Provision the frontend configuration that will be used to expose Moodle and its assets publicly.

```
$ cd ../frontend
$ terraform init
$ terraform plan
$ terraform apply
```

Approve the private endpoint connection request from Frontdoor in moodle-svc-pls resource.

_Private Link Services > moodle-svs-pls > Private Endpoint Connections > Select the request from Front Door and click on Approve._

Install database.

```
$ kubectl -n moodle exec -it deployment/moodle-deployment -- /bin/bash 
$ php /var/www/html/admin/cli/install_database.php --adminuser=admin_user --adminpass=admin_pass --agree-license
```

Deploy Moodle Cron.

_Change image in moodle-cron.yaml_

```
$ cd ../manifests
$ kubectl apply -f moodle-cron.yaml
```

Your Moodle installation is now ready to use.