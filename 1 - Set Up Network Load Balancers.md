# Set Up Network Load Balancers
Overview
In this hands-on lab you learn how to set up a passthrough network load balancer (NLB) running on Compute Engine virtual machines (VMs). A Layer 4 (L4) NLB handles traffic based on network-level information like IP addresses and port numbers, and does not inspect the content of the traffic.

There are several ways you can load balance on Google Cloud. This lab takes you through the setup of the following load balancers:

[Network Load Balancer](https://cloud.google.com/load-balancing/docs/load-balancing-overview?hl=pt-br#a_closer_look_at_cloud_load_balancers)

# TASK 1 - Set the default region and zone for all resources
Set the default region:
gcloud config set compute/region Region(substitua pela regiao)

In Cloud Shell, set the default zone:
gcloud config set compute/zone Zone(substitua pela zona)

# TASK 2 - #Create multiple web server instances

gcloud compute instances create web1 \
    --zone=us-west3-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html'


  gcloud compute instances create web2 \
    --zone=us-west3-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: web2</h3>" | tee /var/www/html/index.html'

  gcloud compute instances create web3 \
    --zone=us-west3-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'

# Create a firewall rule to allow external traffic to the VM instances:
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80

# Run the following to list your instances. You'll see their IP addresses in the EXTERNAL_IP column:
gcloud compute instances list

# Verify that each instance is running with curl:
curl http://[IP_ADDRESS]

# TASK 3 - Configure the load balancing service:
When you configure the load balancing service, your virtual machine instances receives packets that are destined for the static external IP address you configure. Instances made with a Compute Engine image are automatically configured to handle this IP address.

Note: Learn more about how to set up Network Load Balancing from the Backend service-based external passthrough Network Load Balancer overview guide.

# Create a static external IP address for your load balancer:
gcloud compute addresses create network-lb-ip-1 \
  --region Region

# Add a legacy HTTP health check resource:
gcloud compute http-health-checks create basic-check

# TASK 4 -  Create the target pool and forwarding rule
A target pool is a group of backend instances that receive incoming traffic from external passthrough NLBs. All backend instances of a target pool must reside in the same Google Cloud region.

# Run the following to create the target pool and use the health check, which is required for the service to function:
gcloud compute target-pools create www-pool \
  --region Region --http-health-check basic-check

# Add the instances you created earlier to the pool:
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3

# Next you'll make the forwarding rule. A forwarding rule specifies how to route network traffic to the backend services of a load balancer.

Add a forwarding rule:
gcloud compute forwarding-rules create www-rule \
    --region  Region \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool

# TASK 5 - Send traffic to your instances
Now that the load balancing service is configured, you can start sending traffic to the forwarding rule and watch the traffic be dispersed to different instances.

# Enter the following command to view the external IP address of the www-rule forwarding rule used by the load balancer:
gcloud compute forwarding-rules describe www-rule --region Region

# Access the external IP address:
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region Region --format="json" | jq -r .IPAddress)

# Show the external IP address:
echo $IPADDRESS

# Use the curl command to access the external IP address, replacing IP_ADDRESS with an external IP address from the previous command:
while true; do curl -m1 $IPADDRESS; done

# In this lab, you have built a Network Load Balancer and practiced sending traffic to a forwarding rule and watched the traffic get distributed to different instances.
