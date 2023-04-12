# Website-Application-Testing-in-the-Cloud

Suppose you are the DevOps engineer for a marketing agency that specializes in conducting promotional events and A/B testing for its clients. They regularly host limited-time online campaigns with custom web content, requiring frequent updates and deployments.

The agency needs a reliable, automated solution to manage these deployments, ensuring high availability and seamless scalability during peak traffic periods.

Using Kubernetes, the marketing agency can create two deployments with two replicas each, running nginx with custom index.html pages for different promotional campaigns.

A shared load balancer service is implemented to efficiently distribute incoming traffic between both deployments, ensuring the same IP address and port number for a seamless user experience.

Persistent storage is utilized to store the necessary configuration files, and a Kubernetes CronJob is set up to automatically spin up the deployments and the service at a specific time every day.

This automated solution allows the agency to reliably launch and manage promotional events and conduct critical A/B testing campaigns, providing optimal performance and user experience while minimizing manual intervention and maintenance.

In this walk-through, we will create two Kubernetes deployments with MicroK8s. Each deployment will have two pods running the nginx image and a custom index.html page. We will also create a service that points to both deployments, and create persistent storage. Finally, we will set up a CronJob to spin up both deployments and the service every day at 7:00 AM CST.

Pre-Requisites

An AWS Account
MicroK8s already installed on your EC2 Instance
Knowledge of Kubernetes and kubectl
Experience with containerization and Docker
Let’s get started:

Step 1 — Setting up your EC Instance & Installing your Kubernetes Environment

I covered this in great detail in a previous article here. Follow steps 1 & 2 in that article and then return to this article and continue onward to step number 2.

NOTE: When it comes to testing your results, you will want to enable an Elastic IP address when you launch your instance. If you already created one, you can create one post-launch and attach it to your running instance.

Step 2 — Setting up the Directories for our A/B Websites

Since the purpose of this deployment is to test which website performs better. We are going to need to create one directory that will contain two different .html web files for our two different deployments:

mkdir microk8s-deployments && cd microk8s-deployments

Step 3 — Creating ConfigMap files for Each Deployment

Our ConfigMap files are YAML that point to our custom .html files. One ConfigMap per .html file. To create our first YAML:

vim deployment-one-configmap # Creates a new empty file called "deployment-one-configmap.yaml
In our VIM editor, we construct our YAML:

apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-one-configmap
data:
  index.html: |
    This is Deployment One
We hit [CTRL+C] and then type “:wq!” to save our changes and exit VIM.


We create our second ConfigMap that points to our other .HTML for our second deployment in the same way we created our first:

vim deployment-two-configmap #Creates a new empty file called "deployment-one-configmap.yaml
Again in VIM, we construct our second YAML:

apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-two-configmap
data:
  index.html: |
    This is Deployment Two

Now we need to apply our configmaps to our cluster:

microk8s kubectl apply -f deployment-one-configmap
microk8s kubectl apply -f deployment-two-configmap

Step 4— Creating our Deployment YAMLs

Per our use-case, we are going to create two deployments both are accomplishing the same outcome but for their respective web pages.

We create these .yaml files using the VIM editor in the same way we did with our configmaps in the previous step. Each YAML represents a separate deployment but accomplishes the same task independently save for our different Webpages that we are testing:

Each YAML specifies the API version and kind of Kubernetes object being created (a Deployment).
Each gives the Deployment a name (“deployment-one” or “deployment-two”).
Each define the desired number of replicas for the deployment (2 pods in each deployment).
Each specifies a selector to identify which pods the deployment is responsible for (based on the label “app: deployment-one” or “deployment-two”).
Each defines a template for the Pods that should be created by the deployment.
Each defines one container named either “website-a” or “website-b”, using the official nginx Docker image.
Each template exposes port 80.
Each mounts a volume called “html” at the path /usr/share/nginx/html.
Each defines a volume called “html” that gets its contents from a configmap called “deployment-one-configmap”.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-one
spec:
  replicas: 2
  selector:
    matchLabels:
      app: common-app
      deployment: deployment-one
  template:
    metadata:
      labels:
        app: common-app
        deployment: deployment-one
spec:
      containers:
      - name: website-a
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        - name: custom-index
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: html
        emptyDir: {}
      - name: custom-index
        configMap:
          name: deployment-one-configmap
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-two
  labels:
    app: common-app
    deployment: deployment-two
spec:
  replicas: 2
  selector:
    matchLabels:
      app: common-app
      deployment: deployment-two
  template:
    metadata:
      labels:
        app: common-app
        deployment: deployment-two
 spec:
      containers:
      - name: website-b
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        - name: custom-index
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: html
        emptyDir: {}
      - name: custom-index
        configMap:
          name: deployment-two-configmap

YAML for deployment-one

YAML for deployment-two
Step 5 — Creating our Nginix-Service YAML
Our nginix-service.yaml file creates a load balancing service that forwards traffic to any pod deployments that are using a label under app: "common-app", on port 80, which if you remember from our previous step, we also used so that when TCP traffic enters our Load Balancer, our nginix-service will automatically route traffic between our website-a and website-b webservers.

The service is exposed externally using an external load balancer. This allows clients to access the pods through a stable, externally accessible IP address or hostname, regardless of how many replicas of the pods are running or where they are located in the cluster:

Once again, we create this .yaml file using the VIM editor in the same way we did with our ConfigMap and deployments files in the previous step:

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: common-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer

Step 6 — Creating our Persistent Storage & Claim Manifests
The next yaml configuration files that we need to define is for our persistent volume storage and our pod claims to that persistent volume.

Our first YAML will define a cluster-level resource while the latter represents a request for claim on that cluster storage by a particular pod.

Our persistent volume (cluster-level volume) will be configured as follows:

Named ‘website-pv’
Can store up to 250M of data
Can be accessed by a single node at a time using the ‘ReadWriteOnce’ access mode.
The data for this volume is store on the node’s filesystem in the ‘/temp/k8s’ directory
apiVersion: v1
kind: PersistentVolume
metadata:
  name: custom-pv
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/temp/k8s"

Persistent Storage Manifest YAML
Additionally, our persistent-storage-claim.yaml requests for storage by a pod running in the cluster:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: custom-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi

Persistent Storage Claim Manigest YAML
A PVC specifies the required capacity, access mode, and other parameters for the storage that the pod needs.

When a pod requests a PVC, Kubernetes looks for a persistent volume that matches the claim’s storage requirements.

If a suitable persistent volume is found, it is bound to the claim and made available to the Pod.

If a suitable PV is not found, Kubernetes will dynamically provision a new PV that matches the PVC’s requirements.

Step 7 — Creating our CronJob YAML
Our last yaml defines a CronJob that runs a daily batch job to deploy Kubernetes objects using microk8s kubectl apply. This YAML file allows us to automate the deployment of Kubernetes objects on a daily basis using a CronJob

apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-deployments-cronjob
spec:
  schedule: "0 7 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: my-app
        spec:
          containers:
          - name: my-container
            image: busybox
            command: ['sh', '-c', 'microk8s kubectl apply -f /mnt/deployment-one-configmap.yaml && microk8s kubectl apply -f /mnt/deployment-two-configmap.yaml && microk8s kubectl apply -f /mnt/deployment-one.yaml && microk8s kubectl apply -f /mnt/deployment-two.yaml && microk8s kubectl apply -f /mnt/service.yaml']
            volumeMounts:
            - name: manifests
              mountPath: /mnt
          restartPolicy: OnFailure
          volumes:
          - name: manifests
            persistentVolumeClaim:
              claimName: custom-pvc

CronJob YAML
Step 8— Applying our YAMLs to our Kubernetes Cluster
We run the following commands, in order to apply our persistent storage, persistent storage claim, and CronJob yamls to our cluster:

microk8s kubectl apply -f persistent-storage.yaml
microk8s kubectl apply -f persistent-storage-claim.yaml
microk8s kubectl apply -f cronjob.yaml

We have successfully applied our YAMLs to our cluster
Step 9— Testing our Deployment & Verifying our Results
Now, we can deploy our web servers. One for each of the websites that our agency will be testing and one for our Load Balancer:

microk8s kubectl apply -f deployment-one.yaml
microk8s kubectl apply -f deployment-two.yaml
microk8s kubectl apply -f nginx-service.yaml
We can verify the deployments and the number of pods using the following commands:

microk8s kubectl get deployments
microk8s kubectl get pods

Both our deployments as well as the number of pods per deployment were successfully launched.
In order to test public traffic to our deployments, we need to get the IP address from our Load Balancer using the following command:

microk8s kubectl get svc nginx-service

We can use the ‘curl’ command to validate that you eventually see the index.html pages from both deployment one and deployment 2.

for i in {1..10}; do curl 10.152.183.74; done
If we see both pages, we can have confidence that when visitors visit our webpage, they will be randomly routed between our two pages.


Victory is sweet for the captain!
Nice work! You’re successfully created a Kubernetes cluster deployment for website A/B testing!


