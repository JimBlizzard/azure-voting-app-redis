---
page_type: sample
languages:
  - python
products:
  - azure
  - azure-redis-cache
description: "This sample creates a multi-container application in an Azure Kubernetes Service (AKS) cluster."
---

# Azure Voting App

This sample creates a multi-container application in an Azure Kubernetes Service (AKS) cluster. The application interface has been built using Python / Flask. The data component is using Redis.

To walk through a quick deployment of this application, see the AKS [quick start](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough?WT.mc_id=none-github-nepeters).

To walk through a complete experience where this code is packaged into container images, uploaded to Azure Container Registry, and then run in and AKS cluster, see the [AKS tutorials](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app?WT.mc_id=none-github-nepeters).

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## My notes from tutorial

### kubectl cheat sheet <https://kubernetes.io/docs/reference/kubectl/cheatsheet/>

### Best practices for azure container registry <https://docs.microsoft.com/en-us/azure/container-registry/container-registry-best-practices>

### Work with container registry using Azure CLI <https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli>

## Prepare app

From: <https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app>

## Push images to ACR

From: https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr

* sub msft
* rg  aks01rg
* acr blizzaksdemo01

### Create an Azure container registry ACR

    az group create -n aks01rg -l eastus

    az acr create -g aks01rg -n blizzaksdemo01 --sku Basic

### Log in to the container registry

    az acr login -n blizzaksdemo01

### Tag a container image (on local machine)

    docker images

    az acr list -g aks01rg --query "[].{acrLoginServer:loginServer}" --output table

    docker tag azure-vote-front blizzaksdemo01.azurecr.io/azure-vote-front:v1

### Push images to registry

This takes a few minutes

    docker push blizzaksdemo01.azurecr.io/azure-vote-front:v1

### List images in registry

    az acr repository list -n blizzaksdemo01 -o table

See tags for a specific image

    az acr repository show-tags -n blizzaksdemo01 --repository azure-vote-front -o table

## Deploy an AKS cluster

From: <https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster>

### Create a service principal for resource interactions

    az ad sp create-for-rbac --skip-assignment

Can't show the stuff that was output

#### Configure ACR auth

Get the ACR resource ID

    az acr show -g aks01rg -n blizzaksdemo01 --query "id" -o table

Grant correct access for the aks cluster to pull images stored in ACR

    az role assignment create --assignee <appID> --scope <acrID> --role acrpull

#### Deploy a k8s aks cluster

Note: To ensure your cluster operates reliably, you should run at least 2 nodes.

    az aks create -g myResourceGroup -n myAKSCluster --node-count 2 --service-principal <appId> --client-secret <password> --generate-ssh-keys

This command takes several minutes. While it runs, it simply displays "Running..." Be patient. If you get impatient and Ctrl+C the command, it contines to run in the background in Azure. You can check to see if it finished by running

    az aks list

Be patient.

### Install the k8s cli (kubectl)

    az aks install-cli

### Configure kubectl to connect to your aks cluster

    az aks get-credentials -g aks01rg --name blizzaksdemo01

Verify the connection

    kubectl get nodes

## Deploy and run apps in aks

From: <https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-application>

### Update the aks manifest file to point to the ACR login server

Get the acr login server name

    az acr list -g aks01rg --query "[].{acrLoginServer:loginServer}" --output table

    Note: the string passed to the query is case sensitive

Edit the azure-vote-all-in-one-redis.yaml file. Line 51, use your acr login server name so it looks similar to this:

    image: blizzaksdemo01.azurecr.io/azure-vote-front:v1

### Redeploy and run the app in k8s

    kubectl apply -f azure-vote-all-in-one-redis.yaml

### Test the app

To monitor the deployment progress, use kubectl get service. This will also show you the external IP for your cluster.

    kubectl get service azure-vote-front --watch

When EXTERNAL-IP changes from pending to an actual public IP address, use CTRL-C to stop the kubectl watch process.

To see the app in action, open a browser to the external IP address of your service.

If the app didn't load, there might be an authorization problem with your image registry. To view the status of your containers, use the kubectl get pods command. If the container images can't be pulled, see <https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aks#access-with-kubernetes-secret>.

## Scale applications in AKS

From: <https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-scale>

### Manually scale pods that run your app

To see the number and state of pods in your cluster

    kubectl get pods

To manually change the number of pods (here we increase them to 5)

    kubectl scale --replicas=5 deployment/azure-vote-front

Here's example output. If you're quick enough, you can see containers being created.

    kubectl get pods
    NAME                                READY   STATUS              RESTARTS   AGE
    azure-vote-back-847fc9bcb9-znjs2    1/1     Running             0          9m40s
    azure-vote-front-5c568d4b49-h8lk2   1/1     Running             0          7s
    azure-vote-front-5c568d4b49-jjw28   1/1     Running             0          7s
    azure-vote-front-5c568d4b49-nmpbw   0/1     ContainerCreating   0          7s
    azure-vote-front-5c568d4b49-w5r5f   0/1     ContainerCreating   0          7s
    azure-vote-front-5c568d4b49-zqsgq   1/1     Running             0          9m40s

### Autoscale pods that run the app front-end

Kubernetes supports horizontal pod autoscaling <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>. The metrics server <https://v1-12.docs.kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/> is automatically deployed in AKS 1.10 and higher. Check the version of your AKS cluster:

    az aks show -g aks01rg --name blizzaksdemo01 --query kubernetesVersion

If your AKS closter is less than 1.10, you can clone the metrics-server GitHub repo and install the example definitions.

    git clone https://github.com/kubernetes-incubator/metrics-server.git
    kubectl create -f metrics-server/deploy/1.8+/

To use the autoscaler, all containers in your pods must have CPU requests and limits defined. Ours currently request 0.25 CPU with a limit of 0.5 CPU. They're defined in the azure-vote-front-all-in-one-redis.yaml file.

    resources:
      requests:
        cpu: 250m
      limits:
        cpu: 500m

The following will autoscale the number of pods in the *azure-vote-front* deployment. If average CPU across all pods exceeds 50% of requested usage, the autoscaler increases the pods to a max of 10 instances. A minimum of 3 is then defined for the deployment.

    kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10

To see the staus of the autoscaler, use

    kubectl get hpa 

We currently have 5 pods. A few minutes after running the autoscale command the number of pod replicas decreases automatically to 3. You can use kubectl get pods to see the number of pods, and unneeded pods being removed.

    kubectl get pods

### Manually scale AKS nodes

The cluster currently has one node. Here's how to manually increase the number of nodes to three. The command takes a couple of minutes to complete.

    az aks scale -g aks01rg --name blizzaksdemo01 --node-count 3

When the cluster has successfully scaled, the output is similar to this:

    "agentPoolProfiles": [
      {
        "availabilityZones": null,
        "count": 3,
        "enableAutoScaling": null,
        "maxCount": null,
        "maxPods": 110,
        "minCount": null,
        "name": "nodepool1",
        "orchestratorVersion": "1.13.10",
        "osDiskSizeGb": 100,
        "osType": "Linux",
        "provisioningState": "Succeeded",
        "type": "AvailabilitySet",
        "vmSize": "Standard_DS2_v2",
        "vnetSubnetId": null
      }

### To see the current state of the cluster

        az aks show --output json -g aks01rg -n blizzaksdemo01

## Update an app in AKS

From: <https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-app-update>

### Update an application

Make a change to the app. It's in azure-vote/azure-vote/config_file.cfg

### Update the container image

To re-create the front-end application image

    docker-compose up --build -d

### Test the app locally

Open a browser to <http://localhost:8080>

### Tag and push the image

Get the login server name

    az acr list -g aks01rg --query "[].{acrLoginServer:loginServer}" --output table

Use docker tag <https://docs.docker.com/engine/reference/commandline/tag/> to tag the image.

    docker tag azure-vote-front blizzaksdemo01.azurecr.io/azure-vote-front:v2

Use docker push <https://docs.docker.com/engine/reference/commandline/push/> to upload the image to your registry.

    docker push blizzaksdemo01.azurecr.io/azure-vote-front:v2

If you experience issues, make sure you're still logged in. Run az acr login using the name of the ACR that you created earlier. For example:

    az acr login --name blizzaksdemo01

Show the tags of the containers in the registry

    az acr repository show-tags -n blizzaksdemo01 --repository azure-vote-front -o table

### Deploy the updated app

To provide max uptime, multiple instances of the app pod must be running. Verify you have multiple instances of the application pod.

    kubectl get pods

If you don't have multiple instances, run

    kubectl scale --replicas=3 deployment/azure-vote-front

To update the applicaton, use the kubectl set <https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#set> command. Update it to v2.

    kubectl set image deployment azure-vote-front azure-vote-front=blizzaksdemo01.azurecr.io/azure-vote-front:v2

Monitor the deployment using

    kubectl get pods

Example output as pods are being terminated and created:

    NAME                                READY   STATUS              RESTARTS   AGE
    azure-vote-back-847fc9bcb9-znjs2    1/1     Running             0          79m
    azure-vote-front-5c568d4b49-h8lk2   1/1     Running             0          70m
    azure-vote-front-5c568d4b49-jjw28   0/1     Terminating         0          70m
    azure-vote-front-5c568d4b49-zqsgq   1/1     Running             0          79m
    azure-vote-front-76b5bcf9cd-7rjcl   0/1     ContainerCreating   0          9s
    azure-vote-front-76b5bcf9cd-csrhl   1/1     Running             0          9s

### Test the updated app

To view the app, first get the external IP address of the azure-vote-front service, Then open a browser window to the IP address.

    kubectl get service azure-vote-front

## Upgrade K8S in AKS

From: <https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-upgrade-cluster>

Your application will still be alive while Kubernetes is being upgraded. It's magic.

To minimize disruption to running applications, AKS nodes are carefully cordoned and drained. In this process, the following steps are performed:

* The Kubernetes scheduler prevents additional pods being scheduled on a node that is to be upgraded.
* Running pods on the node are scheduled on other nodes in the cluster.
* A node is created that runs the latest Kubernetes components.
* When the new node is ready and joined to the cluster, the Kubernetes scheduler begins to run pods on it.
* The old node is deleted, and the next node in the cluster begins the cordon and drain process.

### Get available cluster versions

    az aks get-upgrades -g aks01rg --name blizzaksdemo01 

### Upgrade a cluster

Note: you can only upgrade one minor version at a time, such as 1.13 to 1.14, not to 1.15. An upgrade will take several minutes. While this command is running your application will still be available.

    az aks upgrade -g aks01rg --name blizzaksdemo01 --kubernetes-version 1.14.6

### Validate an upgrade

    az aks show -g aks01rg --name blizzaksdemo01 --output table

### Delete the cluster

    az group delete --name aks01rg --yes --no-wait

When you delete the cluster, the Azure Active Directory service principal used by the AKS cluster is not removed. For steps on how to remove the service principal, see <https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal#additional-considerations>

To delete the service principal, query for your cluster servicePrincipalProfile.clientId and then delete with az ad app delete. Replace the following resource group and cluster names with your own values

    az ad sp delete --id $(az aks show -g aks01rg -n blizzaksdemo01 --query servicePrincipalProfile.clientId -o tsv)
