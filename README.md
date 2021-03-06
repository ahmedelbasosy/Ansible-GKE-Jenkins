# Ansible-GKE-Jenkins

##### Table of Contents  
- [Introduction](#introduction) <a name="Ansible-GKE-Jenkins"/>
- [Requirements](#requirements) <a name="Ansible-GKE-Jenkins"/>
- [Setup GCP Account](#setup_gcp_account) <a name="Ansible-GKE-Jenkins"/>
- [Setup Ansible Config File](#ansible_cfg_file) <a name="Ansible-GKE-Jenkins"/>
- [Setup GCP Dynamic Inventory File](#inventory_file)  <a name="Ansible-GKE-Jenkins"/>
- [Edit Variables Files](#var_files) <a name="Ansible-GKE-Jenkins"/>
- [Our Playbooks](#playbooks) <a name="Ansible-GKE-Jenkins"/>
- [Edit Jenkins Master Image Files](#jenkins_master_image) <a name="Ansible-GKE-Jenkins"/>
- [Templating NFS-provisioner & Jenkins-Master Deployment](#templating_files) <a name="Ansible-GKE-Jenkins"/>
- [Jenkins Master Deployment](#jenkins_master_deployment) <a name="Ansible-GKE-Jenkins"/>
- [Hello From Jenkins Master](#hello_from_jenkins_master) <a name="Ansible-GKE-Jenkins"/>
# introduction
This Repo shows What Can be done Using Ansible in order to provision a Full Environment having the following:

2- A Bastion host That Act As -->
- A Bastion Host.
- NFS Server to provide NFS storage to our GKE Cluster Both ( or you can use Google Firestore ~ Minimum 1TB Instance )
  -  "NFS storage Class" for Dynamic Provisioning                  --> /srv/nfs/external_nfs_sc
  -  "PersistentVolume" for Jenkins-Master Deployment              --> /srv/nfs/jenkins-master
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

## - Configuring our Google Cloud Account

 Login to your Google Cloud Account using Cloud Console Create a New Project if Needed
 
#### from your gcloud shell export the following environment vars:
 
     export SERVICE_ACCOUNT_ID=ansible-sa
     export PROJECT_ID=xxxxxxxxxxxxxxx
     

#### 1- Creating a Service Account
    
    gcloud config set project $PROJECT_ID

    gcloud iam service-accounts create $SERVICE_ACCOUNT_ID --display-name "Service account for Ansible"
    
    
    
#### 2- Assign the Needed IAM Roles to The Service Account 

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
    
#### 3- Create A Key For The Service Account and save it

    gcloud iam service-accounts keys create \
    .gcp/gcp-key-$SERVICE_ACCOUNT_ID.json \
    --iam-account=$SERVICE_ACCOUNT_ID@$PROJECT_ID.iam.gserviceaccount.com
    
#### 4- Add SSH Key To Project Metadata ( In Case you Are not using OSLogin )

    ssh-keygen -t rsa -f ansible_rsa -C ansible
    gcloud compute project-info add-metadata --metadata-from-file ssh-keys=ansible_rsa.pub

#### 5- Save The Service Account Keys and the SSH Keys to Play Under .files directory
    
     .files
     ????????? .ssh
     ???   ????????? ansible_rsa
     ???   ????????? ansible_rsa.pub
     ????????? SA_Key
         ????????? gcp-key-ansible-sa.json
          
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
     service_account_file: .files/SA_Key/gcp-key-ansible-sa.json
     ansible_ssh_user: ansible                                # Automatically Created When Adding SSH to Project Metadata From #setup_gcp_account step 4
     ansible_ssh_private_key_file: .files/.ssh/ansible_rsa    # You Can Set "private_key_file = .files/.ssh/ansible_rsa" in ansible.cfg file
     
     zones: 
     # populate inventory with instances in these Zones
     - us-central-a
     
     regions: 
     # populate inventory with instances in these Regions
     - us-central
     
     projects: 
     # populate inventory with instances in these Projects
     - YOUR_GCP_PROJECT_ID
     
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
       my-nodepool: "'my-nodepool' in name"                         # Edit This with you NodePool Name
     
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
    ????????? 01-global_vars.yaml                        --> Default GCP Variables ( Project, Region, Zone, Service Account Credentials File )
    ????????? 02-gke_vars.yaml                           --> GKE Cluster and Nodepool Variables ( Cluster_Name, Nodepool_Name, Versions, Number of Nodes )
    ????????? 03-gke_oauth_scopes.yaml                   --> GKE Cluster OAuth Scopes
    ????????? 04-docker_vars.yaml                        --> Docker Variables ( username, password, and Image Name to be used )
    ????????? nfs_ip.yaml                                --> Used to Pass a Variable from 1st play to the 2nd play
    
  ## 01-global_vars.yaml
--> Default GCP Variables ( Project, Region, Zone, Service Account Credentials File )

    gcp_def_zone: "YOUR_GCP_DEFAULT_ZONE"
    gcp_def_region: "YOUR_GCP_DEFAULT_REGION"
    gcp_project: YOUR_GCP_PROJECT_ID
    gcp_cred_kind: serviceaccount
    gcp_cred_file: ../.files/SA_Key/gcp-key-ansible-sa.json
    
  ## 02-gke_vars.yaml

  # IMPORTANT:

  using your gcloud run the following to get the correct values to be used:

    gcloud container get-server-config --zone { gcp_def_zone }

--> GKE Cluster and Nodepool Variables ( Cluster_Name, Nodepool_Name, Versions, Number of Nodes )

    gke_cluster_name: YOUR_GKE_CLUSTER_NAME
    gke_cluster_version: YOUR_GKE_CLUSTER_VERSION               # EX: 1.20.8-gke.2100
    gke_node_pool_name: YOUR_NODEPOOL_NAME
    gke_node_machine_type: NODEPOOL_INSTANCE_TYPE               # EX: e2-medium
    gke_node_kube_version: YOUR_NODEPOOL_KUBE_VERSION           # EX: 1.20.6-gke.1000
    gke_node_image_type: YOUR_NODEPOOL_IMAGE_TYPE               # EX: UBUNTU_CONTAINERD
    gke_node_disk_size_gb: NODEPOOL_BOOT_DISK_SIZE_IN_GB        # EX: 50
    gke_max_pod_per_node: MAX_POD_PER_NODE                      # Max 110
    gke_node_count: GKE_NODEPOOL_INITIAL_COUNT                  # EX: 1
    gke_max_node_count: GKE_NODEPOOL_MAX_COUNT                  # EX: 3
    gke_min_node_count: GKE_NODEPOOL_MIN_COUNT                  # EX: 1

    
  ## 03-gke_oauth_scopes.yaml
--> GKE Cluster OAuth Scopes

    oauth_scopes:
     - "https://www.googleapis.com/auth/compute"
     - "https://www.googleapis.com/auth/devstorage.read_only"
     - "https://www.googleapis.com/auth/logging.write"
     - "https://www.googleapis.com/auth/monitoring"
     - "https://www.googleapis.com/auth/servicecontrol"
     - "https://www.googleapis.com/auth/service.management.readonly"
     - "https://www.googleapis.com/auth/trace.append"
     
  ## 04-docker_vars.yaml 
--> Docker Variables ( username, password, and Image Name to be used )

      docker_username: YOUR_DOCKER_HUB_USERNAME
      docker_password: 'YOUR_DOCKER_HUB_PASSWORD'
      jenkins_master_image: YOUR_DOCKER_IMAGE_FOR_JENKINS_MASTER       # EX: ahmedhelbasosy/jenkins-master

# playbooks

Our Playbook.yaml file would import 3 Playbooks from Playbooks directory:

    playbooks
    ????????? 01-bootsraping-gke.yaml
    ????????? 02-bootstraping-bastion.yaml
    ????????? 03-deploy-NFS_JenkinsMaster.yaml

Our Playbook.yaml file content:
    
    ---
    - name: Running "Bootstraping GKE Cluster" PLAYBOOK
      import_playbook: playbooks/01-bootsraping-gke.yaml 

    - name: Running "Bootstrapping Bastion" PLAYBOOK
      import_playbook: playbooks/02-bootstraping-bastion.yaml

    - name: Running "Deploying NFS Provisioner & Jenkins Master" PLAYBOOK 
      import_playbook: playbooks/03-deploy-NFS_JenkinsMaster.yaml
    

  ## 01-bootsraping-gke.yaml 

    roles:
      - ../roles/gke-NETWORK                   # Creating a GCP Network
      - ../roles/gke-FIREWALL                  # Applying External & Internal Default Firewall Rules to GCP Network 
      - ../roles/Bastion-NFS                   # Creating a Bastion Host with Extra Disk to be used for NFS Shares
      - ../roles/gke-CLUSTER                   # Bootstraping a GKE Cluster
      - ../roles/gke-NodePool                  # Bootstrapping a NodePool 
      
  ## 02-bootstraping-bastion.yaml

    roles:
      - ../roles/Bastion-NFS-preparation                   # Installing Required Software Packages
      - ../roles/jenkins-master-IMAGE                      # Building and Publishing Jenkins-Master Image using Bastion Host
      - ../roles/Templating-Files                          # Templating Files For NFS-Provisioner & Jenkins-Master Deployment
      
  ## 03-deploy-NFS_JenkinsMaster.yaml

    roles:
      - ../roles/gke-NFS-provisioner                      # Deploying NFS-Provisioner To Our GKE Cluster
      - ../roles/gke-JENKINS-Master-Deployment            # Deploying Jenkins-Master To our GKE Cluster

# jenkins_master_image

  ### IMPORTANT:
  Our Image wiil be tagged TWICE:
  - jenkins-master:date                                   # date +%d%m%y
  - jenkins-master:latest

you will find the Jenkins-Master Image Files Under:

      roles/jenkins-master-IMAGE/files/jenkins-master
      
      jenkins-master/
      ????????? Dockerfile                                    # Docker File
      ????????? default-user.groovy                           # .groovy File Used to Setup Default Admin Account
      ????????? plugins.txt                                   # Some Plugins to be installed during Image Build
      
 Our Docker File looks like the following:
      
      FROM jenkins/jenkins
      ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false    # Skipping Setup Wizard
      USER root


      COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
      RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt    # Installing Plugins

      ENV JENKINS_USER admin                          # Default User Name
      ENV JENKINS_PASS password                       # Default User Password

      COPY default-user.groovy /usr/share/jenkins/ref/init.groovy.d/


# templating_files
The Templating-Files role is used to Customizing Some YAML files and Copying Some Static Files too

to /root/YAML on Bastion Host

    Templating-Files
    ????????? files
    ???   ????????? jenkins-master-deployment
    ???   ???   ????????? jenkins-master-pvc.yaml
    ???   ???   ????????? namespace.yaml
    ???   ???   ????????? services.yaml
    ???   ????????? nfs-provisioner
    ???       ????????? NFS-rbac.yaml
    ???
    ????????? templates
        ????????? NFS-deployment.yaml.j2
        ????????? exports.j2
        ????????? external_nfs_sc.yaml.j2
        ????????? jenkins-master-deployment.yaml.j2
        ????????? jenkins-master-pv.yaml.j2


# jenkins_master_deployment
After Building our Jenkins-Master Image on Bastion Host. It's time to DEPLOY it to our GKE Cluster

- a jenkins-master Namespace will be created
- a PersistenVolume from NFS share to be created --> /nfs/srv/jenkins-master/ on Bastion Host
- a PersistenVolumeClaim to be created
- a jenkins-master:latest Deployment to be created using our Jenkins-Master Image

# hello_from_jenkins_master

At the End you will get a direct link to you Jenkins Pod using it's LoadBalancer Service IP

      TASK [../roles/gke-JENKINS-Master-Deployment : Hello From Jenkins-Master]             
      ok: [bastion-ubuntu] => {
                "msg": "You Can Access Your Jenkins Deployment from http://YOUR_JENKINS_POD_Public_IP:8080"
                  }


### Note: 
      username: admin
      password: password
