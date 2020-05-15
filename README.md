# 10,000 Kubernetes Pods for 10,000 Subscribers

<p align="center"><a href="https://www.youtube.com/watch?v=TODO_VIDEO_ID"><img src="https://img.youtube.com/vi/TODO_VIDEO_ID/0.jpg" alt="10,000 Kubernetes Pods for 10,000 Subscribers!" /></a></p>

> **tl;dr**: See the blog post that accompanies this project: [10,000 Kubernetes Pods for 10,000 Subscribers](https://www.jeffgeerling.com/blog/2020/10000-kubernetes-pods-10000-subscribers).

This repository contains automation to build a large Kubernetes cluster in AWS, and run 10,000 Pods on that cluster. And it does it two ways, because I realized that the first way didn't work without running 1,000 vCPUs in my brand new AWS account (AWS support usually doesn't look kindly on people who open a new account and immediately ask for insane capacity increases!).

  - [`attempt-one-eks`](attempt-one-eks/) is the first attempt: build an EKS cluster with 100 `t3.micro` nodes; later updated to use 14 `m5.16xlarge` nodes, which worked but required an insane amount of computing power, which would cost over $30,000/month!
  - [`attempt-two-k3s`](attempt-two-k3s/) is the second attempt: build a K3s cluster with one `c5.2xlarge` master and 100 `c5.large` nodes. (I almost got it working with `t3.micro` nodes but they started dying when I deployed 100 Pods to each node...).

In the end, I found out that burstable `t3` instances just aren't ready for massive amounts of pods, no matter what. I run into networking _and_ burst CPU limits. And EKS has some annoying limitations with it's current VPC CNI networking, but those could be overcome if you take the time to swap out a different networking solution.

If you're interested in automating Kubernetes with Ansible, I have the _perfect_ book for you: [Ansible for Kubernetes](https://www.ansibleforkubernetes.com).
