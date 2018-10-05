AutoScalr For EKS Tutorial (Preview)
=========================

This tutorial walks through using AutoScalr for EKS to achieve:
- Simplicity of Fargate scaling, 
but implemented on top of EC2s, so it can be run today before Fargate for EKS is available
- Cost-Optimization yielding 50-80% lower operational cost via:
    - Dynamic Instance Right-Sizing
    - Blended Spot Instance Usage
    

Requirements
------------

-	Access to an [AWS Account](https://aws.amazon.com) 
-	[eksctl](https://eksctl.io) (or build an EKS cluster with your favorite toolset)
-	[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 

Resource Utilization
---------------------

This tutorial requires running EKS with worker nodes that vary from 2 to a dozen or so.  
This will be outside the free tier and you will incur some AWS cost to go through it, but it should only run for a period
of an hour or less, so as long as you delete the EKS cluster when you are done, the total costs should be no more than a dollar or two.


Tutorial Steps
---------------------

### 1. Create a new EKS Cluster

Use eksctl (or your preferred toolset) to build a new EKS cluster

```sh
$ eksctl create cluster --name=eks-autoscalr --nodes=5 --node-type=m4.large --nodes-min=2 --nodes-max=20 --region=us-west-2
```

It will take ~15 minutes for the EKS cluster to be initialized, so grab a cup of coffee or check your emails.

### 2. Verify kubectl access to cluster

Once the cluster is up and running, verify kubectl access is properly configured by:


```sh
$ kubectl get nodes
```

The output should show a couple of nodes in the cluster:
```sh
$ kubectl get nodes
NAME                                           STATUS    ROLES     AGE       VERSION
ip-192-168-165-88.us-west-2.compute.internal   Ready     <none>    2m        v1.10.3
ip-192-168-238-93.us-west-2.compute.internal   Ready     <none>    2m        v1.10.3
```

### 3. Install Kubernetes Dashboard (Optional)

You can use kubectl for the entire tutorial, but it is quicker / easier to change scaling parameters via the dashboard.

Install the Kubernetes Dashboard with this yaml file in this repo: 

```sh
$ kubectl apply -f kubernetes-dashboard.yaml
```

Start a local proxy to access the dashboard (it is blocking, so put in a separate shell or put in background)
```sh
$ kubectl proxy
```
Access the dashboard via the proxy at

[http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)

You should be able to login by clicking Skip

For more options or troubleshooting, see: https://github.com/kubernetes/dashboard/blob/master/README.md

### 4. Install Example CPU and Memory task deployments

These will be used to simulate CPU and Memory intenstive workloads in the EKS cluster:

```sh
$ kubectl apply -f cpu-intensive-app.yaml
$ kubectl apply -f memory-intensive-app.yaml
```

Those deployment files will start 10 pods each.
Verify they are running either via the dashboard under Deployments for kube-public or via kubectl:

```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      10        10        10           10          46s
memory-intensive-app   10        10        10           10          49s
```

### 5. Configure AutoScalr for the EKS Cluster

- Login to your [AutoScalr](https://app.autoscalr.com/authentication/signin) account
  - or signup for free trial [AutoScalr Free Trial](https://aws.amazon.com/marketplace/pp/B074N1N5QM)
  - For a video walk-through of the singup, watch this [AutoScalr Signup Video](https://youtu.be/yxgwUdmmL1I)
- Select Kubernetes -> Add New
- Follow the wizard steps:
  - Select the Region and Auto Scaling Group Name for the worker nodes of the cluster
  - Specify Instance Types of m4.large, c4.large, r4.large
  - Leave Availability Zones as is
  - Set both Spot Parameter sliders to 0 (we will come back to these later) and click Next
  - Leave the default Scaling parameters, and rename to something simple like AutoScalr-EKS
  - Update the IAM Role for the worker nodes to allow read access to the Auto Scaling Group as described
    - Update the PolicyTagDiscovery inline policy
  - Take the default Kubernetes version (1.10), and download the AutoScalr.yaml generated file that contains
   all the settings you have specified so far.
  - Deploy AutoScalr to the EKS cluster as described by

```sh
$ kubectl apply -f ~/Downloads/AutoScalr.yaml
```
After 30 seconds to a minute, refresh the AutoScalr UI and the EKS cluster should appear under the Kubernetes section.

### 6. Scale the CPU and Memory deployments

For this section we will be changing the number of replicas of the CPU and Memory deployments and 
watching how the autoscaling reacts.  The simplest way to change the number of replicas is in the Kubernetes Dashboard, 
go to the Deployments for the kube-public namespace, select the deployment of interest, and click Scale in the top right. 
Alternatively, you can edit the cpu-intensive-app.yaml and memroy-intensive-app.yaml files and apply them with kubectl to 
accomplish the same thing. For checking the status of deployments, kubectl provides faster feedback to watch the containers
starting, but you can also monitor via the dasbhoard if you prefer.  With kubectl it is simply:
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      10        10        10           10          46s
memory-intensive-app   10        10        10           10          49s
```
The DESIRED shows the target count of pods for the deployment, and AVAILABLE is the number that are up and running.  As we make changes 
we will watch how fast the AVAILABLE count reaches the DESIRED state for different scaling scenarios.

#### Scaling Out Within Spare Capacity

If spare capacity is set correctly, the vast majority of times scaling a deployment should be very fast. 
The new pods should start up in just a few seconds. Then if necessary, in the background AutoScalr replensihes the 
spare capacity to be ready for other scale out events.

- Change the number of replicas for the memory-intensive-app deployment to 13.
- Watch the response via kubectl, within a few seconds the AVAILABLE count should reach 13:
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      10        10        10           10          1m
memory-intensive-app   13        12        13           13          1m
```

- Repeat the process for the cpu-intensive-app deployment, changing scale to 13.
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      13        13        13           13          2m
memory-intensive-app   13        13        13           13          2m
```

All 12 pods for both deployments are running and available, but lets check what AutoScalr is doing in the background.
- In the AutoScalr web UI, go to the Metrics for Kubernetes cluster
- Note that AutoScalr detected that the target spare capacity is now below the target and triggered adding a new EC2 instance
to the cluster to restore the specified spare capacity.
  - Go to your EC2 console in your AWS account and you should see a new instance initializing
  - In the Kubernetes dashboard, go to Nodes, in about 3 minutes, you should see the new node appear
  - About 30 seconds after showing up, the new node should go to the Ready state  
  - Now simulate another scale out event by changing the deployment targets for both from 13 to 15, and watch the response via kubectl
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      15        15        15           15          6m
memory-intensive-app   15        15        15           15          6m
```

#### Scaling Out Beyond Spare Capacity

For the vast majority of cases, normally the spare capacity is replaced in the background and is available by the time any
additional scale out events occur. But in the rare case that a sudden large burst of traffic arrives, 
the spare capacity might be completely consumed. Lets simulate this case by over doubling the number of pods, and watching
AutoScalr's response.

- Change the number of replicas for both the the cpu-intensive-app and memory-intensive-app deployment to 30.
- Check the status of the deployments with kubectl:
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      30        24        20           20          9m
memory-intensive-app   30        25        21           21          9m
```
- Some pods will start very quickly, but there are not enough resources to start all 30.
- AutoScalr detects this condition, determines exactly how many more nodes are needed, and starts them
- Go the Metrics in the AutoScalr UI and note how the target vCPU and Memory levels rose significantly, and the corresponding instances were added 
- Go to your EC2 console and see the new nodes being added
- Go to the Nodes section of the Kubernetes dashboard and you can see the new nodes as they join in ~3 minutes
- Watch via kubectl as the remaining pods start 
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      30        30        30           30          14m
memory-intensive-app   30        30        30           30          14m
```
- The entire process should take 4-5 minutes, and is the same regardless of the scale level.  Even scaling 10x would take
the same 4-5 minutes. 

### 7. Reduce Cost

AutoScalr automatically drives the EKS cluster to the most cost-effective set of instances that still meets the load and spare capacity 
requirements.  To see the cost of the cluster, go to the Cost tab of the Metrics page for the EKS cluster in the AutoScalr UI.
The green band shows how much savings is being generated relative to Fargate pricing. EKS Fargate is not available yet,
so pricing has not been announced, so this pricing is based upon current ECS Fargate pricing. The light blue band signifies additional 
potential savings available to you, and the dark blue band the theoretical optimal based upon our observation of 
many EKS clusters.  To reduce the cost of this cluster further:
- Go to the Settings page for this EKS cluster in the AutoScalr UI
- Under the Spot Parameters section, set
  - Max spot in 1 market to 25%
  - Max spot in all markets to 90%
  - Save settings
- Go to Metrics page and watch AutoScalr's actions
  - AutoScalr will begin replacing some of the instance with spot instances while following the parameters you specified
  - After ~5-6 minutes the transitions should be complete, go back to the Cost tab and see how the aggregate costs have been 
  significantly lowered and should be much closer to the Optimal curve.
  - You can play with different settings and see the different cost savings generated.  In general, it is a good strategy
  limit the max in 1 spot market to be less than or equal to the amount of target spare capacty, that way even in the unlikely event
  of the loss of an entire spot market, the spare capacity would absorb it similar to a scale up event and there would be no impact at 
  all on the application's availablity or performance, and then AutoScalr would 
  replenish the capacity in other spot markets and/or On-Demand instances, as required. 

### 8. Dynamic Instance Type Right-Sizing

AutoScalr not only makes sure you have enough nodes powering your EKS cluster, and blends in Spot instances to lower cost, 
it also does automatic "right-sizing" to dynamically pick the most cost-effective set of instance types based upon the 
load characteristics.  We will demonstrate this by changing the relative number of cpu and memory intensive deployment replicas.

- Change the number of replicas for the the memory-intensive-app from 30 down to 1.
- Watch the deployments status via kubectl:
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app      30        30        30           30          22m
memory-intensive-app    1         1         1            1          22m
```
- Go to the AutoScalr metrics page and notice AutoScalr's response
  - AutoScalr will likely remove at least one of the m4.large or c4.large instances
  - Check the "INSTANCE TYPES" tab to see how the blend of instance types has shifted

Now shift the load to be memory intensive:
- Change the number of replicas for the cpu-intensive-app from 30 to 1, and the memory-intensive-app from 1 to 30.
- Watch the deployments status via kubectl:
```sh
$ kubectl --namespace=kube-public get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
cpu-intensive-app       1         1         1            1          35m
memory-intensive-app   30        30        30           30          35m
```
- Check the INSTANCE TYPE tab and watch as the cluster shifts to more cost effective r4.large instance types

### 9. Play

Change the target replicas on both services in different ways and watch AutoScalr's response to changing the EKS cluster
to support.  Try a "double traffic spike" where you scale up by 30% and within a minute or so scale up again another 40%.
You should see responses to both events, even while the first set of instances are still joining the cluster.

### 10. Cleanup

Once you are done, delete you EKS cluster by:

```sh
$ eksctl delete cluster eks-autoscalr
```
It will take a few minutes to shut everything down.
You can check progress and verify everything was correctly deleted in the CloudFormation section of your AWS console.
Check the EKS and EC2 consoles to make sure everything was properly cleaned up.
Occaisionally we have seen some resource prevent the full deletion of the stack and you have to delete it manually and
delete the CloudFormation stack a second time.


