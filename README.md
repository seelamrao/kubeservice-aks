# Configuring Azure Kubernetes Service (AKS) Cluster and running/scaling a .Net Core application  

<!-- TOC -->

- [Development Environment Setup](#development-environment-setup)
- [Containerize the application](#containerize-the-application)
- [Deploy the application to local Kubernetes Cluster](#deploy-the-application-to-local-Kubernetes-Cluster)
- [Push the image to Azure Container Registry using the Azure CLI](#push-the-image-to-azure-container-registry-using-the-azure-cli)
- [Create AKS cluster using the Azure CLI](#create-aks-cluster-using-the-azure-cli)
- [Deploy the application to AKS Cluster using .yml file](#deploy-the-application-to-aks-cluster-using-yml-file)
- [Scale the application and Kubernetes infrastructure](#scale-the-application-and-kubernetes-infrastructure)
	- [Scaling Azure Kubernetes Service (AKS) Cluster nodes](#scaling-azure-kubernetes-service-aks-cluster-nodes)
	- [Scaling Azure Kubernetes Service (AKS) Cluster pods](#scaling-azure-kubernetes-service-aks-cluster-pods)
- [Update the application then push the image to ACR and update deployment](#update-the-application-then-push-the-image-to-acr-and-update-deployment)

<!-- /TOC -->

### Development Environment Setup
- Install Docker
- Enable Kubernetes Cluster on Docker, under Docker -> Settings
- Install Azure CLI and Azure PowerShell modules

### Containerize the application
- Clone this repo to a folder
- Open a command prompt and navigate to your project folder.
- Use the following commands to build and run your Docker image:
	```
	docker build . -t kubeservice:local
	docker run -d -p 8000:80 kubeservice:local
	```
- View the .Net Core application running from a container by navigating to  [localhost:8000](http://localhost:8000/)

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/kubeservice-local-docker.png)

### Deploy the application to local Kubernetes Cluster
- Deploy docker image to Kubernetes
	```
	kubectl run kubeservice-deployment --image=kubeservice:local --port=80 --replicas=3
	```
	- ```kubectl``` command
		- to verify the deployment:  ```kubectl get deployments```
		- to verify the replicasets: ```kubectl get rs```
		- to verify the Pods: ```kubectl get pods```

- Expose the application to outside world, use the following command
	```
	kubectl expose deployment kubeservice-deployment --type=NodePort
	```
- Verify the service
	```
	kubectl get svc
	```
	```
	Expected Result:
	NAME                     TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
	kubernetes               ClusterIP      10.0.0.1      <none>           443/TCP        42d
	kubeservice-deployment   NodePort       10.0.120.27   <none>           80:31057/TCP   1m
	```
- Finally access the app using browser: http://localhost:31057

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/kubeservice-local-kubernetes.png)

### Push the image to Azure Container Registry using the Azure CLI
- Login to Azure
	```
	az login
	```
- Create Azure Resource Group
	```
	az group create -n vvkubeserviceacr-rg -l eastus
	```
- Create Azure Container Registry
	```
	az acr create -n vvkubeserviceacr -g vvkubeserviceacr-rg -l eastus --sku standard
	```
	- Verify the Azure Container Registry
		```
		az acr list -o table
		Expected Result:
		NAME              RESOURCE GROUP       LOCATION    SKU       LOGIN SERVER                 CREATION DATE         ADMIN ENABLED
		vvkubeserviceacr  vvkubeserviceacr-rg  eastus      Standard  vvkubeserviceacr.azurecr.io  2018-10-27T20:04:22Z
		```

- Push the image to Azure Container Registry

	- Login to Azure Container Registry
		```
		az acr login -n vvkubeserviceacr
		```
	- Tag the docker image prior to push the image to Azure Container Registry
		```
		docker tag kubeservice:local vvkubeserviceacr.azurecr.io/kubeservice:v1
		```
	- Push the tagged image to Azure Container Registry
		```
		docker push vvkubeserviceacr.azurecr.io/kubeservice:v1
		```
	- Verify the repository in Azure Container Registry
		```
		az acr repository list -n vvkubeserviceacr -o table
		Result:
		kubeservice
		```
![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/azure-container-registry.png)

### Create AKS cluster using the Azure CLI
AKS uses the Service Principal to access other Azure Services like the Azure Container Registry (ACR).  It is recommended to register an application in Azure AD and create an identity for the application (i.e. Service Principal)

- Create Service Principal
	```
	az ad sp create-for-rbac --skip-assignment

	Result:
	{
        "appId": "<appId>",
        "displayName": "<displayName>",
        "name": "<name>",
        "password": "<password>",
        "tenant": "<tenant>"
    }
	```
- Create acrID variable
	```
	$acrID = az acr show --name vvkubeserviceacr --resource-group vvkubeserviceacr-rg --query "id" --output tsv
	```
- Create a role assignment with appid and acrID
	```
	az role assignment create --assignee "<appId>" --role Reader --scope $acrID
	```
- Create AKS Cluster
	```
	az aks create --name vvkubeserviceaks --resource-group vvkubeserviceacr-rg --node-count 1 --generate-ssh-keys --service-principal "<appId>" --client-secret "<password>" --location eastus
	```
- Get the AKS cluster credentials to update local environment
   ```
   az aks get-credentials --name vvkubeserviceaks --resource-group vvkubeserviceacr-rg
   ```

- Verify the AKS Cluster in Azure Cloud
	```
	kubectl get nodes

	Expected Result look like this:
	NAME                       STATUS    ROLES     AGE       VERSION
	aks-nodepool1-33154602-0   Ready     agent     6m        v1.9.11
	```

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/azure-aks-cluster.png)

### Deploy the application to AKS Cluster using .yml file
- Below is the content of `kubeservice-deploy.yml` file:
	```
	apiVersion: apps/v1
	kind: Deployment
	metadata:
		name: kubeservice-deployment
		labels:
		    app: kubeservice
	spec:
		replicas: 3
		template:
			metadata:
				name: kubeservice
				labels:
					app: kubeservice
			spec:
				containers:
			      - name: kubeservice
			         image: vvkubeserviceacr.azurecr.io/kubeservice:v1
			         imagePullPolicy: IfNotPresent
			    restartPolicy: Always
		selector:
		    matchLabels:
			      app: kubeservice
	---
	apiVersion: v1
	kind: Service
	metadata:
		 name: kubeservice-service
	spec:
	  selector:
	    app: kubeservice
	  ports:
	    - port: 80
	  type: LoadBalancer
- Deploy the application to AKS Cluster
	```
	kubectl apply -f .\kubeservice-deploy.yml
	```
- To monitor the service deployment
	```
	kubectl get service --watch

	Result:
	NAME                  TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
	kubernetes            ClusterIP      10.0.0.1      <none>        443/TCP        20m
	kubeservice-service   LoadBalancer   10.0.154.43   <pending>     80:30845/TCP   15s
	kubeservice-service   LoadBalancer   10.0.154.43   137.135.124.83   80:30845/TCP   1m
	```

### Scale the application and Kubernetes infrastructure

#### Scaling Azure Kubernetes Service (AKS) Cluster nodes
- Get the current nodes
	```
	kubectl get nodes

	Result:
	NAME                       STATUS    ROLES     AGE       VERSION
	aks-nodepool1-33154602-0   Ready     agent     26m       v1.9.11
	```
- Scaling nodes
	```
	az aks scale --resource-group vvkubeserviceacr-rg --name=vvkubeserviceaks --node-count 3

	Result:
	<json with success message>
	```
- Get the current nodes after scaling
	```
	kubectl get nodes

	Result:
	NAME                                      READY     STATUS    RESTARTS   AGE
	kubeservice-deployment-5fbf677667-ffftl   1/1       Running   0          17m
	kubeservice-deployment-5fbf677667-kvfg2   1/1       Running   0          17m
	kubeservice-deployment-5fbf677667-mjgbq   1/1       Running   0          17m
	```

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/kubeservice-aks-Instance-1.png)

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/kubeservice-aks-Instance-2.png)

#### Scaling Azure Kubernetes Service (AKS) Cluster pods
- Verify the current replicas # of application
	```
	kubectl get pods

	Result:
	NAME                                      READY     STATUS    RESTARTS   AGE
	kubeservice-deployment-5fbf677667-ffftl   1/1       Running   0          18m
	kubeservice-deployment-5fbf677667-kvfg2   1/1       Running   0          18m
	kubeservice-deployment-5fbf677667-mjgbq   1/1       Running   0          18m
	```
- Verify the current deployment
	```
	kubectl get deployment

	Result:
	NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	kubeservice-deployment   3         3         3            3           19m
	```
- Scale the pods to 5
	```
	kubectl scale --replicas=5 deployment/kubeservice-deployment

	Result:
	deployment.extensions "kubeservice-deployment" scaled
	```
- Verify the current deployment after scale
	```
	kubectl get deployment

	Result:
	NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	kubeservice-deployment   5         5         5            5           22m
	```
- Verify the current replicas # of application
	```
	kubectl get pods

	Result:
	NAME                                      READY     STATUS    RESTARTS   AGE
	kubeservice-deployment-5fbf677667-b6p8l   1/1       Running   0          2m
	kubeservice-deployment-5fbf677667-ffftl   1/1       Running   0          22m
	kubeservice-deployment-5fbf677667-kvfg2   1/1       Running   0          22m
	kubeservice-deployment-5fbf677667-mjgbq   1/1       Running   0          22m
	kubeservice-deployment-5fbf677667-qdrsf   1/1       Running   0          2m
	```

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/kubeservice-aks-after-scale.png)

### Update the application then push the image to ACR and update deployment

- Change the application
	```
	Change the label "Host Name:" with "AKS Host Name:"
	```
- Create a new image
		```
		docker build . -t vvkubeserviceacr.azurecr.io/kubeservice:v2
		```
- Push the image to Azure Container Registry
	```
	docker push vvkubeserviceacr.azurecr.io/kubeservice:v2
	```
- Update the deployment
	```
	kubectl set image deployment kubeservice-deployment kubeservice=vvkubeserviceacr.azurecr.io/kubeservice:v2
	```

-  To monitor the deployment
	```
	kubectl get pods

	Result:
	NAME                                    READY     STATUS    RESTARTS   AGE
	kubeservice-deployment-9c594f98-5bwgp   1/1       Running   0          1m
	kubeservice-deployment-9c594f98-b4szp   1/1       Running   0          1m
	kubeservice-deployment-9c594f98-ctbzp   1/1       Running   0          1m
	kubeservice-deployment-9c594f98-pjztm   1/1       Running   0          1m
	kubeservice-deployment-9c594f98-w75g4   1/1       Running   0          1m
	```
- Test the updated application
	```
	kubectl get service

	Result:
	NAME                  TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
	kubernetes            ClusterIP      10.0.0.1      <none>           443/TCP        1h
	kubeservice-service   LoadBalancer   10.0.154.43   137.135.124.83   80:30845/TCP   1h
	```
- Run the application to see the changes made by navigating to the url: http://137.135.124.83

![alt text](https://github.com/venkataveera/kubeservice/blob/master/content/kubeservice-aks-after-app-change.png)
