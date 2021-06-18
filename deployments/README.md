# Deployments
You can use the Helm chart or do the individual `kubectl` commands from the Kubernetes folder. See the specific information below. Before you start, please make your namespace accordingly. If you do not want your namespace to be `openrmf` adjust the [./kubernetes/namespace.yaml](./kubernetes/namespace.yaml) file and your [./chart/openrmf/namespace.yaml](./chart/openrmf/values.yaml) file for helm.

```
kubectl apply -f ./kubernetes/namespace.yaml
```

> OpenRMF v 0.13 and beyond requires Jaeger. See the documentation if putting this into Kubernetes for the Jaeger Operator at https://www.jaegertracing.io/docs/1.17/operator/#installing-the-agent-as-daemonset. This is to show tracing and other features of the application internal calls for debugging and information when required. 

Also make sure you have a persistent volume as there are several pieces in here that require the PV. There is a [PV YAML](./kubernetes/pv.yaml) file to show how this works. 

## Helm
For deployments using helm see the [chart/openrmf](./chart/openrmf/) folder. There is a values.yaml file that has comments and fields to use. If you wish to use the helm chart to generate the YAML like I do, you can run the following command below from the deployments folder (after you do a git clone or download the code ZIP) to make the files. 

```
helm template RELEASE_NAME chart/openrmf --output-dir DIR_NAME -n NAMESPACE
```
or to put into a single file to deploy
```
helm template RELEASE_NAME chart/openrmf > ./openrmf.yaml
```
Once the file(s) are generated you can apply the files. Make sure the namespace in the values.yaml file for the chart and the 
namespace you made in step 1 are the same!

1. Create your namespace, i.e. `kubectl apply -f ./deployments/kubernetes/namespace.yaml`
2. Apply the file(s), i.e. `kubectl apply -f <path-to-where-your-helm-YAML-files-are> -n <namespace-you-specified>`

## Kubernetes
For a straight kubernetes (k8s) installation w/o helm go to the [kubernetes](./kubernetes) folder and make the namespace with the `kubectl apply -f ./namespace.yaml`. Then deploy all the pieces locally. You may have to adjust the services based on your setup.

## Generating Secrets
To use secrets in the YAML file you need to generate the values in base64 encoding. The username, initial root password, database name, 
etc. are all used in other places through the API YAMLs and database YAML files to bring up and connect to MongoDB.

```bash
echo -n 'openrmf' | base64 
```

# AWS EKS Specifics

## Setup the Persistent Volume

If you wish to use the Amazon Web Services Kubernetes service EKS, then follow the information here https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html to setup a persistent volume to use across your cluster if you have not done so already. The OpenRMF uses persistent volume claims (PVC) to store database data.

* curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
* aws iam create-policy --policy-name Amazon_EBS_CSI_Driver --policy-document file://example-iam-policy.json

Keep a copy of the arn returned: 
* "arn": "arn:aws:iam::xxxxxxxxxxxx:policy/Amazon_EBS_CSI_Driver",

You also an run `kubectl -n kube-system describe configmap aws-auth` to get the information
```
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"mapRoles":"- rolearn:  arn:aws:iam::xxxxxxxxxx:role/openrmf-eks-workers-NodeInstanceRole-XXXXXXXX\n  use...

Data
====
mapRoles:
----
- rolearn:  arn:aws:iam::xxxxxxxxx:role/openrmf-eks-workers-NodeInstanceRole-XXXXXXXXXXXXXXX
  username: system:node:{{EC2PrivateDNSName}}
  groups:
    - system:bootstrappers
    - system:nodes

Events:  <none>
```

You then attach the policy to the role so you can use it.

```
aws iam attach-role-policy --policy-arn arn:aws:iam::xxxxxxxx:policy/Amazon_EBS_CSI_Driver --role-name openrmf-eks-workers-NodeInstanceRole-XXXXXXXXXX
```

You then run the following command to setup the PV: 
```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

Now you can have a persistent volume claim (PVC) in your deployment YAML files for anything you may need, such as a database or file uploads in your application.

## Add HTTPS to your EKS endpoint

To get https working on your EKS endpoints, read this article: https://aws.amazon.com/premiumsupport/knowledge-center/terminate-https-traffic-eks-acm/. This is the one I got to work successfully every time. And it is what I did in my charts for now while I test others. The issue (read PITA) is the DNS you get from each LoadBalancer, you have to add CNAME records for a TLD to point to them. No path level routing. Not very IaC so still working toward a better solution.

Or if you need path based routing, https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/. And then https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/tasks/ssl_redirect/ for the path routing across all pieces.

## Deploy the Metrics Server
Please read up on https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html to see how you do that. 
* Pull down the metrics server tar gz
* Untar
* Apply the YAMLs in the directory
* Run `kubectl get deployment metrics-server -n kube-system`

## Using Network Policies on EKS
You need to look to kubectl apply -f kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.6/config/v1.6/calico.yaml as an example to enable network separation and tenant isolation. There are some starting NetworkPolicy YAML files in the OpenRMF chart. But you need something like Calico or Cilium or other CNI plugins setup on your EKS Cluster. 