# Sample Calculator application

This little project has some sample codes and instructions (vsphere) to get you started with
- Creating a Tanzu Kubernetes Grid cluster
- Prepare the cluster with necessary components (eg: ingress, psp, pvc etc) to deploy a workload
- Allowing Tanzu k8s Cluster to trust a self-signed certificated container registry [Read More Here]()
- deploy a sample application (microservice based calculator) on the Tanzu k8s cluster.

Use the docker compose file for local development of micro-services (various math functionalities are destructred into micro services).

Below are the micro services code repositories:
- [Frontend UI (javascript ReactJS)](https://github.com/alinahid477/vmw-calculator-frontend)
- [Backend Add Service (java Spring Boot)](https://github.com/alinahid477/vmw-calculator-addservice)
- [Backend Subtract Service (java Spring Boot)](https://github.com/alinahid477/vmw-calculator-subtractservice)
- [Backend Division Service (C# dotnet core)](https://github.com/alinahid477/vmw-calculator-divisionservice)
- [Backend Multiply Service (javascript nodejs)](https://github.com/alinahid477/vmw-calculator-multiplyservice)

And some handy components like:
- [Sample Jenkins pipeline](https://github.com/alinahid477/sample-jenkins-pipelines)
- [Local Development using NginX](https://github.com/alinahid477/vmw-calculator-devops)


STEP 0: Create vSphere "Namespace"
=======================================
To avoid confusion, this namespace is different to Kubernets namespace. This "vSphere Namespace" is
- a resource pool for k8s cluster that will be created under.
- grouping for all the clusters that will be created for a purpose (eg: Dev, UAT, Prod etc).

For example: in a vSphere SDDC of a certain organisation can have namespaces by name of application.
Namespace name: My Calculator App (which will most likely be created by OPS team) and in that namespace DevOps or Developer can create clusters like "dev", "uat", "prod".

OR 

it can be created in another way as well like:
vSphere admin can create the below "vSphere namespace for kube cluster" namespaces:
- vSphere Namespace: dev
- vSphere Namespace: uat
- vSphere Namespace: prod
And then the DevOps or Developer can crate kubernetes cluster in those namespaces like:
- Kubernetes cluster name: calculator-app (in Dev)
- Kubernetes cluster name: calculator-app (in uat)  
- Kubernetes cluster name: calculator-app (in prod)

Which ever works/suits best for the purpose.

Think of this "vSphere namespace" as a logical grouping of Kubernetes clusters through which vSphere Admin or the OPS team can
- assign users to the namespace, 
- resource boundary parameters to the underlying VMs of kubernetes Pods, 
- apply Storage through storage policy set in vSphere etc.


##### Create namespace in vSphere:
- Go to Menu > Workload management > create namespace
- Assign users.
- Assign storage by selecting the right available storage policy (In kubernetes world this is like creating a Persistent Volume. We will this this PV later on our Kubernetes cluster through Persistent volume claim aka PVC)
- Optionally the OPS team can also add/assign governance parameters such as what type compute resource will be created for the kubernetes cluster in the vSphere namespace (eg> CPU size, memory limit etc).




STEP 1: Prepare workstation for Kubernetes and TKG
======================================================

- in vSphere client navigate to Menu > Workload Management > select the right vSphere Namespace (in this case it was named calc). This should bring up the summary console. 

***UPDATE: If you are using ssh tunnel through the docker container as described here: https://github.com/alinahid477/VMW/tree/main/calcgithub THEN place the downloaded kubectl and kubectl-vsphere file in the kubectl directory and skip the below on setting up local machine.***

OR

**You can use [vsphere with tanzu wizard](https://github.com/alinahid477/vsphere-with-tanzu-wizard) to interact with k8s cluster (both supervisor and workload)**

- Download kubectl and kubectl-vShpere and save it to appropreate location of your workstation. 
    - I was running debian linux distro (Mint 20.1 - Ulyssa).  
    - I saved it in ~/programs/vsphere/bin
    - Then appended this line in my ~/.bashrc "export PATH=$PATH:/home/ali/programs/vsphere/bin" so that I can run kubectl from anywhere. (this is adding binaries to path).
- Also appended "export KUBECTL_VSPHERE_PASSWORD=mysecretpassword" in my ~/.bashrc (this is vSphere user password). This so that when login into the vSphere namespace or cluster the password is taken from env vars. 


STEP 2: Create kubernetes cluster
=======================================
**You can use [vsphere with tanzu wizard](https://github.com/alinahid477/vsphere-with-tanzu-wizard) to create workload cluster and skip the below**


To create cluster login into vsphere kubernetes using tkg:

`$ kubectl-vsphere login --insecure-skip-tls-verify --server sddc.private.local -u administrator@vsphere.local`

// to logout: `kubectl vsphere logout`

then:

`$ kubectl apply -f kubernetes/tanzu/calc-k8-cluster.yaml`


## file explanation:
In the yaml file notice the below section:
- apiVersion: run.tanzu.vmware.com/v1alpha1 #TKG API endpoint 
- kind: TanzuKubernetesCluster #required parameter
This is how Tanzu TKG comes into play.

Also notice the below:
- name: calc-k8-cluster --> this is the name of your cluster 
- namespace: calc  --> this is the name of the "vSphere namespace" (as explained in STEP 0).

Then rest of the items in the file is pretty self explanatory.

This takes approx 10mins to complete and kubernetes cluster to be ready. (Good time for a coffee break. Don't drink beer at this stage)

STEP 3: Login to Kubernetes cluster
=======================================

**You can use [vsphere with tanzu wizard](https://github.com/alinahid477/vsphere-with-tanzu-wizard) for automating one click login to skip the below**

***UPDATE: If you are using ssh tunnel through the docker container as described here: https://github.com/alinahid477/VMW/tree/main/calcgithub THEN skip this section as this is already done.***

- `$ kubectl vsphere login --tanzu-kubernetes-cluster-name calc-k8-cluster --server sddc.private.local --insecure-skip-tls-verify -u administrator@vsphere.local`
- `$ kubectl config use-context calc-k8-cluster`





STEP 4: Prepare kubernetes cluster for application
==================================================
Now that the cluster is created lets prepare the cluster for our app.

Below are the steps (basically, apply all yaml in kubernetes/global).

- create kubernetes namespace: 

    `$ kubectl apply -f kubernetes/global/namespace.yaml`

- pod security policy: 

    `kubectl apply -f kubernetes/global/allow-psp-clusterrole.yaml`

    **why?** 
    
    read details here: https://kubernetes.io/docs/concepts/policy/pod-security-policy/ 
    
    and 
    
    here: https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-CD033D1D-BAD2-41C4-A46F-647A560BAEAB.html

- allow Jenkins CICD to deploy workload:

    **if you do not have a CICD rool already you can use Jenkins and deploy Jenkins on k8s using [JenkinsOnK8S Wizard](https://github.com/alinahid477/jenkinsonk8s)**

    `kubectl apply -f kubernetes/global/jenkins-service-account.yaml`

    `kubectl apply -f kubernetes/global/jenkins-sa-rbac.yaml`
    
    We will use this service account in our Jenkins pipeline (with Kubernetes CLI plugin; see Jenkinsfile of any application in this project and check the Readme in Jenkins folder)

- Deploy ingress controller (CONTOUR with ENVOY; release-1.16):

    Following the quick start from contour: https://projectcontour.io/getting-started/. I took the (Option 1).
    ***However, there's a gotcha: the contour.yaml (at https://projectcontour.io/quickstart/contour.yaml) file contains a image pull from docker.io/projectcontour/contour-operator:v1.16.0 which without docker auth fails with dockerhub's rate-limit issue. So I downloaded the yaml --> separated the namespace from the giant yaml into a separate 'namespace.yaml' and the rest of the content into 'contour.yaml' --> created dockurhub regcred in that namespace ---> made a slight mods to contour.yaml to add a imagepull secret to the service account, deployment and daemonset --> finally, applied the contour.yaml.***
    
    - Create namespace to contour's operator

    `kubectl apply -f kubernetes/projectcontour/namespace.yaml`

    - Create dockerhub dockerhubregkey in the namespace 'projectcontour'

    `kubectl create secret docker-registry contourdockerhubregkey --docker-server=https://index.docker.io/v2/ --docker-username=<dockerhub username> --docker-password=<dockerhubpassword> --docker-email=your@email.com --namespace projectcontour`

    - Modify the contour.yaml to add imagepull secret to the service account, deployment and daemonset
    Find the sevice account, deployment and daemonset and add a imagepullsecret called 'contourdockerhubregkey' (as created in the above step) there.
    *this is already done in the kubernetes/projectcontour/contour.yaml file in this repo now*

    - deploy operator

    `kubectl apply -f kubernetes/projectcontour/contour.yaml`

    check the deployment

    `kubectl get all -n projectcontour`

    Mine looked like this:
    ```
    kubectl get all -n projectcontour
    NAME                                READY   STATUS      RESTARTS   AGE
    pod/contour-7f85cb8685-lzxqk        1/1     Running     0          5m7s
    pod/contour-7f85cb8685-r5rzd        1/1     Running     0          5m7s
    pod/contour-certgen-v1.16.0-fv268   0/1     Completed   0          5m11s
    pod/envoy-5mkxs                     2/2     Running     0          5m5s
    pod/envoy-p6g5t                     2/2     Running     0          5m6s

    NAME              TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
    service/contour   ClusterIP      10.96.151.99   <none>           8001/TCP                     5m10s
    service/envoy     LoadBalancer   10.96.246.71   192.168.220.11   80:32693/TCP,443:30791/TCP   5m9s

    NAME                   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    daemonset.apps/envoy   2         2         2       2            2           <none>          5m7s

    NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/contour   2/2     2            2           5m9s

    NAME                                 DESIRED   CURRENT   READY   AGE
    replicaset.apps/contour-7f85cb8685   2         2         2       5m9s

    NAME                                COMPLETIONS   DURATION   AGE
    job.batch/contour-certgen-v1.16.0   1/1           31s        5m15s
    ```

    *Looks like nothing failed*. Happy days.

    - Get the external IP or LB IP

    `kubectl -n projectcontour get service`

    - Create L4 LB to ingress and configure route to application services

    `kubectl apply -f kubernetes/global/ingress-contour-calculator.yaml`

- Deploy ingress controller (NGINX -- *Skip this if you have used contour*):

    **Why?** check my blog post here:

    - Install Helm on workstation. 
      
      `$ sudo snap install helm --classic (I had snap installed on my workstation)`

      `$ helm repo add bitnami https://charts.bitnami.com/bitnami`
      
      `$ helm repo update`

    - Install NGINX ingress controller on the calc-k8-cluster cluster

        `$ helm install calc-k8-ingress-controller bitnami/nginx-ingress-controller --namespace calc-k8-ingress-controller --create-namespace`

    - Apply ingress to our namespace "calculator"

        `$ kubectl apply -f kubernetes/global/ingress-nginx-calculator.yaml`

    - customise deployed NGINX ingress according to our applications needs
        
        `$ kubectl apply -f kubernetes/global/ingress-nginx-configmap.yaml`

        This will also create a L4 load balancer connecting to ingress controller. 
        Also this yaml will tell NGINX to map path and redirection to appropriate app via the "spec/rules".        

- Add the environment variables in the cluster:
    Our apps (in particular the calc-frontend) utilises some environment variable

    `kubectl apply -f kubernetes/global/configmap.prod.yaml`


STEP 5: Integrate private container registry to the cluster
=============================================================
In order for Kubertes to pull image from a authenticated private reposity in a private data centre or cloud our kubernetes cluster needs to know how to authenticate to the registry (in this case the restry is Harbor hosted on my SDDC / private cloud).

There are few different ways of doing it. In this example I did it by applying commands:

- **Create secret**:

    `kubectl create secret docker-registry harbor-regcred --docker-server=1.2.3.4 --docker-username=administrator@vsphere.local --docker-password=mysecretpassword --docker-email=admin@vsphere.com --namespace calculator`

    ** I kept the secrets.yaml file so that I can re-deploy if needed by running this command and saving the output in a yaml file. 
    
    `kubectl get secret harbor-regcred --output=yaml -n calculator`

    In this case the output is saved in kubernetes/global/harbor-secret.yaml.

    So next time if in case I am re-creating in a different environment I could do:

    `kubectl apply -f kubernetes/global/harbor-secret.yaml`

    Checkout this page: https://kubernetes.io/docs/concepts/configuration/secret/ for more details on this.

- **Configmap for private registry with self signed ssl**

    It is very common to have a private registry served using self signed certificate. In my case when Harbor was provisioned in the private cloud it was assigned a self signed cert.

    When kubernetes deploy an image it is like "docker pull". 

    In order to image pull from a registry with self signed cert the host or IP (in my case IP) need to be in the the /etc/docker/daemon.json of the running container.

    *BUT how do we dynamically do it with kubernetes when PODs / containers are create and destroyed (managed) by kubernetes itself??*

    This is what I did to resolve this.
    - Add the content of the daemon.json (of docker) as config to make it available to kubernetes.
    - During deployment volume mount it in the pod to /etc/docker/daemon.json.

    So create the config map (one off thingy as part of making cluster ready).
    
    `kubectl apply -f kubernetes/global/harbor-allow-insecure-registries.yaml`

    ***This DOES NOT WORK anymore. APPLY trust Tanzu Namespace wide. README located at: kubernetes/tanzu/readme.md***


STEP 6: Prepare app for deployment to the kubernetes cluster
================================================================
Let's use the "calc-addservice" app to list out the steps. All the other apps "calc-substractservice" and "calc-frontend" will do the exact same.

- **The Deployment file for kubernetes:**

    As everything else in kubernetes a deployment job is also defined using yaml.

    For this app the yaml file is calc-addservice/kubernetes/deployment.yaml. Few things to notice here:
    - volumeMounts: this is a dynamic volume from the configmap we applied at step 5
    - imagePullSecrets: this is the docker-registry secret we created in step 5.
    The rest of the file is self explanatory.

- **The Jenkins file for CICD with Jenkins:**

    This one is simple enough. 
    - Pod template: This section tells Jenkins master to spin a new POD (with Kubernetes -- with the help of Kubernetes Plugin). 
      - The POD will contain 2 containers:
        - Maven: to execute mvn command for building JAR of the java app.
        - Docker: to execute docker command for building docker images and pushing to our Harbor registry.
        - serviceAccount: here we will mention the service we created when we provisoined Jenkins. This SA is for jenkins to spin new pod, nothing to do with our app.
    - The POD will also have volume mount to /etc/docker/daemon.json (as explained in STEP 4).
    
    The rest of file is self explanatory. (see README of Jenkins section for the credentails)


STEP 7: Configure Jenkins trigger
==================================
We will configure in Jenkins:
- Poll every hour (or necessary interval or necessary schedule like every night after midnight) from the connected GIT repo.
- Use the "Jenkins" file for pipeline.
- Trigger Jenkins

# That's it. 
Our apps are now deployed to kubernetes.

*After deployed it is available at: calculator.10.xxx.xxx.68.nip.io (as per the ingress controller setting)*


Handy Commands:
=================================
`kubectl config view -o jsonpath='{"Cluster name\tServer\n"}{range .clusters[*]}{.name}{"\t"}{.cluster.server}{"\n"}{end}'`
`kubectl get configmap harbor-allow-insecure-registries -o jsonpath='{.data.daemon\.json}' -n calculator`
`kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='default')].data.token}"|base64 --decode`

