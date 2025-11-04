## Configure the load balancing service

gcloud compute addresses create network-lb-ip-1 \
  --region us-west3

gcloud compute http-health-checks create basic-check

gcloud compute target-pools create www-pool \
  --region us-west3 --http-health-check basic-check

gcloud compute target-pools add-instances www-pool \
    --instances web1,web2,web3

gcloud compute forwarding-rules create www-rule \
    --region  us-west3 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool

gcloud compute forwarding-rules describe www-rule --region us-west3

IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region us-west3 --format="json" | jq -r .IPAddress)

echo $IPADDRESS

while true; do curl -m1 $IPADDRESS; done