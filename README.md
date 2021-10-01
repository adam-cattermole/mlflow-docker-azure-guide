# Getting started with MLflow on Azure

In this blog post we aim to introduce MLflow, deploy on Azure using docker-compose, and run a simple instrumented example model.

## Why MLflow

_TODO_

## Architecture

A typical MLflow deployment consists of the tracking server, a backend store to maintain results, and an artifact store that will contain larger objects from the end of the run.

* **Backend store**: will often be a database of some kind, so long as the URL can be provided as an [SQLAlchemy database URL](https://docs.sqlalchemy.org/en/14/core/engines.html#database-urls)
* **Artifact store**: contains larger objects and supports many cloud provider solutions (Amazon S3, Azure Blob, ...) as well as alternate remote storage solutions (FTP, NFS, HDFS, ...)



> _**NOTE**_: In this tutorial we will deploy the MySQL database on the same VM as the tracking server, however in production environments it may be beneficial to deploy the database in a more scalable fashion.

* _diagram of mlflow deployed in azure / docker_

## Deployment

> _**NOTE**_: These deployment steps were accurate at the time of writing, but some installation steps for dependencies may have changed. Please refer to respective websites for up to date installation instructions.

The focus of this section is to deploy the MLflow tracking server to a virtual machine running in Azure, leveraging a MySQL database as the backend store, and Azure blob storage as the artifact store. We also secure the tracking server with basic-auth using Traefik, providing SSL certificates to encrypt communication.

To simplify the deployment process some helper scripts have been provided at [adam-cattermole/mlflow-docker-azure](https://github.com/adam-cattermole/mlflow-docker-azure).

### Create VM

Firstly we must deploy a VM that our tracking server and MySQL database will run on. In this example we will use a relatively lightweight VM that is expected to have high uptime with few concurrent users. We therefore selected a Standard B2s (2 vcpus, 4 GiB memory) VM running Ubuntu 20.04, however this should be chosen as appropriate for your use case.

It is useful to configure the VM with a custom DNS name. It may be necessary to shutdown the VM and detach the network interface temporarily to configure this. In our example we have configured the DNS name as `mlflow-tracking.eastus.cloudapp.azure.com`.

* _image of deployed VM_

### Configure Environment

For the deployment of various components by the helper scripts, we must set a series of environment variables.

First we must clone the repository:

```bash
git clone https://github.com/adam-cattermole/mlflow-docker-azure.git
cd mlflow-docker-azure
```

The [configure-env.sh](https://github.com/adam-cattermole/mlflow-docker-azure/blob/main/configure-env.sh#L3-L18) file provides a set of variables at the top of the file which must be populated. Example values are provided below. Note that the `AZURE_STORAGE_ACCESS_KEY` is left blank as we set this once the storage account has been created:

```bash
### Set these variables to be written to .bashrc/stdout

# MYSQL
MYSQL_USER="root"
MYSQL_PASSWORD="root"

# MLFLOW
MLFLOW_TRACKING_USERNAME="admin"
MLFLOW_TRACKING_HOSTNAME="mlflow-tracking.eastus.cloudapp.azure.com"

# AZURE
AZURE_STORAGE_ACCESS_KEY=""
AZURE_STORAGE_ACCOUNT="mlflowsa"
AZURE_STORAGE_CONTAINER="mlflow"
AZURE_RESOURCE_GROUP="mlflow-rg"
AZURE_RESOURCE_GROUP_LOCATION="eastus"
```

Once we have made the changes to this file, we can `echo` the variables to the prompt using:

```bash
./configure-env.sh list [password]
```

Or export them directly to `~/.bashrc`:

```bash
./configure-env.sh write [password]
```

The value provided for `[password]` is used to log in to the tracking server.

> _**NOTE**_: Due to a bug in MLflow, the `password` cannot contain symbols when using Mlflow with Docker (as described in a future section) as the `password` string is not surrounded in quotes when passed to the container. Unfortunately the [relevant bug](https://github.com/mlflow/mlflow/issues/3381) (and [provided fix](https://github.com/harupy/mlflow/commit/a4f23cfb280477528d9de0a2f1d141ad25687b9d)) never made it into a release.

Once these have been added to the shell, we can enable these changes by logging out and in again, or by running:

```bash
source ~/.bashrc
```

### Create Blob Storage

Now that the environment variables have been set, we can create the storage account. This can be performed manually, but we will use the helper script provided.

We must first install the [Azure Command Line Interface](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli). At the time of writing this could be done using the following single command:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Once installed we can login:

```bash
az login
```

We can deploy the required resources using:

```bash
./create-azure-sa.sh create_sa
```

This command will write out the `AZURE_STORAGE_ACCESS_KEY` export command to add to your environment. It is also possible to retrieve this later by either re-running the command above, or through the azure portal. The command will only create the resources when it does not find them present with the same name. It is worth updating the `AZURE_STORAGE_ACCESS_KEY` environment variable in your shell configuration `~/.bashrc` and reloading.

```bash
source ~/.bashrc
```

### Install Dependencies

The deployment scripts rely on Docker and Docker Compose and so these must also be installed and configured on the tracking VM.

#### Install Docker

Most configuration is now complete, and so we must install Docker. The following is an excerpt of the commands used. Up to date installation instructions can be found [here](https://docs.docker.com/engine/install/ubuntu/).

> ##### Set up the repository
> 1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:
> ```bash
> sudo apt-get update
> sudo apt-get install \
>    apt-transport-https \
>    ca-certificates \
>    curl \
>    gnupg \
>    lsb-release
>```
> 2. Add Docker’s official GPG key:
>```bash
>curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
>```
> 3. Use the following command to set up the stable repository. To add the nightly or test repository, add the word nightly or test (or both) after the word stable in the commands below. Learn about nightly and test channels.
>```bash
> echo \
>  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
>  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
>```
> ##### Install Docker Engine
> 1. Update the apt package index, and install the latest version of Docker Engine and containerd, or go to the next step to install a specific version:
> ```bash
> sudo apt-get update
> sudo apt-get install docker-ce docker-ce-cli containerd.io
> ```
> ##### Manage Docker as a non-root user
> 1. Create the docker group.
> ```bash
> sudo groupadd docker
> ```
> 2. Add your user to the docker group.
> ```bash
>  sudo usermod -aG docker $USER
> ```
> 3. Log out and log back in so that your group membership is re-evaluated. </br>
> ...</br>
> On Linux, you can also run the following command to activate the changes to groups:
> ```bash
>  newgrp docker 
> ```

#### Install Docker Compose

Up to date instructions can be found [here](https://docs.docker.com/compose/install/). The following commands were accurate at the time of writing.

> Run this command to download the current stable release of Docker Compose:
>
> ```bash
> sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
> ```
> To install a different version of Compose, substitute 1.29.2 with the version of Compose you want to use.
>
> If you have problems installing with curl, see Alternative Install Options tab above.
>
> Apply executable permissions to the binary:
>
>```bash
> sudo chmod +x /usr/local/bin/docker-compose
> ```
<!-- > Note: If the command docker-compose fails after installation, check your path. You can also create a symbolic link to /usr/bin or any other directory in your path.
>
> For example:
>
> ```bash
> sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
> ``` -->

### Run the Server

Once we have docker running and all of the required environment variables in the shell, we can test deploying the resources. All of the required configuration is provided within the [docker-compose.yaml](https://github.com/adam-cattermole/mlflow-docker-azure/blob/main/docker-compose.yml) file, extracting the values set in the environment.

Simply run:

```bash
docker-compose up --build
```

This:

* Creates the MySQL container with a default database named 'mlflow'
* Starts a traefik container to route connections, creating the required certificates for the DNS name provided
* Launches the tracking server communicating with the local MySQL instance as well as the remote artifact store

### Expose the server

We need to expose the VM on port `443` so that we can access it via `HTTPS`. This can easily be done through azure portal, navigating through: VM -> Networking -> Inbound port rules -> Add inbound port rule

* *image of the networking rules in the dashboard*
* *image of the rule itself and settings*

## Running our first Model

* grab simple example from mlflow website
* Talk through instrumenting the code


### Dependencies on the running machine

* python
* mlflow
* azure-storage-blob
* paramiko??
* env variables

#### Configure Project

To set up a project in mlflow we need to create the MLproject file. This can be used to define operating conditions and

Run the model:

```bash
mlflow run ...
```




Note that whilst the MLflow model is stored remotely in the tracking server, the execution of the model is performed on the local calling machine, wrapped by requests to the tracking server as well as direct interaction with the artifact store. In some cases it may be beneficial to deploy the code to a remote server, especially when leveraging a GPU-enabled machine.

### Using Docker for Remote Model Deployment

#### Updating the Configuration

* add docker_env to MLproject file
* expose env username/password

A simple workaround is to provide the `DOCKER_HOST` environment variable exported to your shell. Docker automatically uses this variable for all docker-related commands.

```bash
export DOCKER_HOST="ssh://azureuser@mlflow-training.eastus.cloudapp.azure.com
```

As MLflow uses the docker command directly underneath, setting this variable ensures that an MLflow Docker run will be performed on the remote training machine we have configured.

```bash
mlflow run . -A gpus=all
```

## Conclusion



## Notes

* docker-compose
* paramiko
* azure-storage-blob