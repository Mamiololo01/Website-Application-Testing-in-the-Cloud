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
