Kubernetes Ingress Controller as an Interface

The Kubernetes Ingress resource defines a generic API specification for managing HTTP(S) traffic routing to services within a Kubernetes cluster.

However, Kubernetes itself does not include an implementation of an Ingress Controller. It only provides the framework (the interface) for how traffic should be routed based on the Ingress resource.

Cloud Provider and Custom Implementations

Every cloud provider (and some third-party vendors) offers their own implementation of the Ingress Controller to work with their specific load balancing technologies. Each implementation translates the generic Kubernetes Ingress specification into actions specific to their infrastructure.

Here’s how it works:

* Kubernetes provides the Ingress specification as a common interface.

* Cloud providers (or third parties) provide the Ingress Controller as the implementation, which is responsible for:

    * Reading the Ingress resources in the cluster.

    * Configuring their underlying load balancers or proxies accordingly.

Why Does Each Provider Need Its Own Implementation?

1. **Different Load Balancers:** Each cloud provider uses its own load balancing technology (e.g., ALB for AWS, Google HTTP Load Balancer for GCP). The Ingress Controller must translate the Kubernetes Ingress rules into configurations compatible with the provider's load balancer.

2. **Cloud-Specific Features**: Different cloud providers offer unique features such as:

    * AWS: Web Application Firewall (WAF), ACM for SSL certificates.
    
    * GCP: Backend services and URL maps.
    
    * Azure: Path-based routing using Azure Application Gateway. Each implementation must support these features.

3. **Annotations:** Cloud providers extend the Kubernetes Ingress specification with annotations to allow users to specify cloud-specific configurations. For example:

    * AWS-specific annotation: alb.ingress.kubernetes.io/scheme: internet-facing
    
    * GCP-specific annotation: kubernetes.io/ingress.global-static-ip-name: my-static-ip

* Ingress: Acts as the interface or specification that defines routing rules for HTTP(S) traffic.

* Ingress Controller: Acts as the implementation that translates the Ingress specification into actions specific to a load balancer or proxy.

This separation of interface (Ingress) and implementation (Ingress Controller) allows Kubernetes to remain vendor-agnostic while still supporting the unique capabilities of each cloud provider.

-------------


***The AWS Load Balancer Controller is a Kubernetes controller that runs inside your EKS cluster as a deployment (just like any other Kubernetes application). It is deployed as a set of pods that monitor Kubernetes resources like Ingress and Service.***


**What resources does the AWS Load Balancer Controller need to access?**

The AWS Load Balancer Controller interacts with AWS services to manage load balancers and related networking resources on behalf of your EKS cluster. Here's a list of resources it commonly needs to access in AWS:

1. Elastic Load Balancers (ALBs or NLBs):

    * The controller creates and configures Application Load Balancers (ALBs) or Network Load Balancers (NLBs) when you create Ingress or Service resources in your EKS cluster.

2. Target Groups:

    * These are backend targets (EC2 instances, IP addresses, or pods) that the load balancer forwards traffic to. The controller configures these based on Kubernetes services.

3. VPC and Subnets:

    * The controller needs to identify and use the VPC and subnets where your EKS worker nodes are running so that it can associate load balancers and route traffic properly.

4. Security Groups:

    * The controller ensures that appropriate security group rules are applied to the load balancers so they can receive and forward traffic securely.
5. Route 53 (optional):

    * If you’re managing DNS records (e.g., setting up custom domain names for your Ingress), the controller might interact with Route 53 to configure DNS.

---------

# Installation


```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster expense \
    --approve


```

This command associates an OIDC (OpenID Connect) provider with your EKS cluster. It enables your Kubernetes workloads (like pods) to securely assume AWS IAM roles without requiring AWS access keys.

Step-by-Step Explanation

1. Creates an OIDC Provider for the Cluster:

    * EKS clusters use IAM roles for service accounts to allow pods to interact securely with AWS resources.

    * OIDC is used as the bridge between Kubernetes service accounts and AWS IAM roles.
    * This command registers the OIDC provider for the cluster in AWS.

2. Allows Kubernetes to Use IAM Roles via OIDC:

    * Once the OIDC provider is associated, AWS can verify the service account's identity using the tokens issued by the Kubernetes API.
    * This is necessary to let Kubernetes workloads assume IAM roles securely without hardcoding AWS access keys.

3. Authorizes the OIDC Provider:

    * The --approve flag ensures the OIDC provider is automatically approved during the setup.



```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster expense \
    --approve
```

Download an IAM policy for the LBC using one of the following commands:

```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

```

Create an IAM policy named AWSLoadBalancerControllerIAMPolicy. If you downloaded a different policy, replace iam-policy with the name of the policy that you downloaded.

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```
Take note of the policy ARN that's returned.


Create an IAM role and Kubernetes ServiceAccount for the LBC. Use the ARN from the previous step.

```
eksctl create iamserviceaccount \
--cluster=expense \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::528757792370:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve

```

Add controller to cluster

```
helm repo add eks https://aws.github.io/eks-charts

```
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=expense --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
``

