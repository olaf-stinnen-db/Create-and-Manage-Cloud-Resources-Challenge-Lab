https://www.youtube.com/watch?app=desktop&v=xAAn7Cgu-OE

# Create-and-Manage-Cloud-Resources-Challenge-Lab
# Task 1: Create a project jumphost instance
## ...use f1-micro for small Linux VMs, and use n1-standard-1 for Windows or other applications, such as Kubernetes nodes

1. Navigation menu > Compute engine > VM Instance
2. On the console, Go to Compute Engine > VM instances > Create
3. Enter the Instance Name as "Instance name"
4. Under Machine Configuration section, Select N1 series and under machine Type select ‘f1-micro‘
5. Click on ‘Create’.

# Task 2: Create a Kubernetes service cluster
## Create a cluster (in the us-east1-b zone) to host the service
### From Kubernetes Engine Quick Start Lab, Task 1

1. gcloud config set compute/zone us-east1-b

## Naming normally uses the format team-resource; for example, an instance could be named nucleus-webserver1
### From Kubernetes Engine Quick Start Lab, Task 2: Create a GKE cluster

2. gcloud container clusters create nucleus-webserver1

## Use the Docker container hello-app (gcr.io/google-samples/hello-app:2.0) as a place holder; the team will replace the container with their own work later.
### From Kubernetes Engine Quick Start Lab, Task 3: Get authentication credentials for the cluster

3. gcloud container clusters get-credentials nucleus-webserver1

### From Kubernetes Engine Quick Start Lab,Task 4: Deploy an application to the cluster
#### 1. To create a new Deployment hello-server from the hello-app container image, run the following kubectl create command:
#### "kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0"

4. kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:2.0

## Expose the app on App port on port number
### From Kubernetes Engine Quick Start Lab,Task 4: Deploy an application to the cluster
#### 2. To create a Kubernetes Service, which is a Kubernetes resource that lets you expose your application to external traffic, run the following kubectl expose command:
#### "kubectl expose deployment hello-server --type=LoadBalancer --port 8080"
#### In this command:
####    --port specifies the port that the container exposes.
####    type="LoadBalancer" creates a Compute Engine load balancer for your container.

5. kubectl expose deployment hello-app --type=LoadBalancer --port 8080

### From Kubernetes Engine Quick Start Lab,Task 4: Deploy an application to the cluster
#### 3. To inspect the hello-server Service, run kubectl get: "kubeectl get service"

6. kubectl get service 

# Task 3: Setup an HTTP load balancer
## From Task Description "Use the following code to configure the web servers; the team will replace this with their own configuration later.":
### GCP Doku Startskripts auf Linux-VMs verwenden: https://cloud.google.com/compute/docs/instances/startup-scripts/linux
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

# Task 3.1 Create an instance template :
## From Set Up Network and HTTP Load Balancers Quick Lab, Task 5: Create an HTTP load balancer, 5.1 First, create the load balancer template
### From GCP Doku Linux-Startskript aus einer lokalen Datei an eine vorhandene VM übergeben: https://cloud.google.com/compute/docs/instances/startup-scripts/linux
gcloud compute instance-templates create nginx-template
--metadata-from-file startup-script=startup.sh

# Task 3.2. Create a target pool :
## From Set Up Network and HTTP Load Balancers Quick Lab, Task 3: Configure the load balancing service, 
### Task 3.1 Add a target pool in the same region as your instances. Run the following to create the target pool and use the health check, which is required for the service to function:
#### gcloud compute target-pools create www-pool \
####    --region us-central1 --http-health-check basic-check
gcloud compute target-pools create nginx-pool
#### When prompted, select ‘N’ and select appropriate region number: us-east1-b

# Create an HTTP load balancer with a managed instance group of 2 nginx web servers
# Task 3.3 Create a managed instance group :
## From Set Up Network and HTTP Load Balancers Quick Lab, Task 5: Create an HTTP load balancer, 2. Create a managed instance group based on the template:
### gcloud compute instance-groups managed create lb-backend-group \
###   --template=lb-backend-template --size=2 --zone=us-central1-a
### Parameter zu Befehl "gcloud compute instance-groups managed create" wird erläutert in: https://cloud.google.com/sdk/gcloud/reference/compute/instance-groups/managed/create

gcloud compute instance-groups managed create nginx-group
--base-instance-name nginx
--size 2
--template nginx-template
--target-pool nginx-pool

### Check if the two instances have been created
### Parameter zu Befehl "gcloud compute instances list" in: https://cloud.google.com/sdk/gcloud/reference/compute/instances/list
gcloud compute instances list

4 .Create a firewall rule to allow traffic (80/tcp) :

gcloud compute firewall-rules create www-firewall --allow tcp:80

gcloud compute forwarding-rules create nginx-lb
--region us-east1
--ports=80
--target-pool nginx-pool

gcloud compute forwarding-rules list

5 .Create a health check :

gcloud compute http-health-checks create http-basic-check

gcloud compute instance-groups managed
set-named-ports nginx-group
--named-ports http:80

6 .Create a backend service and attach the manged instance group :

gcloud compute backend-services create nginx-backend
--protocol HTTP --http-health-checks http-basic-check --global

gcloud compute backend-services add-backend nginx-backend
--instance-group nginx-group
--instance-group-zone us-east1-b
--global

7 .Create a URL map and target HTTP proxy to route requests to your URL map :

gcloud compute url-maps create web-map
--default-service nginx-backend

gcloud compute target-http-proxies create http-lb-proxy
--url-map web-map

8 .Create a forwarding rule :

gcloud compute forwarding-rules create http-content-rule
--global
--target-http-proxy http-lb-proxy
--ports 80

gcloud compute forwarding-rules list
