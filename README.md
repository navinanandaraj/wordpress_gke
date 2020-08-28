# wordpress_gke
POC for setting up wordpress on GKE

### GKE Cluster
---------------
As 90% of the customer traffic is from SFO and Tokyo, we will setup 2 regional clusters, one in asia-northeast1 and the other in us-west1. All external customer traffic will be routed through a GCLB to provide HA and fault tolerance for any regional failures.

Below is a sample gcloud command to create a GKE cluster in asia-northeast1. us-west1 would be something similar with its own subnet and secondary IP ranges 

gcloud beta container --project "micro-shoreline-220414" clusters create "gke-cluster-tokyo" --region "asia-northeast1" --no-enable-basic-auth --cluster-version "1.15.12-gke.2" --release-channel "stable" --machine-type "n1-standard-8” --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "1" --enable-stackdriver-kubernetes --enable-private-nodes --enable-private-endpoint --master-ipv4-cidr "172.16.0.0/28" --enable-ip-alias --network "projects/micro-shoreline-220414/global/networks/default" --subnetwork "projects/micro-shoreline-220414/regions/asia-northeast1/subnetworks/gke-subnet-northeast1" --cluster-ipv4-cidr "10.97.248.0/21" --services-ipv4-cidr "10.98.255.0/27" --default-max-pods-per-node "110" --enable-network-policy --enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --maintenance-window-start "2020-08-25T04:00:00Z" --maintenance-window-end "2020-08-25T08:00:00Z" --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR,SA,SU" --enable-shielded-nodes --node-locations "asia-northeast1-a","asia-northeast1-b"

Few key configuration settings
1.Private GKE cluster
2.Aias IP ranges enabled for Pods and service CIDRs
3.Master Authorized networks to a /28 subnet in the VPC
4.Auto upgrade with 4 hour maintenance window 
5.Cluster Autoscaling enabled
6.Pod Security Policy enabled to run containers as non-root
7.Stackdriver Logging and Monitoring enabled 
8.Log router sink setup for streaming to datawarehouse

