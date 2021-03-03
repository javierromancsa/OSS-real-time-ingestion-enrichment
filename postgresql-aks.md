## Clone repo to deploy the postgresql server:
```
git clone https://github.com/javierromancsa/postgresql-aks.git
cd postgresql-aks/
```
## Edit the configmap yaml file to update the password and deploy all the yamls
```
kubectl create -f postgres-configmap.yaml
kubectl create -f postgres-storage.yaml
kubectl create -f postgres-deployment.yaml
```
## Check that no errors have been show
```
kubectl get configmaps
kubectl get pvc
kubectl get pods
kubectl get services
```
**Note** If you don't see a private IP on the services output for the postgres deployment them you have to give network contributer role to AKS SP
![pics](https://github.com/javierromancsa/images/blob/main/postgresql-04.png)

https://docs.microsoft.com/en-us/azure/aks/internal-lb#use-private-networks

## Using the services output copy the External-IP address and use it for psql in order to connect to the postgresql
```
psql -h 10.5.5.85 -U postgresadmin --password -p 5432 -d somedb -f movies.sql
```
![pics](https://github.com/javierromancsa/images/blob/main/postgresql-01.png)

I use the following blogs as a reference:
- https://severalnines.com/database-blog/using-kubernetes-deploy-postgresql
- https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv
