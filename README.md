# Fleetman Project

This project creates a Kubernetes cluster using kops in an AWS EC2 t2.medium instance with 6 pods. 

## The pods include:

- Position Simulator: A service that simulates the positions of vehicles in a fleet. It generates random GPS coordinates and sends them to the Position Tracker service.

- MongoDB: A NoSQL database used to store the positions of the vehicles.

- Position Tracker: A service that receives position updates from the Position Simulator service and stores them in the MongoDB database.

- Queue: A message queue used to decouple the Position Tracker service from the Position 

- Simulator service. It stores the position updates until the Position Tracker service is ready to process them.

## Requirements

To run this project, you'll need the following tools:

- kops 
`https://github.com/kubernetes/kops/blob/master/docs/install.md` 
- kubectl
`https://github.com/kubernetes/kops/blob/master/docs/install.md` Search for "other dependencies"
- AWS CLI
`https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md`

You'll also need an AWS account with the following permissions:

- EC2
- Route 53
- IAM
- S3

## Getting Started

To create the Kubernetes cluster, follow these steps:

### Set up the environment:

- If your local machine is running on windows or mac, create a Linux EC2 t2.medium instance on AWS (I recommend Cygwin as a CLI for Windows)

- Download the key pair and copy it to your current working directory

- Check that you security group enables Port 22 (SSH) inbound traffic 

- Select connect and run the command that appears on SSH client tab

Example:

`ssh -i "fleetman-kp.pem" ec2-user@ec2-44-213-118-88.compute-1.amazonaws.com`

- Run `aws configure` to be able to create users and groups (using a key pair from your user)

### Install kops

`curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64`

`chmod +x kops`

`sudo mv kops /usr/local/bin/kops`

To test, run:

`kops`

### Install kubectl

1. Download the latest release with the command:

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`

2. Download the kubectl checksum file:

`curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"`

3. Validate the kubectl binary against the checksum file:

`echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check`


- `sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`

- Test to ensure the version you installed is up-to-date:

`kubectl version --client`

### Create the kops IAM user from the command line using the following:

- `aws iam create-group --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess --group-name kops`

- `aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess --group-name kops`

- `aws iam create-user --user-name kops`

- `aws iam add-user-to-group --user-name kops --group-name kops`

- `aws iam create-access-key --user-name kops`

To test, run:

- `aws iam list-users` 

14. Because "aws configure" doesn't export these vars for kops to use, we export them now

- `export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)`

- `export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)`

### DNS:

- If you bought your domain with AWS, then you should already have a hosted zone in Route53. 

If you own `example.com`, your records for Kubernetes would look like 

`etcd-us-east-1c.internal.clustername.example.com`

- To test your DNS:

`dig ns subdomain.example.com`

### Create an S3 bucket to store the cluster configuration:

1. `aws s3api create-bucket --bucket <bucket-name> --region <region> --create-bucket-configuration LocationConstraint=<region>`

### Create the Kubernetes cluster:

1. Check availability zones:

`aws ec2 describe-availability-zones --region us-west-2`

2. Export the S3 bucket name as an environment variable:

`export KOPS_STATE_STORE=s3://<bucket-name>`

`NAME=prefix.fleetman.com`

3. Create the cluster:

`kops create cluster  --name ${NAME} --cloud aws --zones us-west-2a,us-west-2b,us-west-2c>`

# Set admin context - If this isn't set up, you'll have an `unauthorized error` during validation

`kops export kubecfg ${NAME} --admin=87600h` 

3. Edit the cluster configuration to add the Position Simulator, MongoDB, Position Tracker, and Queue nodes:

`kops edit ig nodes --name ${NAME}`

4. Create the following YAML files in the current working directory:

`nano storage-aws.yaml`

Add storage-aws.yaml content

`mongo-stack.yaml`

Add mongo-stack.yaml content

`workloads.yaml`

Add workloads.yaml content

`services.yaml`

Add services.yaml content

5. Then run:

`kubectl apply -f .`

6. Update the cluster configuration:

`kops update cluster --name ${NAME} --state ${KOPS_STATE_STORE} --yes`

7. Wait for the cluster to be ready:

`kops validate cluster --name ${NAME} --state ${KOPS_STATE_STORE}`

## Usage

Once the cluster is ready, you can use the Kubernetes CLI (kubectl) to interact with the nodes:

- To list all the system components:

`kubectl -n kube-system get po`

## Wanna go for a walk? 

1. Run:

`kops delete cluster --name ${NAME}`

2. Click 'Stop' on your AWS instance

When you come back, restart your instance and recreate your cluster

Run this command to search for env. variables:

`history | grep export`

And run the command with a `!` before its number:

`!command number`

