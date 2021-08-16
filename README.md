# Ansible-GKE-Jenkins

##### Table of Contents  
- [Introduction](#introduction) <a name="Ansible-GKE-Jenkins"/>
- [Requirements](#requirements) <a name="Ansible-GKE-Jenkins"/>
- [Setup GCP Account](#setup_gcp_account) <a name="Ansible-GKE-Jenkins"/>
- [Setup Ansible Config File](#ansible_cfg_file) <a name="Ansible-GKE-Jenkins"/>
- [Setup GCP Dynamic Inventory File](#inventory_file)  <a name="Ansible-GKE-Jenkins"/>
- [Edit Variable Files](#var_files) <a name="Ansible-GKE-Jenkins"/>

# introduction
This Repo shows What Can be done Using Ansible in order to provision a Full Environment having the following:

2- A Bastion host That Act As -->
- A Bastion Host.
- NFS Server to provide NFS storage to our GKE Cluster Both ( or you can use Google Firestore ~ Minimum 1TB Instance )
  -  "NFS storage Class" for Dynamic Provisioning
  -  "PersistentVolume" for Jenkins-Master Deployment  
- Docker Host where we will build our images
  -  jenkins-master
- Kubernetes client to GKE cluster, so we will create our resources from there

3- GKE Cluster -->
- Static GKE Cluster
- AutoScaling Nodepool
- NFS Storage Class from our Bastion Host

4- Building Automated Jenkins Master Docker Image --> 
- Skipping Setup Wizard
- Creating Admin Account
- installing Some Plugins

5- Jenkins Deployment -->
- Deploy Jenkins Master on GKE

# requirements
1- Google Cloud Account -->
- Service Account
- SSH Key to be added to project Metadata

# setup_gcp_account

- Configuring our Google Cloud Account

 Login to your Google Cloud Account using Cloud Console Create a New Project if Needed
 
 from your gcloud shell export the following environment vars:
 
     export SERVICE_ACCOUNT_ID=ansible-sa
     export PROJECT_ID=xxxxxxxxxxxxxxx
     

1- Creating a Service Account
    
    gcloud config set project $PROJECT_ID

    gcloud iam service-accounts create $SERVICE_ACCOUNT_ID --display-name "Service account for Ansible"
    
    
    
2- Assign the Needed IAM Roles to The Service Account 

    for role in \
    'roles/editor' \
    'roles/iam.serviceAccountUser' \
    'roles/compute.instanceAdmin' \
    'roles/compute.instanceAdmin.v1' \
    'roles/compute.osLogin' \
    'roles/compute.osAdminLogin' \
    'roles/container.clusterAdmin' \
    'roles/container.admin'
    
    do \
    gcloud projects add-iam-policy-binding \
    $PROJECT_ID \
    --member=serviceAccount:$SERVICE_ACCOUNT_ID@$PROJECT_ID.iam.gserviceaccount.com \
    --role=$role
    done
    
3- Create A Key For The Service Account and save it

    gcloud iam service-accounts keys create \
    .gcp/gcp-key-$SERVICE_ACCOUNT_ID.json \
    --iam-account=$SERVICE_ACCOUNT_ID@$PROJECT_ID.iam.gserviceaccount.com
    
4- Add SSH Key To Project Metadata ( In Case you Are not using OSLogin )

    ssh-keygen -t rsa -f ansible_rsa -C ansible
    gcloud compute project-info add-metadata --metadata-from-file ssh-keys=ansible_rsa.pub

5- Save The Service Account Keys and the SSH Keys to Play Under .files
    
     .files
     ├── .ssh
     │   ├── ansible_rsa
     │   └── ansible_rsa.pub
     └── SA_Key
         └── gcp-key-ansible-sa.json
          
# ansible_cfg_file

      [defaults]
      host_key_checking = False                             # Disabling Host Key checking
      private_key_file = .files/.ssh/ansible_rsa            # SSH Private Key 
      inventory = gkeplay.gcp.yml                           # Inventory File 
      [inventory]
      enable_plugins = gcp_compute

# inventory_file

GCP Dynamic inventory file name should be in that format ( don't use inventory.gcp.yml )
     
     NAME.gcp.yml
     
A Detailed Description of Inventroy File:
    
     plugin: google.cloud.gcp_compute
     auth_kind: serviceaccount
     service_account_file: .files/SA_Key/KEY_FILE.json
     ansible_ssh_user: ansible
     ansible_ssh_private_key_file: .files/.ssh/ansible_rsa    # You Can Set "private_key_file =" in ansible.cfg file
     
     zones: 
     # populate inventory with instances in these Zones
     - us-central-a
     
     regions: 
     # populate inventory with instances in these Regions
     - us-central
     
     projects: 
     # populate inventory with instances in these Projects
     - PROJECT_ID
     
     filters: 
     # Populate inventory with instances matching these filtes
     - machineType = n1-standard-1
     - scheduling.automaticRestart = true AND machineType = n1-standard-1
     
     keyed_groups:
     # Create groups from GCE labels
     - prefix: zone
       key: zone
     
     groups:
     # Custom Grouping of Instances
       bastion-hosts: "'bastion' in name"
       my-nodepool: "'my-nodepool' in name"
     
you can add the following too ( For this play we don't need them )

     hostnames:
     # List host by name instead of the default public ip
     - name
     
     compose:
     # Set an inventory parameter to use the Public IP address to connect to the host
     # For Private ip use "networkInterfaces[0].networkIP"
     ansible_host: networkInterfaces[0].accessConfigs[0].natIP

You Can Verify by doing the following:

     ansible-inventory -i NAME.gcp.yml --graph
     
The Output should look like the following:

    @all:
    |--@bastion_hosts:
    |  |--bastion-ubuntu
    |--@my_nodepool:
    |  |--gke-my-cluster-1-my-nodepool-98246188-0n39
    |--@ungrouped:
    |--@zone_us_central1_a:
    |  |--bastion-ubuntu
    |  |--gke-my-cluster-1-my-nodepool-98246188-0n39
    |--@zone_us_west4_c:
    |  |--node-gke-pool-7021
    
# var_files
    vars/
    ├── 01-global_vars.yaml                        --> Default GCP Variables
    ├── 02-gke_vars.yaml                           --> GKE Cluster and Nodepool Variables
    ├── 03-gke_oauth_scopes.yaml                   --> GKE Cluster OAuth Scopes
    └── nfs_ip.yaml                                --> Used to Pass a Variable from 1st play to the 2nd play
    
