# Elastic Kubernetes Service (EKS)

This project involves the following:

- Creating a kubernetes cluster on AWS using EKS
- Deploying MongoDB and Mongo-Express
- Deploying a web application
- Visualizing the cluster on grafana dashboard
- Installing auto-scaler to manage cluster nodes
- Deploying nginx application
- Configuring a serverless fargate profile

## EKS

1. Go to IAM page
    - Click on Roles, then create role
        + Select AWS Service
        + Under 'use case' dropdown, choose EKS. Then check 'EKS - Cluster' and click next.
            - This attaches the AmazonEKSClusterPolicy
        + Click next
        + Give it a role name and then create.

2. Creating VPC for EKS cluster
- Follow documentation here: https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html

- Go to CloudFormation page
    - Click on 'Create stack'
    - Check 'Template is ready' and check 'Amazon S3 URL'
    - Paste the yaml template url copied from the documentation page above (https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html) and click on next.

        + You can use view in designer to see pictorial view of the architecture of the resources to be created

![template1-designer]()

    - Give it a stack name. Click on next. Continue until Submit.
    - This creates all the resources and connections

![cloud formation]()

3. Create EKS Cluster

- Go to EKS page
    - Under Add cluster, click create
    - Give the cluster a name and choose latest version of kubernetes
    - Confirm cluster service role is using the newly created IAM EKS role and go Next
    - Choose VPC and security group created with cloud formation
    - IPv4 for cluster address family
    - For cluster endpoint access, choose public and private and click next
    - Leave configure logging and select add-ons as default and click next, and next.
    - Then review and create (takes some time)

![cluster]()

4. Setup an EC2 instance to interact with the cluster

- Go to EC2 page
    - Launch instance with ubuntu AMI (t2.micro)
    - You can use default vpc and create
    - Confirm the security group has ssh access

- Log into the server (ssh)
    - `apt update`
    - `apt install unzip`
    - Install aws cli following documentation here: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
    - Configure aws cli using `aws configure` (input access and secret key)
    - Do `aws configure list` to confirm

- Connecting to the cluster
    - `aws eks update-kubeconfig --region specify-region --name name-of-cluster`
        - This creates a /root/.kube/config file. Do `ls -a` to see the .kube directory.

- Install kubectl
    - Follow the 'install using native package management' here: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
    - `kubectl version`
    - `kubectl get all -A`
    - `kubectl cluster-info`

![kubectl]()

5. Create IAM role for the node group

- Go to IAM page
    - Click on Roles, then create role
        + Select AWS Service
        + Check EC2 as the use case and click next
        + Attach the following policies and click next
            - AmazonEKSWorkerNodePolicy
            - AmazonEC2ContainerRegistryReadOnly
            - AmazonEKS_CNI_Policy
        + Give the role a name and create

6. Create node group

- Go to EKS page
    - Under clusters, go to compute
    - Add node group
        + Specify a name
        + Select the IAM role and click next
        + Set compute and scaling configuration
            - Default AMI, t2.small, 20g disk size
            - Desired, minimum and maximum as default (2)
            - Click next
        + Check 'configure remote access to nodes'
            - Select EC2 key pair and check 'all' for remote access. Then click next.
        + Review and create
    
![node group]()

    - This should add more EC2 instances

![more instances]()

    `kubectl get all -A` to see all resources

![get all]()

    `kubectl get nodes`

![get nodes]()

7. Install ingress

- Follow the process here: https://kubernetes.github.io/ingress-nginx/deploy/#aws

    - This will install the ingress controller and deploy network load balancer

    - `kubectl get ns` to see ingress-nginx namespace created

8. Create namespaces for deployments

    `kubectl create ns mongo`

    `kubectl create ns devopsacad`

9. Create a directory for all our deployment files

    `mkdir manifests`

    `cd manifests`

10. Create and paste the code for all our deployment and service files
    `vi mongo-secret.yaml`
    `vi mongo-configmap.yaml`
    `vi mongo-deployment.yaml`
    `vi mongo-express.yaml`
    `vi devopsacad-deployment.yaml`
    `vi devopsacad-svc.yaml`
    `vi ingress-deployment.yaml`

11. Deploy
    `kubectl apply -f mongo-secret.yaml`
    `kubectl apply -f mongo-configmap.yaml`
    `kubectl apply -f mongo-deployment.yaml`
    `kubectl apply -f mongo-express.yaml`
    `kubectl apply -f devopsacad-deployment.yaml`
    `kubectl apply -f devopsacad-svc.yaml`
    - Do `kubectl get all -A` to see all resources

![kubectl get all]()

12. Get load balancer endpoint
    - In the instance page, scroll down and open load balancers
    - Copy the dns name

13. Map endpoint to route
    - Go to Route 53 page
    - Go to hosted zones
    - Map the dns name copied to the mongo-express, devopsacad and grafana CNAME records created

14. Run the ingress deployment
    `kubectl apply -f ingress-deployment.yaml`
    - Do `kubectl get ingress -n mongo`
        + Confirm mapped address
    - `kubectl get ingress -n devopsacad`
        + Confirm mapped address

15. Browser access
    - Copy the mongo-express.domain-name record fron Route 53 and open in a browser
    - Same for the devopsacad.domain-name

**Visualization**

1. Install helm (package manager for kubernetes)
    - Follow the guide for Ubuntu here: https://helm.sh/docs/intro/install/

2. Using helm to install prometheus stack
    - Follow guide here: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
    - `kubectl get all -A -n default` to see

3. Use ingress to expose the grafana service
    - Defined in grafana-ingress.yaml
    - `vi grafana-ingress.yaml` and paste
    - `kubectl create -f grafana-ingress.yaml`
    - //Make sure to define the endpoint in route 53 hosted zones//
    - Copy the endpoint name and open in browser (grafana.domain-name)
        + Get admin username password from https://github.com/helm/charts/tree/master/stable/prometheus-operator#grafana

4. Grafana dashboard
    - Home > Dashboards > General
    - Click on any resource to see

## Kubernetes Autoscaler
- Defined in k8s-cluster-autoscaler.yaml
- Make sure the version of the autoscaler image matches the version of the kubernetes deployed on aws during cluster creation

- Start up the EKS mmanager instance and create cluster if necessary

- In addition to the node group role already created, we need to create some permissions and add to that node group role

- Go to IAM page,
    - Go to policies, create policy
        - Click JSON and replace the code with k8s-custom-autoscaling-new.json. Then click next.
        - This will show the services that the permissions will affect (EC2, EC2 Auto Scaling, EKS)
        - Give the policy a name and create
    - Go to roles
        - Select the EKS node role created previously
        - Under permissions, go to Add permissions > Attach policies
        - Search for the just created policy, check and add it to the role (customer manages policy)
    
- Go to EKS page
    - Go to clusters > cluster-name
    - Under compute, click add node group
        + Give it a name
        + Attach the node role created and click next
        + Use defaults, but change instance type to t2.micro and click next //t2.micro won't be enough so use higher memory instance or go with *1* below
        + Remove public subnet and leave only the private subnets
        + Toggle on 'configure remote access to nodes'
            - Specify key pair
            - Under allow remote access from, check 'All'. Then click next.
        + Create

- Copy the public ip address of the eks-manager instance and ssh into it
    - Confirm aws configure `aws configure list`
    - Confirm and update the .kube/config file (`ls -a`) with the new cluster deployment (if necessary)
        - `aws eks update-kubeconfig --name name-of-cluster --region specify-region`
        - Now, do `kubectl cluster-info` to confirm the cluster is running.
            + The display endpoint here should match the API server endpoint in the overview page of the clusters page.
        - `kubectl get nodes`
        - `kubectl describe node node-name` to see specific node info

- Update the autoscaler yaml file (if necessary)
    - //You can see the tags for this file under auto-scaling groups under ec2 instance page//

- Create the autoscaler yaml file on the server and paste the code
    - `vi auto-scaler.yaml`

- Apply the file
    - `kubectl apply -f auto-scaler.yaml` //Everything is created in the kube-system namespace//
    - Do `kubectl get deployment -n kube-system cluster-autoscaler`
    - `kubectl get pods -n kube-system`

*1* Create new node group to accomodate resources
- Go to EKS > clusters > cluster-name > compute > Add node group
    - Specify name and add the node role
    - Use t3.small and configure just like before
    - Create
- Go to instance page to confirm instance creation

- On the server, do `kubectl get nodes`
- `kubectl dexcribe pod pod-name -n kube-system`
- `kubectl logs cluster-autoscaler-pod-name -n kube-system`

**Deployment**

- Let's deploy an nginx app, defined in nginx.yaml
    - `vi nginx.yaml` paste the code
    - `kubectl apply -f nginx.yaml` //confirm load balancer on aws
    - `kubectl get deploy` //Copy the endpoint and open in browser

- Let's confirm auto-scaler
    - `kubectl get deploy`
    - `kubectl edit deploy deployment-name` //change replica to maybe 50
    
    - Now do `kubectl get pods`
    - `kubectl get deployment -n kube-system`
    - `kubectl logs cluster-autoscaler-pod-name -n kube-system` //Unable to schedule all pods as max size defined for auto-scaling is reached

    - Now, we can edit the node group and increase the maximum size to say 3.
        - This will launch a new ec2 instance, increase nodes and schedule more pods in the deployment
        - `kubectl get nodes` to see more nodes
    
    - `kubectl edit deploy deployment-name` //change replica to maybe 5
    - `kubectl get pods`
    - `kubectl logs cluster-autoscaler-pod-name -n kube-system` //Scaling down


**Fargate profile for deployment**
- //You can't run auto-scaler or helm chart on fargate//
- //One pod - one fargate profile. One pod - one virtual machine//
- //Fargate is for single pod application deployment//
- //Use can use both fargate and node groups at the same time//

1. Create an IAM role for fargate

- Go to IAM page
    - Go to roles > create role
        + Check aws service
        + For use case, select eks, then check 'eks - fargate pod' and click next
        + This attaches the AmazonEKSFargatePodExecutionRolePolicy. Click next.
        + Add role name and create

- Add fargate profile
    - Go to EKS > clusters > cluster-name
        - Under compute, click on add fargate profile
            + Give it a name
            + Attach the fargate role
            + Slect only private subnets and click next
            + Selct the namespace of the deployment, eg nginx //may need to add namespace to the deployment file//And also create nginx namespace on the server// `kubectl create ns nginx`
            + Expand match labels and add label to specify the pod within that namespace to start up with fargate //same as defined in the yaml file// In your yaml file, you can add "profile: fargate" under match labels for deployment//
            + You can specify up to 5 namespaces and match labels
            + Click next
            + Review and create

- Create fargate
    - `vi nginx-fargate.yaml` and paste the code
    - `kubectl create -f nginx-fargate.yaml` //this schedules our deployment fargate (serverless so no ec2 instance)
    - `kubectl get pods -n nginx`
    - `kubectl describe pod name-of-pod -n nginx`
    - `kubectl get nodes`

**Deleting clusters**
- First delete node group
- Then delete fargate profile
- Finally delete the cluster
- Delete all the 3 IAM roles created
- Also delete the stack under cloudformation