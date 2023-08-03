Installation of az cli:

```bash
curl -L https://aka.ms/InstallAzureCli | bash
```
```bash
az list provider | grep RedHatOpenshift
```
(checks if necessary provider is registered for our subscription ID)

Create (optional) custom domain for cluster (set in - -domain when creating cluster)
Export settings:
```
LOCATION=eastus                 # the location of your cluster
RESOURCEGROUP=aro-rg            # the name of the resource group where you want to create your cluster
CLUSTER=cluster                 # the name of your cluster
```

Create a virtual network with 2 subnets:

```
az network vnet create \
   --resource-group $RESOURCEGROUP \
   --name aro-vnet \
   --address-prefixes 10.0.0.0/22
az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name aro-vnet \
  --name master-subnet \
  --address-prefixes 10.0.0.0/23
az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name aro-vnet \
  --name worker-subnet \
  --address-prefixes 10.0.2.0/23
```

Create the cluster :
```bash
az aro create --resource-group kmm-clust_group --name kmm-cluster --vnet aro-vnet --master-subnet master-subnet --worker-subnet worker-subnet --pull-secret pull-secret.txt
```
It takes about 30/35 min. to finish. Once the cluster is created you’ll be shown a JSON output with the cluster details. OCP version installed by default is 4.10.54 as of May 16,2023 but you can set  - -version  in the azo aro create command to stick to a specific version.

You can get the web console URL by running:
```bash
az aro show     --name kmm-cluster     --resource-group $RESOURCEGROUP     --query "consoleProfile.url" -o tsv
```

You can get kubeadmin initial credentials by running:
```bash
az aro list-credentials   --name kmm-cluster   --resource-group $RESOURCEGROUP
```

The default cluster is made by 3 master nodes and 3 worker nodes.


As I forgot to set the desired version from the beginning I had to upgrade the cluster afterwards all the way to 4.12 from 4.10 in order to get KMM installed from OperatorHub so this was the pathway:

Upgraded to default 4.10.54 to 4.11 from OCP UI.

Once upgraded to 4.11.39 changed channel to stable-4.12 but a warning about admin need to ACK changes showed up and had to unblock 4.12 upgrade as some apis seem to have disappeared in 4.12. You have to be sure that you’re not using those old APIS then:
```bash
oc -n openshift-config patch cm admin-acks --patch '{"data":{"ack-4.11-kube-1.25-api-removals-in-4.12":"true"}}' --type=merge
```
Upgraded to 4.12.15

