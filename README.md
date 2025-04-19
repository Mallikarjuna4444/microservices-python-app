# Devops Project: video-converter
Converting mp4 videos to mp3 in a microservices architecture.

## Architecture

<p align="center">
  <img src="./Project documentation/ProjectArchitecture.png" width="600" title="Architecture" alt="Architecture">
  </p>

This diagram outlines an architecture for a video-to-audio conversion service, possibly used in a web or mobile app. Here's a step-by-step breakdown of how the system works:

---

### 🔁 **Workflow Description:**

1. **User Request:**
   - A user sends a request to the system (e.g., to log in, upload a video, or download audio).

2. **Login (`/login` endpoint):**
   - The request hits an API Gateway and routes to the `/login` API.
   - User credentials are validated.
   - On success, a **JWT token** is returned and user data is stored in a **PostgreSQL** database.

3. **Upload Video (`/upload` endpoint):**
   - The user uploads a video via `/upload`.
   - The video is stored temporarily (perhaps on local storage or a blob store).
   - The video is also sent to a **RabbitMQ video queue** for further processing.

4. **Video Processing:**
   - A **Converter** service picks up video tasks from the **video queue**.
   - It converts the video into MP3 format.
   - The MP3 file is stored in **MongoDB**.
   - Once completed, a message is sent to the **mp3 queue**.

5. **Download Audio (`/download` endpoint):**
   - When the user accesses `/download`, the system fetches the MP3 file from **MongoDB**.
   - The file is then served back to the user.

6. **Notifications:**
   - A notification service sends **push notifications** to the user once the MP3 is ready.
   - It can also send an **email** to notify them of completion.

---

### 🧰 **Technologies Involved:**

- **API Gateway**: Manages routing of requests.
- **PostgreSQL**: Stores user credentials and login data.
- **RabbitMQ**: Handles async messaging via video/mp3 queues.
- **MongoDB**: Stores the final MP3 files.
- **Microservices (API/Converter/Notification)**: Handle specific endpoints and tasks.

Yes, exactly! The **notification service** is designed to **listen** to the **mp3 queue**, and once it **receives a message** (which means an MP3 file is ready), it triggers the notification process.

---

### 🔔 Here's how it typically works step-by-step:

1. ✅ **Converter finishes processing**
   - Converts video to MP3
   - Stores the MP3 in MongoDB (or blob storage)
   - Sends a message to the **mp3 queue** with details like:
     ```json
     {
       "user_id": "12345",
       "file_id": "abcde.mp3",
       "status": "completed",
       "timestamp": "2025-04-19T12:00:00Z"
     }
     ```

2. 📬 **Notification service listens** to the mp3 queue:
   - It's like a subscriber that’s constantly waiting for new messages.

3. 📣 **On message arrival**, the notification service:
   - Parses the message
   - Looks up the user’s contact info (email, push token, etc.)
   - Sends:
     - **Push notification** ("Your audio is ready to download!")
     - **Email** (with a download link or status update)

4. 🧾 (Optional) It might also log the notification or update user history in a database.

---

### 💡 Why this is awesome:
- **Asynchronous**: User doesn’t need to wait.
- **Reliable**: If notification fails, you can retry.
- **Modular**: You can add more post-processing (e.g., analytics, logs) without touching the converter.

---

So yes, the notification service acts **only after** it receives a message from the **mp3 queue** — it’s the green light to notify the user that their file is ready.

---

## Deploying a Python-based Microservice Application on AWS EKS

### Introduction

This document provides a step-by-step guide for deploying a Python-based microservice application on AWS Elastic Kubernetes Service (EKS). The application comprises four major microservices: `auth-server`, `converter-module`, `database-server` (PostgreSQL and MongoDB), and `notification-server`.

### Prerequisites

Before you begin, ensure that the following prerequisites are met:

1. **Create an AWS Account:** If you do not have an AWS account, create one by following the steps [here](https://docs.aws.amazon.com/streams/latest/dev/setting-up.html).

2. **Install Helm:** Helm is a Kubernetes package manager. Install Helm by following the instructions provided [here](https://helm.sh/docs/intro/install/).

3. **Python:** Ensure that Python is installed on your system. You can download it from the [official Python website](https://www.python.org/downloads/).

4. **AWS CLI:** Install the AWS Command Line Interface (CLI) following the official [installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

5. **Install kubectl:** Install the latest stable version of `kubectl` on your system. You can find installation instructions [here](https://kubernetes.io/docs/tasks/tools/).

6. **Databases:** Set up PostgreSQL and MongoDB for your application.

### High Level Flow of Application Deployment

Follow these steps to deploy your microservice application:

1. **MongoDB and PostgreSQL Setup:** Create databases and enable automatic connections to them.

2. **RabbitMQ Deployment:** Deploy RabbitMQ for message queuing, which is required for the `converter-module`.

3. **Create Queues in RabbitMQ:** Before deploying the `converter-module`, create two queues in RabbitMQ: `mp3` and `video`.

4. **Deploy Microservices:**
   - **auth-server:** Navigate to the `auth-server` manifest folder and apply the configuration.
   - **gateway-server:** Deploy the `gateway-server`.
   - **converter-module:** Deploy the `converter-module`. Make sure to provide your email and password in `converter/manifest/secret.yaml`.
   - **notification-server:** Configure email for notifications and two-factor authentication (2FA).

5. **Application Validation:** Verify the status of all components by running:
   ```bash
   kubectl get all
   ```

6. **Destroying the Infrastructure** 


### Low Level Steps

#### Cluster Creation

1. **Log in to AWS Console:**
   - Access the AWS Management Console with your AWS account credentials.

2. **Create eksCluster IAM Role**
   - Follow the steps mentioned in [this](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html) documentation using root user
   - After creating it will look like this:

   <p align="center">
  <img src="./Project documentation/ekscluster_role.png" width="600" title="ekscluster_role" alt="ekscluster_role">
  </p>

   - Please attach `AmazonEKS_CNI_Policy` explicitly if it is not attached by default

3. **Create Node Role - AmazonEKSNodeRole**
   - Follow the steps mentioned in [this](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role) documentation using root user
   - Please note that you do NOT need to configure any VPC CNI policy mentioned after step 5.e under Creating the Amazon EKS node IAM role
   - Simply attach the following policies to your role once you have created `AmazonEKS_CNI_Policy` , `AmazonEBSCSIDriverPolicy` , `AmazonEC2ContainerRegistryReadOnly`
     incase it is not attached by default
   - Your AmazonEKSNodeRole will look like this: 

<p align="center">
  <img src="./Project documentation/node_iam.png" width="600" title="Node_IAM" alt="Node_IAM">
  </p>

4. **Open EKS Dashboard:**
   - Navigate to the Amazon EKS service from the AWS Console dashboard.

5. **Create EKS Cluster:**
   - Click "Create cluster."
   - Choose a name for your cluster.
   - Configure networking settings (VPC, subnets).
   - Choose the `eksCluster` IAM role that was created above
   - Review and create the cluster.

6. **Cluster Creation:**
   - Wait for the cluster to provision, which may take several minutes.

7. **Cluster Ready:**
   - Once the cluster status shows as "Active," you can now create node groups.

#### Node Group Creation

1. In the "Compute" section, click on "Add node group."

2. Choose the AMI (default), instance type (e.g., t3.medium), and the number of nodes (attach a screenshot here).

3. Click "Create node group."

#### Adding inbound rules in Security Group of Nodes

**NOTE:** Ensure that all the necessary ports are open in the node security group.

<p align="center">
  <img src="./Project documentation/inbound_rules_sg.png" width="600" title="Inbound_rules_sg" alt="Inbound_rules_sg">
  </p>

#### Enable EBS CSI Addon
1. enable addon `ebs csi` this is for enabling pvcs once cluster is created

<p align="center">
  <img src="./Project documentation/ebs_addon.png" width="600" title="ebs_addon" alt="ebs_addon">
  </p>

#### Deploying your application on EKS Cluster

1. Clone the code from this repository.

2. Set the cluster context:
   ```
   aws eks update-kubeconfig --name <cluster_name> --region <aws_region>
   ```

### Commands

Here are some essential Kubernetes commands for managing your deployment:


### MongoDB

To install MongoDB, set the database username and password in `values.yaml`, then navigate to the MongoDB Helm chart folder and run:

```
cd Helm_charts/MongoDB
helm install mongo .
```

Connect to the MongoDB instance using:

```
mongosh mongodb://<username>:<pwd>@<nodeip>:30005/mp3s?authSource=admin
```

### PostgreSQL

Set the database username and password in `values.yaml`. Install PostgreSQL from the PostgreSQL Helm chart folder and initialize it with the queries in `init.sql`. For PowerShell users:

```
cd ..
cd Postgres
helm install postgres .
```

Connect to the Postgres database and copy all the queries from the "init.sql" file.
```
psql 'postgres://<username>:<pwd>@<nodeip>:30003/authdb'
```

**Login to pod and execute the query present in init.sql file**

To log into a PostgreSQL pod running in Kubernetes, you'll usually go through these steps:


### 🚪 1. **Access the Pod**
```bash
kubectl exec -it <postgres-pod-name> -- bash
```
> Replace `<postgres-pod-name>` with the actual pod name.

---

### 🐘 2. **Login to Postgres**
Once inside the pod, use the `psql` command:
```bash
psql -U <username> -d <database>
```
- `<username>`: Your Postgres user (e.g. `postgres`)
- `<database>`: Name of the DB you want to connect to (e.g. `postgres`)


### RabbitMQ

Deploy RabbitMQ by running:

```
helm install rabbitmq .
```

Ensure you have created two queues in RabbitMQ named `mp3` and `video`. To create queues, visit `<nodeIp>:30004>` and use default username `guest` and password `guest`

**NOTE:** Ensure that all the necessary ports are open in the node security group.

### Apply the manifest file for each microservice:

- **Auth Service:**
  ```
  cd auth-service/manifest
  kubectl apply -f .
  ```

- **Gateway Service:**
  ```
  cd gateway-service/manifest
  kubectl apply -f .
  ```

- **Converter Service:**
  ```
  cd converter-service/manifest
  kubectl apply -f .
  ```

- **Notification Service:**
  ```
  cd notification-service/manifest
  kubectl apply -f .
  ```

### Application Validation

After deploying the microservices, verify the status of all components by running:

```
kubectl get all
```

### Notification Configuration



For configuring email notifications and two-factor authentication (2FA), follow these steps:

1. Go to your Gmail account and click on your profile.

2. Click on "Manage Your Google Account."

3. Navigate to the "Security" tab on the left side panel.

4. Enable "2-Step Verification."

5. Search for the application-specific passwords. You will find it in the settings.

6. Click on "Other" and provide your name.

7. Click on "Generate" and copy the generated password.

8. Paste this generated password in `notification-service/manifest/secret.yaml` along with your email.

Run the application through the following API calls:

# API Definition

- **Login Endpoint**
  ```http request
  POST http://nodeIP:30002/login
  ```

  ```console
  curl -X POST http://nodeIP:30002/login -u <email>:<password>
  ``` 
  Expected output: success!

- **Upload Endpoint**
  ```http request
  POST http://nodeIP:30002/upload
  ```

  ```console
   curl -X POST -F 'file=@./video.mp4' -H 'Authorization: Bearer <JWT Token>' http://nodeIP:30002/upload
  ``` 
  
  Check if you received the ID on your email.

- **Download Endpoint**
  ```http request
  GET http://nodeIP:30002/download?fid=<Generated file identifier>
  ```
  ```console
   curl --output video.mp3 -X GET -H 'Authorization: Bearer <JWT Token>' "http://nodeIP:30002/download?fid=<Generated fid>"
  ``` 

## Destroying the Infrastructure

To clean up the infrastructure, follow these steps:

1. **Delete the Node Group:** Delete the node group associated with your EKS cluster.

2. **Delete the EKS Cluster:** Once the nodes are deleted, you can proceed to delete the EKS cluster itself.
