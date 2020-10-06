# Step by Step MOVECLOUD-T002

In this task you will run the WEB and API application on the cluster, while it connects with the CosmosDB. The INIT container, that was pushed to the registry as well, can be used to populate the CosmosDB. You will use the kubectl commands and YAML files to deploy them.

>This task has a Starter solution, that creates a Pull Request containing some files and instructions. 
>
> In order to run the automated Starter Solution, you need to go through the [Setup prerequisites](/Challenges/Prequisites/RunThroughSetup.md) first!

1. In your GitHub Codespace, open a PowerShell Terminal and run the starter solution. A Pull Request with 2 YAML files will be created

```
.workshop/workshop-step.ps1  Start "MOVECLOUD-T002"
```

2. In your GitHub repository, navigate to the Tab Pull Requests and open the Pull Request with DEVWF-T004 in the title

![Shows the menu item for navigating to the Pull Request](pullrequestmovecloudt002.png)

3. In the Pull Request, check the conversation, Commits, Checks and Files Changed Tabs, and got through the instructions and changes.

4. On the Conversation Tab, press the Merge Pull Request Button, to merge the files in to the main branch. Link the Pull Request to your Azure Boards Work item for Module 1 by typing AB#Module1WorkItemID in the title or description of the Pull Request Commit Message. 

![Shows the button for merging a Pull Request in GitHub](images/mergePullRequest.png)

Now your repository contains 3 new "multi-staged" docker file.

6. In your GitHub Codespace, update your files to the latest version by pulling them.

![](images/2020-10-05-12-10-11.png)

7. To be able to pull a container from the GitHub Container Registry in to the AKS cluster, you need to configure a pull secret in AKS. You can do this by running the kubectl create secret command. Retrieve the GitHub Personal Access Token, that you also used in [DEVWF-T007](/Challenges/Module1-ImprovingDeveloperFlow/Tasks/DEVWF-T007.md). In your PowerShell terminal create a variable called $ghToken and use this token as value

```
$ghToken = "9ad18...xxxx"
```

8. Create a new Kubernetes secret called `pullsecret` that uses this variable to pull from your GitHub Container Registry. Execute this command in your terminal window

```
kubectl create secret docker-registry pullsecret --docker-server=https://ghcr.io/ --docker-username=notneeded --docker-password=$ghToken
```

9. To be able to access the CosmosDB, you need to add the connectionstring as a Kubernetes Secret. Retrieve the connectionstring to the CosmosDB in the portal or use the following command and use the Primary MongoDB ConnectionString.

```
az cosmosdb keys list -n $cosmosDBName -g $resourceGroupName --type connection-strings
```

10. Add the contentdb database as part of the connectionstring and add it as as a Kubernetes secret. `....documents.azure.com:10255/contentdb?ssl=true`

```
$mongodbConnectionString="connectionString=mongodb://xxx.documents.azure.com:10255/contentdb?ssl=true&replicaSet=globaldb"
kubectl create secret generic cosmosdb --from-literal=db=$mongodbConnectionString
```

11. Now that both secrets are added, modify the api-deploy.yml file. Update the following snippet to retrieve your own image

```YAML
      spec:
        containers:
        - image: ghcr.io/<yourgithubaccount>/fabrikam-api:latest 
```

12. Deploy the API Deployment and Service to Kubernetes

```
kubectl apply -f api-deploy.yml
kubectl apply -f api-service.yml
```

13. Login to the Kubernetes Dashboard to see if the pods successfully deployed. Use the command `az aks browse --name $aksName --resource-group $resourcegroupName` to open the Dashboard.

![](k8sDashboard.png)

14. In the AKS folder, create a file called web-deploy.yml. Fill the file with the following content. Update <yourgithubaccount> with your GitHub account name.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
      app: web
  name: web
spec:
  replicas: 1
  selector:
      matchLabels:
        app: web
  strategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 1
      type: RollingUpdate
  template:
      metadata:
        labels:
            app: web
        name: web
      spec:
        containers:
        - image: ghcr.io/<yourgithubaccount>/fabrikam-web:latest 
          env:
            - name: CONTENT_API_URL
              value: http://api:3001
          livenessProbe:
            httpGet:
                path: /
                port: 3000
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 10
            failureThreshold: 3
          imagePullPolicy: Always
          name: web
          ports:
            - containerPort: 3000
              protocol: TCP
          resources:
            requests:
                cpu: 125m
                memory: 128Mi
          securityContext:
            privileged: false
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
        imagePullSecrets:
        - name: pullsecret  
```

15. 14. In the AKS folder, create a file called web-service.yml. Fill the file with the following content

```YAML
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: web
spec:
  ports:
    - name: web-traffic
      port: 80
      protocol: TCP
      targetPort: 3000
  selector:
    app: web
  sessionAffinity: None
  type: LoadBalancer
```

16. Deploy the Web deployment and web service with the following command

```
kubectl apply -f web-deploy.yml
kubectl apply -f web-service.yml
```

17. In your Kubernetes Dashboard, navigate to the Services Tab. You will find the Web Service with an external IP. Click the IP address and find that the website shows up, without any data.

![](2020-10-06-14-07-53.png)

18. Add a file called init-deploy.yml to your AKS folder, and set the contents to the snippet below. Update <yourgithubaccount> with your GitHub account name.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
      app: init
  name: init
spec:
  replicas: 1
  selector:
      matchLabels:
        app: init
  strategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 1
      type: RollingUpdate
  template:
      metadata:
        labels:
            app: init
        name: init
      spec:
        containers:
        - image: ghcr.io/<yourgithubaccount>/fabrikam-init:latest 
          env:
            - name: MONGODB_CONNECTION
              valueFrom:
                secretKeyRef:
                  name: cosmosdb
                  key: db              
          livenessProbe:
            httpGet:
                path: /
                port: 3000
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 10
            failureThreshold: 3
          imagePullPolicy: Always
          name: init
          ports:
            - containerPort: 3000
              hostPort: 80
              protocol: TCP
          resources:
            requests:
                cpu: 125m
                memory: 128Mi
          securityContext:
            privileged: false
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
        imagePullSecrets:
        - name: pullsecret  
```
17. In your Kubernetes Dashboard, navigate to the Services Tab. You will find the Web Service with an external IP. Click the IP address and find that the website shows up, but now the Speaker and Session pages show data.

> When you do not want to type all commands try the solution Pull Request by running
```
.workshop/workshop-step.ps1  Solution "MOVECLOUD-T002"
```