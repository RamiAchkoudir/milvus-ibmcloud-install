[README.md](https://github.com/user-attachments/files/22255936/README.md)
# milvus-ibmcloud-install

[Milvus](https://milvus.io/) is an open-source vector database built to power embedding similarity search and AI applications. The steps below explain how to provision on IBM Cloud.

## Pre Requisites

- IBM Cloud account and workspace
- Resource Group

## IBM Cloud Setup

1. Go to the IBM Cloud Catalog (https://cloud.ibm.com/catalog)
2. From the ‘Compute’ category, select the Virtual Server for VPC option.
3. Set the following configuations:
   - Architecture: Intel
   - Location: your choice. Recommendation is to use the location closest to you.
   - Name: provide a unique name
   - Resource group: select Resource Group created as part of the pre-requisites
   - Tags: optional
   - Access management tags: optional
   - Image: leave as default (Centos 7.9 has been tested for this guide)
   - Profile: bx2d-4×16
     > This needs to be bx2d-4×16 as per the Milvus documentation hardware pre-reqs (https://milvus.io/docs/prerequisite-docker.md)
4. Click on the button to create an SSH key. Set the following configurations:
   - Provide a name for the SSH key
   - Assign the SSH to the same Resource Group
   - Tags: optional
   - Access management tags: optional
   - SSH key type: RSA
   - SSH key input method
5. Click the ‘Save private key’ button to download it to your machine, storing it where you keep your SSH keys.
6. Click the ‘Save public key’ button to download it to your machine
7. The previous two steps activate the ‘Create’ button. Click create and wait for the server to be provisioned.
8. From the ‘Virtual Server Instances’ page on IBM Cloud, you need to reserve a floating IP to access the VSI. Under network interfaces, click on the pencil to edit.
9. Under ‘Floating IP address’, select ‘Reserve a new floating IP’ and click save.
10. To be able to SSH into the VM, the security group needs to be updated. Navigate to ‘Security Groups’ from the panel on the left
11. Click on the security group which is associated with your VSI.
12. Select ‘Manage Rules’
13. Enable security groups for TCP all - needed to do this to access the VM
14. Edit the TCP inbound rules to allow any range
    > Important Note: Depending on the nature of the project (use of client data etc.), opening the ports for everything is not a good idea and you should set the port range to ports *19530-19530*

You are now ready to connect to the VSI

## Connect to the VSI

1. Navigate to your terminal
2. SSH into the machine. You need to copy the Floating IP address from IBM Cloud:

```
    ssh root@<IP address> -i <path to private key>
```

3. Enter ‘yes’ if asked to continue connecting
4. If you get an error saying `Permissions 0644 for 'xxxprv' are too open.`, you need to change the permissions on the private key. Enter:

   ```
   chmod 600 <private key file>
   ```

   Then reconnect via the ssh command

You should now have access to the VSI.

## Installing Pre-requisites on VM

The next steps follow the Milvus pre-requisite documentation https://milvus.io/docs/prerequisite-docker.md

1. First the rpm repository needs to be installed. Run:

   ```
   sudo yum install -y yum-utils
   sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

   ```

   > This is from https://docs.docker.com/engine/install/centos

2. Next, install Docker Engine and Docker compose:

   ```
   sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

3. Enter ‘y’ twice, when asked if the items to be installed is OK

4. Start docker by running:

   ```
   sudo systemctl start docker
   ```

5. Install wget to be able to get the yaml file:
   ```
   sudo yum install wget
   ```

## Install Milvus

The next steps follow the Milvus documentation for the stand alone version https://milvus.io/docs/prerequisite-docker.md

1. Download the yaml file:

   ```
   wget https://github.com/milvus-io/milvus/releases/download/v2.3.0/milvus-standalone-docker-compose.yml -O docker-compose.yml
   ```

2. Start Milvus:

   ```
   sudo docker compose up -d
   ```

3. Check the containers are running. The port Milvus is running on should be 19530 but check this in the output of the following command
   ```
   docker ps
   ```

## Changing the default password

By default the root user (password: Milvus) is created. It is best practice to change the password of the root user when you install Milvus.

To enable authentication you have to set `common.security.authorizationEnabled` in as `true` when configuring Milvus to enable authentication.

There are two ways of setting this up. 

You can [copy the configuration file into the container](#copy-the-configuration-file-into-the-container) or you can [Apply configurations to docker compose](#apply-configurations-to-docker-compose).

Once authentication is enabled and you are connected to your milvus instance you can change the root user password.

```
from pymilvus import utility
utility.reset_password('user', 'old_password', 'new_password', using='default')

# Or you can use an alias function update_password
utility.update_password('user', 'old_password', 'new_password', using='default')
```

Make sure to reset the connection after changing the password and testing that the old password is no longer useable.

### Copy the configuration file into the container.

1. Download the `milvus.yaml` configuration file from [https://raw.githubusercontent.com/milvus-io/milvus/v2.3.9/configs/milvus.yaml](https://raw.githubusercontent.com/milvus-io/milvus/v2.3.9/configs/milvus.yaml)

2. Find and update the value of `authorizationEnabled` to `true`

```console
  security:
    authorizationEnabled: true
    # The superusers will ignore some system check processes,
    # like the old password verification when updating the credential
    # superUsers: root
    tlsMode: 0
```

3. Copy the new configuration file into the `milvus-standalone` container.

```console
docker cp milvus.yaml <container_id>:milvus/configs/
```
   >Note: You can get the container id by running `docker ps`

### Apply configurations to docker compose

Milvus provides this guide on how to add a `milvus.yaml` configuration file to the milvus-standalone container in the `docker-compose.yml`. -> [https://milvus.io/docs/configure-docker.md](https://milvus.io/docs/configure-docker.md)


## Accessing Milvus

Now that Milvus is installed, you have a few options of what to do next.

### Run the GUI (Attu)

Milvus doesn't come with a GUI but there is a service called Attu which can easily provide a front end. The documentation and git repo is here: [https://github.com/zilliztech/attu](https://github.com/zilliztech/attu).

```
docker run -p 8000:3000 -e MILVUS_URL={<ENTER IP ADDRESS>}:19530 zilliz/attu:v2.2.6
```

### Connecting via an SDK

Install the pymilvus SDK either on the VSI or your local machine to access. https://milvus.io/docs/install-pymilvus.md

## Troubleshooting

There is a fix required for version 3.3 of Milvus. Open the docker-compose.yml file in vi (or equivalent editor):

```
vi docker-compose.yml
```

Replace line 16 with test: `["CMD", "etcdctl", "endpoint", "health"]` and save the file.
