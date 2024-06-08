
# Frappe ERPNext APP
This project showcases the deployment of the Frappe Framework along with the ERPNext and HR applications using Docker and Kubernetes. This project focuses on building a custom Docker image for the ERPNext and HR applications for Frappe and deploying them within a Minikube Kubernetes cluster utilizing Helm for orchestration and management.

## Table of Contents

- [Docker Image](#docker-image)
- [Docker Compose ](#docker-compose)
- [Helm Deployment](#helm-deployment)
   - [Values File Modification](#values-file-modification)
   - [job-create-site Template Modification](#job-create-site-template-changes)
   - [Run the APP](#running-the-app)
   - [Access the app](#access-the-app)
- [Argo CD](#argo-cd)


1. ## Docker Image
   
    - Create apps.jason file which holds repository links to your Frappe applications`Docker/apps.json`
    ```
    [
        {
        "url": "https://github.com/frappe/erpnext",
        "branch": "version-15"
        },
    
        {
        "url": "https://github.com/frappe/hrms",
        "branch": "version-15"
        }
    ]
    ```
    - Generate base64 string from the apps.json file 
    ```
    export APPS_JSON_BASE64=$(base64 -w 0 apps.json)
    ```
    - Build the docker file `Docker/Dockerfile`with the specified build arguments to customize the build process.
    ```
    docker build \
    --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
    --build-arg=FRAPPE_BRANCH=version-15 \
    --build-arg=PYTHON_VERSION=3.11.6 \
    --build-arg=NODE_VERSION=18.18.2 \
    --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
    --tag=custom-hrms \
    --file=Docker/Dockerfile .
    ```
    - Push image to DockerHub 
    ```
    #change image tag
    docker tag custom-hrms omartarekabdelall/frappe:hrms

    docker login 
    docker push
    ```
   Through these steps, you have  customized Frappe ERPNext images to include the desired HRMS app and prepared them for Kubernetes deployment by pushing the images to DockerHub.

2. ## Docker Compose 
    You can have a quick set up and run to the entire application stack to view the app components and interact with it. Ensure you have Docker and Docker Compose installed on your machine.

    ### docker-compose file
    - Used the built docker image `omartarekabdelall/frappe:hrms` 
    - Add the following command after site creation in order to install the HRMS APP to the bench site and dashbourd.
      ```
       bench --site frontend install-app hrms;
      ``` 

    ### Access the frappe app
    you can access the app through  frontend site port mapping to the local host.\
    Navigate to `http://localhost:8080`


3. ## Helm Deployment 
    This section outlines the setup and modification to the Helm deployment and provides guidance on how to verify and access the application within a local Minikube cluster.
    ### Clone the Frappe helm charts
    ```
    git clone https://github.com/frappe/helm.git
    ```
    ### Values File Modification
    - Use the Built Image
        ```
        image:
          repository: omartarekabdelall/frappe 
          tag: hrms 
          pullPolicy: IfNotPresent
        ```
    - Ingress Modifications 
        - Change the ingress Class field to the name of your Ingress controller class
        - enable the ingress svc by changing `enabled` field to `true`
        - modify the host entery to the desired domain name for reaching your Frappe app
        ```
        ingress:
            ingressName: "erpnext-ingress"
            className: "kong"
            enabled: true
            annotations: {}
            hosts:
            - host: erp.cluster.local
                paths:
                - path: /
                pathType: Prefix
        ```
        
    - enable the `createSite` job by changing `enabled` field to `true`

    ### job-create-site Template Modification
    - Add the following command after site creation in order to install the HRMS APP to the bench site and dashbourd.\
     Refer to `helm/erpnext/templates/job-create-site.yaml` line 72.
         ```
         bench --site erp.cluster.local install-app hrms
         ```
     
    - You can also add the following commands  if you have an assest rendering issue.
        ```
        bench build
        bench --site erp.cluster.local clear-website-cache
        ```


    ### Run the APP 
    - Deploy an NFS server provisioner within a Kubernetes cluster, enabling persistent storage via NFS
        ```
        kubectl create namespace nfs

        helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner

        helm upgrade --install -n nfs in-cluster nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner --set 'storageClass.mountOptions={vers=4.1}' --set persistence.enabled=true --set persistence.size=8Gi
        ```
    - Install the Helm Deployment
        ```
        cd /helm

        helm install frappe-bench ./erpnext -n erpnext --set persistence.worker.storageClass=nfs
        ```
    - App verification
        - Check the `frappe-bench-erpnext-new-site` pod logs to observe the progress of site creation and the installation of ERPNext and HRMS applications.
            ```
            kubectl logs <frappe-bench-erpnext-new-site-podName> -n erpnext
            ```
            ![create site logs](https://github.com/Omar-tarek3/Assets/blob/master/frappe-app/create-site.png)

        - Verify the creation of the desired sites and apps by executing commands to the `frappe-bench-erpnext-worker-d` pod and navigating to the follwing files:
            ```
            kubectl exec -it <frappe-bench-erpnext-worker-d-podName>  -n erpnext -- sh

            #view the created erp.cluster.local site
            ls sites 

            #view installed apps (farppe, erpnext, hrms)
            vim /sites/apps.txt 
            ```

            ![sites verify](https://github.com/Omar-tarek3/Assets/blob/master/frappe-app/bench%20apps.png)

    ### Access the app
    1. ##### Setup DNS Mapping:
        - I added an entery at `C:\Windows\System32\drivers\etc\hosts` file to map the site name configured in ingress resouce rule `erp.cluster.local` to the IP `127.0.0.1` where the Ingress controller runs.

            ```
            #C:\Windows\System32\drivers\etc\hosts entery
            127.0.0.1 erp.cluster.local
            ``` 
    -  Run the following command to forward port 8000 on your local machine to the port used by the LoadBalancer Ingress service:
        ```
        kubectl port-forward svc/my-kong-kong-proxy 8000:443
        ```
    - This command will make the Ingress controller accessible at `https://localhost:8000`

    - Navigate to `https://erp.cluster.local` in order to hit the endpoint for the frappe app

    ![frappe app](https://github.com/Omar-tarek3/Assets/blob/master/frappe-app/frappe-app.png)


4. ## Argo CD
   This section provides an overview of setting up and configuring Argo.
   
   - Create `argocd` namespace and apply the Argo CD installation manifest:
     ```
      kubectl create namespace argocd

      kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
      ```
   - Access the Argo  UI:
      -  Forward the Argo server port to your local machine.
      
      ```
      kubectl port-forward -n argocd svc/argocd-server 8083:443
      ```
      - Navigate to `http://localhost:8083` to access the Argo UI.
   
   - Configure an application manifest to hold your Argo configuration and apply this manifest to your local Kubernetes cluster:
   
     ```
     kubectl apply -f ./app.yaml
     ```
     Refer to `app.yaml` to view setup and deployment configuration for the app with Argo

    ![Argo](https://github.com/Omar-tarek3/Assets/blob/master/frappe-app/Screenshot%202024-06-08%20164918.png)

