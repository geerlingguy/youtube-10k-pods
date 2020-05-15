# K3s Attempt

This directory contains automation to build a K3s master and 100 node Auto Scaling Group (ASG) in AWS, install K3s, and run 10,000 Pods on the cluster.

To run the Ansible playbook, you need to ensure the following:

  - You have [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed. Also make sure to install the `boto` and `openshift` Python libraries.
  - You have an AWS account, and you have at least 200 available vCPUs in EC2 (so you can run 100 c5.large instances).
  - You have an AWS account IAM profile's credentials in `~/.aws/credentials`, and that account has pretty much admin-level permissions (manage VPCs, EC2 instances, ASGs, etc.).
  - You have a Key configured named `jeffgeerling_aws` which you'll use to connect to instances. If you don't have a key named that, you'll have to edit the `asg.yml` CF template... or maybe I'll make it more flexible and configurable someday.

## Configure things

You should update, at a minimum, the `aws_region` and `aws_profile` in `vars` in `main.yml`, assuming you're not me.

## Build things

Run the Ansible playbook:

    ansible-playbook main.yml

## Set up K3s Inventory

The playbook should've cloned `k3s-ansible` to this directory. You need to add all the servers just built to it's inventory file.

### Get the Master IP

Get the EC2 master node's IP address:

```
aws --profile jeffgeerling --region us-east-1 ec2 describe-instances \
  --filters "Name=tag:Name,Values=k3s-master" \
  "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[PublicIpAddress]" \
  --output=text
```

### Get the Nodes' IPs

Get a list of all the EC2 ASG nodes using AWS CLI or whatever other means:

```
aws --profile jeffgeerling --region us-east-1 ec2 describe-instances \
  --filters "Name=tag:Name,Values=k3s-node" \
  "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[PublicIpAddress]" \
  --output=text
```

### Add the IPs to the K3s Ansible playbook inventory

Edit the `k3s-ansible/inventory/hosts.ini` file, adding this list under the `node` group.

Finally, edit the `k3s-ansible/inventory/group_vars/all.yml` file and add:

  - `k3s_version: v1.17.5+k3s1` (update existing variable)
  - `ansible_user: admin` (update existing variable)
  - `ansible_ssh_common_args: '-o StrictHostKeyChecking=no'`
  - `ansible_ssh_private_key_file: ~/.ssh/jeffgeerling_aws.pem` (the local path to your Key File)

## Deploy the K3s

In the `k3s-ansible` directory, run:

    ansible-playbook site.yml -i inventory/hosts.ini

## Deploy the Pods

SSH into the master node:

    ssh admin@MASTER_IP_ADDRESS

Switch to the root account:

    sudo su

Verify all 100 nodes are up:

    # kubectl get nodes | grep Ready | tail -n +2 | wc -l
    100

Deploy the hello-kubernetes deployment:

    kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4

Edit the deployment:

    kubectl edit deployment hello-node

Change the `replicas` value. Set it to `10000`. And wait...

## Watch the rollout

    watch "kubectl get pods | wc -l"

Eventually, the number should reach `10000`. Once it does, check for whether the pods are all `Running`:

    kubectl get pods | grep Running | wc -l

This should also match `10000`.

Plot twist: When I tried this with free-tier `t3.micro` instances, after getting up to about `7100` running pods, the numbers started diminishing as nodes started falling off into `NotReady` state. The reason? T3 burstable instances were absolutely _destroying_ burst CPU credit, and they all started going offline about the same time as they were severely limited once they tried starting up 100 containers each.

I had to bump the instances to `c5.large` so they'd have the sustained CPU power to not die when they started getting saturated with pods. And I had to bump the master to `c5.2xlarge`, and even then, `k3s` hit 700-800% CPU usage while the pods were getting deployed!

Moral of that story? Don't run major production-scale workloads on burstable AWS instances. Youch.

Yay. Now... sleep. This took way too long. I was going to automate everything from top to bottom but sometimes you just gotta know when to throw in the towel.
