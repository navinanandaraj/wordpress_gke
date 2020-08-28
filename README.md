# wordpress_gke
POC for setting up wordpress on GKE

### GKE Cluster
---------------
As 90% of the customer traffic is from SFO and Tokyo, we will setup 2 regional clusters, one in asia-northeast1 and the other in us-west1. All external customer traffic will be routed through a GCLB to provide HA and fault tolerance for any regional failures.

Below is a sample gcloud command to create a GKE cluster in asia-northeast1. us-west1 would be something similar with its own subnet and secondary IP ranges 

gcloud beta container --project "micro-shoreline-220414" clusters create "gke-cluster-tokyo" --region "asia-northeast1" --no-enable-basic-auth --cluster-version "1.15.12-gke.2" --release-channel "stable" --machine-type "n1-standard-8" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "3" --enable-stackdriver-kubernetes --enable-private-nodes --enable-private-endpoint --master-ipv4-cidr "172.16.0.0/28" --enable-ip-alias --network "projects/micro-shoreline-220414/global/networks/default" --subnetwork "projects/micro-shoreline-220414/regions/asia-northeast1/subnetworks/gke-subnet-northeast1" --cluster-ipv4-cidr "10.97.192.0/18" --services-ipv4-cidr "10.98.255.0/28" --default-max-pods-per-node "110" --enable-network-policy --enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --maintenance-window-start "2020-08-25T04:00:00Z" --maintenance-window-end "2020-08-25T08:00:00Z" --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR,SA,SU" --enable-shielded-nodes --node-locations "asia-northeast1-a","asia-northeast1-b"

Key configuration settings<br/>
1. Private GKE cluster<br/>
2. Alias IP ranges enabled for Pods and service CIDRs<br/>
3. Master Authorized networks to a /28 subnet in the VPC<br/>
4. Auto upgrade with 4 hour maintenance window<br/>
5. Cluster Autoscaling enabled<br/>
6. Pod Security Policy enabled to run containers as non-root<br/>
7. Stackdriver Logging and Monitoring enabled<br/>
8. Log router sink setup for streaming to datawarehouse<br/>


### Persistent Volume
---------------------
Assumption that there is a Wordpress admin app for the initial setup that loads the content to a persistent disk<br/><br/>
Create a snapshot of the persistent disk that has all the contents, plugins etc<br/>
Create a regional persistent disk from the above snapshot. The disk would provide redundancy across 2 zones where the GKE nodes are hosted<br/>
Create a K8s PerisistentVolume (PV) and a PersistentVolumeClaim (PVC) resource using the above disk with ReadOnlyMany accessmode. This would enable multiple pods to mount the disk as readonly. Below is a sample yaml for creating PV and PVC<br/><br/>
apiVersion: v1<br/>
kind: PersistentVolume<br/>
metadata:<br/>
  name: pv-wordpress-v1<br/>
spec:<br/>
  capacity:
    storage: 500G<br/>
  accessModes:<br/>
    - ReadOnlyMany<br/>
  claimRef:<br/>
    namespace: default<br/>
    name: pv-wordpress-claim-v1<br/>
  gcePersistentDisk:<br/>
    pdName: wordpress=pd-v1<br/>
    fsType:ext4<br/>
\--<br/>
apiVersion: v1<br/>
kind: PersistentVolumeClaim<br/>
metadata:<br/>
  name: pv-wordpress-claim-v1<br/>
spec:<br/>
  accessModes:<br/>
    - ReadOnlyMany<br/>
  resources:<br/>
    requests:<br/>
      storage: 500G


### Cloud SQL
-------------
Setup a Cloud SQL instance in asia-northeast1 for the dynamic content. Since the DB usage is rare, we are ok with 1 CloudSQL instance in 1 region


### WP deployment 
-----------------
We’ll be using a base WP image from docker image. The DockerFile would include any custom changes on top of the bas image and maintained in a version control like GIT. There will be a CICD process to build a new docker image and push it to the project GCR for any WP version changes.


### WP content changes
----------------------
A separate PD for admin changes. Any content changes done through the WP admin app will be persisted to this disk. For making the new content available to the customers, we will create a new snapshot version of this disk. A new regional persistent disk (v2) will be created using this snapshot, followed by creating a v2 version of the PV and PV claim. The pod yaml will refer to this PV claim for the new deployment. The deployment can either be a blue/green or a rolling update. B/G is preferred.

