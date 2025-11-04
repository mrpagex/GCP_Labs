# Set Up Application Load Balancers
Overview
In this hands-on lab you learn how to set up a Layer 7 (L7) application load balancer on Compute Engine virtual machines (VMs). L7 load balancers can understand HTTP(S) protocols, allowing them to make routing decisions based on parameters like URL, headers, cookies, and the content of the request. This allows for improved application performance and responsiveness.

There are several ways you can load balance on Google Cloud. This lab takes you through the setup of the following load balancers:
Application Load Balancer
https://cloud.google.com/load-balancing/docs/load-balancing-overview?hl=pt-br#a_closer_look_at_cloud_load_balancers

## TASK 1 - Set the default region and zone for all resources
Set the default region:
gcloud config set compute/region Region(substitua pela regiao)

In Cloud Shell, set the default zone:
gcloud config set compute/zone Zone(substitua pela zona)

# TASK 2 - Same as 1 - Set Up NLB

# TASK 3 - Create an Application Load Balancer
Application Load Balancing is implemented on Google Front End (GFE). GFEs are distributed globally and operate together using Google's global network and control plane. You can configure URL rules to route some URLs to one set of instances and route other URLs to other instances.

Requests are always routed to the instance group that is closest to the user, if that group has enough capacity and is appropriate for the request. If the closest group does not have enough capacity, the request is sent to the closest group that does have capacity.

To set up a load balancer with a Compute Engine backend, your VMs need to be in an instance group. The managed instance group provides VMs running the backend servers of an external application load balancer. For this lab, backends serve their own hostnames.

# First, create the load balancer template:
gcloud compute instance-templates create lb-backend-template \
   --region=Region \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'

Managed instance groups (MIGs) let you operate apps on multiple identical VMs. You can make your workloads scalable and highly available by taking advantage of automated MIG services, including: autoscaling, autohealing, regional (multiple zone) deployment, and automatic updating.

# Create a managed instance group based on the template:
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=Zone

# Create the fw-allow-health-check firewall rule:
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80

  Note: The ingress rule allows traffic from the Google Cloud health checking systems (130.211.0.0/22 and 35.191.0.0/16). This lab uses the target tag allow-health-check to identify the VMs

# Now that the instances are up and running, set up a global static external IP address that your customers use to reach your load balancer:
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
Copied!
Notice that the IPv4 address that was reserved:

gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global

Note: Save this IP address, as you need to refer to it later in this lab.

# Create a health check for the load balancer (to ensure that only healthy backends are sent traffic):
gcloud compute health-checks create http http-basic-check \
  --port 80

# Create a backend service:
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global

# Add your instance group as the backend to the backend service:
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=Zone \
  --global

# Create a URL map to route the incoming requests to the default backend service:
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service

Note: URL map is a Google Cloud configuration resource used to route requests to backend services or backend buckets. For example, with an external Application Load Balancer, you can use a single URL map to route requests to different destinations based on the rules configured in the URL map:
Requests for https://example.com/video go to one backend service.
Requests for https://example.com/audio go to a different backend service.
Requests for https://example.com/images go to a Cloud Storage backend bucket.
Requests for any other host and path combination go to a default backend service.    

# Create a target HTTP proxy to route requests to your URL map:
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http

# Create a global forwarding rule to route incoming requests to the proxy:
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80

Note: A forwarding rule and its corresponding IP address represent the frontend configuration of a Google Cloud load balancer. Learn more about the general understanding of forwarding rules from the Forwarding rules overview guide.
https://cloud.google.com/load-balancing/docs/forwarding-rule-concepts?hl=pt-br

# TASK 4 - Test traffic sent to your instances
![alt text](test_traffic.png)


