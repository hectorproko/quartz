---
tags:
  - azure
title: Integrating Azure Container Registry with Docker and Deploying Images via Azure App Service
---
<!--
**Project Insight: Integrating Azure Container Registry with App Service!** In a recent assignment from my Azure Administrator course, I successfully demonstrated the integration of Azure Container Registry (ACR) with Azure App Service. The task involved creating an Azure Container Registry, connecting it to Docker in a VM, uploading a Docker image to ACR, and then deploying this image using Azure App Service. This comprehensive exercise provided practical experience in managing containerized applications in Azure, highlighting the seamless workflow from image creation to deployment. It was a valuable opportunity to deepen my understanding of Azure's container services and their application in a real-world scenario.

#Azure #ContainerRegistry #AppService #Docker #CloudComputing #AzureAdministrator #Containerization #ProfessionalDevelopment
-->


> [!info] Module 5: Assignment - 2
> **Tasks To Be Performed:** 
> 1. Create an Azure Container Registry and connect it to Docker running in VM 
> 2. Upload the image you created in this Azure to container registry 
> 3. Create an app service to deploy the same image 
> 



Using image from [[Assignment 1_Module5_Azure Administrator Course for AZ-103 AZ-104|Assignment 1:Module 5]]

<br>![[Pasted image 20231210212722.png]]


Installing Azure CLI:
```bash
sudo apt install azure-cli
```
<br>![[Pasted image 20231212124827.png]]

### Step 1: Create an Azure Container Registry

1. **I Open the Azure Portal**:

2. **I Create the Container Registry**:
    
    - I search for "Container registries" and click "+Create".
    - I fill in the required details, such as the registry name, subscription, resource group, and location.
    - I choose the pricing tier Standard
    - I review and create the container registry.
      <br>![[Pasted image 20231210214517.png]]


### Step 2: Connect the Registry to Docker in the VM

hectorregistry
hectorregistry.azurecr.io

1. **I Log into the VM**:
    
    - I SSH into my VM where Docker is installed.
2. **I Log into the Azure Container Registry from the VM**:
    
    - I use the Azure to authenticate to the ACR:
      `az login`
        
        
        <br>![[Pasted image 20231210220508.png]]
        
        <br>![[Pasted image 20231210220743.png]]
        <br>![[Pasted image 20231210220813.png]]
        <br>![[Pasted image 20231212120031.png]]
        
        


---

### Step 3: Upload the Image to Azure Container Registry

1. **I Tag the Docker Image**:
    
    - Before pushing the image to ACR, I need to tag it with the ACR login server name:
      <br>![[Pasted image 20231212120337.png]]
      
```bash
docker tag mynewimage:latest <registry_name>.azurecr.io/mynewimage:latest
```

```bash
docker tag hshar/webapp_updated:latest hectorregistry.azurecr.io/webapp_updated:latest
```
<br>![[Pasted image 20231212120520.png]]

2. **I Push the Image to ACR**:
- Login to the container registry
```
sudo az acr login --name hectorregistry
```
<br>![[Pasted image 20231212125808.png]]

- I use the Docker CLI to push the image to the ACR:
```bash
sudo docker push hectorregistry.azurecr.io/webapp_updated:latest
```
<br>![[Pasted image 20231212130333.png]]

Back in the portal I verity the image was uploaded
<br>![[Pasted image 20231212130744.png]]


### Step 4: Create an App Service to Deploy the Image

1. **I Open the Azure Portal Again**:
    
    - I navigate back to the Azure Portal.
2. **I Create a New App Service**:
    
    - I search for "App Services".
    - I click "+ Create" and select "+ Web App".
      <br>![[Pasted image 20231213115310.png]]

    - I fill in the details such as the app name, subscription, resource group, and plan.
    - make sure select right OS Linux and for "Publish" select "Docker Container"
      <br>![[create web app.png]]
      
    - Under "Docker," I select the "Single Container" option.
    - I choose Azure Container Registry as the source and select the image I pushed earlier.
      <br>![[Pasted image 20231213120821.png]]
3. **I Review and Create the Web App**:
    
    - I review all the settings and click "Create" to deploy the web app.
      <br>![[Pasted image 20231213121046.png]]
4. **I Verify the Deployment**:
    - I "Got to resource"
    - Once the app is deployed, I navigate to the URL provided by the App Service.
      <br>![[Pasted image 20231213121214.png]]
    - I should see the application running, served by the Docker image deployed from ACR.
      <br>![[Pasted image 20231213123114.png]]




