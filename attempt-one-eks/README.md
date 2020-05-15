# EKS Attempt

This directory contains automation to build a 100 node EKS Kubernetes cluster in AWS, and attempts to run 10,000 Pods on that cluster.

To run the Ansible playbook, you need to ensure the following:

  - You have [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed. Also make sure to install the `boto` and `openshift` Python libraries.
  - <s>You have an AWS account, and you have at least 200 available vCPUs in EC2 (so you can run 100 t3.micro instances).</s>
    - Scratch that: t3.micro instances in EKS can only run 4 pods per instance due to VPC CNI networking limitations. Ouch. So, instead, you need to have at least 900 available vCPUs in EC2 (so you can run 14 m5.16xlarge instances).
  - You have an AWS account IAM profile's credentials in `~/.aws/credentials`, and that account has pretty much admin-level permissions (manage VPCs, EC2 instances, EKS clusters and Node Groups, etc.).
  - You have the patience to wait 15+ minutes for AWS to create an EKS cluster for you.

## Configure things

You should update, at a minimum, the `aws_region` and `aws_profile` in `vars/main.yml`, assuming you're not me.

## Build things

Run the Ansible playbook:

    ansible-playbook main.yml

## Check things

Get your kubeconfig set up for EKS:

    $ aws eks --region us-east-1 --profile jeffgeerling update-kubeconfig --name eks-example --kubeconfig ~/.kube/eks-example
    $ export KUBECONFIG=~/.kube/eks-example

Verify all the nodes are running:

    # kubectl get nodes | grep Ready | tail -n +2 | wc -l
    100

Watch the pods as they roll out:

    watch "kubectl get pods | wc -l"

Verify that all 10,000 pods are running:

    $ kubectl get pods | grep Running | wc -l
    10000

Yay!
