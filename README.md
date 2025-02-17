# PLATFORM ENGINEERING STARTER KIT

## Introduction

In this repository we are going to see how to automate the creation of a new development environment for a team. We'll provide an environment with these capabilities:

- A Kubernetes cluster ready to be used
- A set of git repositories
- A base platform ready with Kafka and Kafka tools, Redis, Postgresql, Prometheus or Grafana
- An example application using this platform in order to being used as starter point for the dev team
- A CI/CD based on Tekton and ArgoCD configured and ready

## Pre-requisites

- AWS account
- [Eksctl](https://eksctl.io/)
- Kubectl
- Helm
- Gitlab Account and token

## Repository structure (WIP)

This image shows the repository structure:

![repo_structure](doc/pictures/repo_structure.jpg)

## The use case

In this PoC we are going to automate an IDP creation for our company. In our case, the current development environment is composed for these components:

- A Kubernetes cluster
- A Postgresql instance
- A Kafka broker and some UI Tool
- Redis
- Prometheus and Grafana
- CI/CD

So, our teams need an IDP with these requirements. The following image describes these requirements:

![requirements](doc/pictures/requirements.jpg)

Install and maintain this IDP isn't productive for development teams so it was decided to delegate these tasks to a Platform team in order to the development teams keep their focus on build business capabilities instead on keeping their environments up and running.

This Platform team also needs to automate those kind of tasks in order to:

- Be more efficient, productive and avoid human mistakes when perform platform tasks
- Build golden paths over the platform to help the development teams to build business capabilities
- Be able to evolve the platform

*TODO: Governance, Networks, security groups, etc...Talking about how difficult are those points*

## Solution design

- Design the steps to provide an IDP
- Decide how the provisioning of the IDP is going to be automated

### IDP

The target of this stage is to provide an IDP ready to be used for the development teams. This IDP has to meet the requirements described above.

So, the platform team begins to design the platform and how it could be automated. In order to separate concerns and build an evolvable platform, they decided to perform multiple steps to build the IDP instead of having just one. So, the steps to automate the creation of the platform are summarized in this picture:

![steps](doc/pictures/steps.jpg)

So, the steps are:

1. Automate the creation of the platform base, that means, creating the kubernetes cluster and the main database. The reasons to perform this step seperated are:

   - being able to choose different kubernetes and database providers (AWS, GCP, Azure,...) without impacting in development teams or even, being able to provide different physicall platforms depending on the needs. For instance: development, QA,...,etc
   - keeping the process of providing the main infrastructure isolated to be evolved when needed
2. Automate the creation of the platform tools. The reasons to perform this step seperated are:

   - adding, updating (for instance, versions) or deleting tools autonomously with minimum impact in development teams
   - being able to choose different options depending on the context. For instance: an open source Kafka provider for dev environments and a licensed Kafka provider for production environment
   - keeping the process of providing the platform tools isolated to be evolved when needed
3. Automate the creation of the development repositories and CI/CD tools. The reasons to perform this step seperated are:

   - providing repositories with the right structure and with an example of the component to develop (for instance: microservice, spa, etc...)
   - providing CI/CD tools ready to be used for the development teams, allowing them to easily build and deploy business features

### How to automate the provisioning of IDPs

Once the steps to provide an IDP are designed, the platform team has to decide what tools can use to automate the steps to build the IDP.

The platform team chooses [Crossplane](https://www.crossplane.io/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) in order to create a control plane cluster to provide, centrally, the IDPs:

![crossplane-cluster](doc/pictures/crossplane-cluster.jpg)

From this control plane, the different steps to create the IDP will be automated and managed.

### Putting everything together

#### The (k8s) control plane cluster

The following picture shows the target to achieve:

![step1](doc/pictures/step1.jpg)

So, the first step is to create a K8s cluster to be the control plane cluster. In this PoC we are going to create and use an AWS EKS cluster but you can use other K8s distribution if you want.

Let's see that in more detail how this cluster is going to be composed:

##### Crossplane

[Crossplane](https://www.crossplane.io/) is the core of our control plane cluster. By Crossplane, we'll automate the infrastructure creation using a declarative way (Kubernetes-style). So, we are going to deploy Crossplane and configure it with these components:

- Compositions, definitions and claims: in this PoC, we'll provide resources and claims to create an EKS cluster and a Postgresql RDS instance
- Providers: we'll install these providers:

  - AWS: this provider will be used to create and manage AWS infrastructure
  - K8s: by this provider we'll be able to create objects in the managed kubernetes
  - Helm: to manage and deploy Helm charts in K8s clusters
  - Gitlab: we'll use this provider to create groups and projects in Gitlab in order to provide repositories to the different teams

##### ArgoCD

We are going to use [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) to implement GitOps and being able to keep always synchronized Crossplane resources with the "Platform Repository". By this way, everytime a resource is updated in Git, the change is propagated automatically to our control plane cluster.

![argo-crossplane-resources](doc/pictures/argo-crossplane-resources.jpg)

We'll also use ArgoCD to manage the multiple claims requests to create infrastructure. So, when a dev team needs a Kubernetes cluster to build and deploy its applications, it creates a claim file and push it to "Claim Requests Repository". Then, ArgoCD pulls that change and creates a new claim object managed by Crossplane. Finally, Crossplane creates the infrastructure associated to that claim

![how-claims-works](doc/pictures/how-claims-works.jpg)

## Hands-on

### Control Plane Cluster

The first step is creating and configuring the Control Plane Cluster.

In this PoC, we are going to create an EKS cluster using [eksctl](https://eksctl.io/). The cluster configuration is in the file *cluster-conf.yaml*.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: crossplane-poc-cluster
  region: eu-west-3
  version: "1.27"

nodeGroups:
  - name: ng-1
    instanceType: m5.xlarge
    desiredCapacity: 2
    volumeSize: 80
```

The following script encapsulates the cluster creation:

```bash
sh 01.create-cluster.sh
```

Once the creation is finished, we have an empty EKS:

![empty_eks](doc/pictures/empty_eks.jpg)

This k8s cluster will be our Control Plane. The next step is to configure ArgoCD and Crossplane.

### Installing ArgoCD and Crossplane

Once the main cluster is up and running, we have to install and configure [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and [Crossplane](https://www.crossplane.io/). To do this, just execute the following script:

```bash
sh 02.setup_control_plane_cluster.sh <AWS_ACCESS_KEY_ID> <AWS_SECRET_ACCESS_KEY> <GITHUB_TOKEN>
```

This script takes about 5-6 minutes, so be patient. When the script ends, we'll see the url to access to ArgoCD and the credentials. Something like this:

```bash
ARGOCD URL: https://a8628e104aa7549c3a979c27185934f3-369920353.eu-west-2.elb.amazonaws.com:80/
ARGOCD Credentials: admin/FkmHYXTwAWT33eFZ
```

Now, we have our control plane cluster ready with ArgoCD and Crossplane installed:

![step1](doc/pictures/setup_control_plane_cluster.jpg)

If we go to ArgoCD url we'll see that there are no applications:

![argocd_init_screen](doc/pictures/empty_argocd.jpg)

In the next steps, we'll create two applications: crossplane-resources and claims

### ArgoCD Applications

Now, we need to create two ArgoCD applications in order to manage our platform:

- crossplane-resources, to manage and synchronize the resources that the platform provides (EKS, RDS,...etc)
- claims, to manage and synchronize the requests of the platform users

With these two applications, our control plane cluster will be ready to manage claims and create the resources requested

![step1](doc/pictures/step1.jpg)

To create the applications, just execute the following script:

```bash
sh 03.setup_control_plane_argocd_apps.sh
```

Once the script finishes, if we enter to the ArgoCD url we'll see these two applications created:

![argocd_apps](doc/pictures/argocd_apps.jpg)

Let's see them in more detail!

#### ArgoCD App: crossplane-resources

We have to create an application to keep synchronized crossplane resources between the platform git repository and the control plane cluster. This app is listening for changes in "https://github.com/jaruizes/platform-argo-crossplane/platform-gitops-repositories/crossplane-resources" . So, if we add more crossplane composites or update what exists, ArgoCD applies those changes to the main kubernetes cluster.

If you click in this app, you can see that the objects in "platform-gitops-repositories/crossplane-resources" are managed by ArgoCD:

![crossplane_initial_composite](doc/pictures/crossplane_initial_composite.jpg)

ArgoCD has installed those objects in the cluster and keeps them synchronized with the repository.

We can also check it directly against the cluster, executing this command:

```bash
kubectl get Composition -n crossplane-system
```

We'll see the composition installed:

```bash
NAME            XR-KIND         XR-APIVERSION             AGE
platformbase    platformbase    jaruiz.crossplane.io/v1   10m
platformtools   platformtools   jaruiz.crossplane.io/v1   10m
```

And if we execute:

```bash
kubectl get CompositeResourceDefinition -n crossplane-system
```

we'll see the composite definition installed:

```bash
NAME                                  ESTABLISHED   OFFERED   AGE
platformsbase.jaruiz.crossplane.io    True          True      11m
platformstools.jaruiz.crossplane.io   True          True      11m
```

#### ArgoCD App: **teams-claims**

This application is listening for changes in "https://github.com/jaruizes/platform-argo-crossplane/platform-gitops-repositories/claims" . So, a team wants to get a K8s cluster, the team will push a file containing the K8s claim in that path, ArgoCD will apply that change to the main kubernetes cluster and Crossplain will create the new cluster

If we click to see the details of the app, we'll see that there is no objects there. That is because we haven't requested any claim (that means that we haven't performed any push to "https://github.com/jaruizes/platform-argo-crossplane/platform-gitops-repositories/claims")

![argocd_inital_teams_claims](doc/pictures/argocd_inital_teams_claims.jpg)

### Creating the IDP

Now it's time to create the IDP using the control plane cluster. As we said before, we've divided it into three steps:

![steps](doc/pictures/steps.jpg)

In the following steps, we are going to push three claims to "https://github.com/jaruizes/platform-argo-crossplane/platform-gitops-repositories/claims" in order to create the IDP

#### Platform base

Now it's the moment to create a claim. You can find claim examples in the folder "claims-example"

So, we copy the file "claims-example/platformbase-claim.yaml" to the folder "platform-gitops-repositories/claims" and call it "products-platform-base-claim.yaml", for instance.

So, if we assume that we are going to create the base platform for the products team, the claim file will have the following content:

```yaml
apiVersion: jaruiz.crossplane.io/v1
kind: platformbaseclaim
metadata:
  name: products-base-platform
  labels:
    cluster-owner: products-team
spec:
  name: products
```

As you can see, in this claim we only can set the name of the team. In our case, we call it "products".

Now, we push this file to the repository:

![eks-claim-team-x](doc/pictures/product-base-platform-claim-push.jpg)

Now, if we go to ArgoCD and we open the application "**teams-claims**" and refresh it, we'll see how all the components associated to the composition are being created (it will last 10 or 15 minutes...):

![argocd_cluster_created](doc/pictures/argocd_cluster_created.jpg)

Besides that, we can check it using kubectl:

```bash
kubectl get clusters.eks.aws.upbound.io -n crossplane-system
```

we'll see how our cluster is ready:

```bash
NAME               READY   SYNCED   EXTERNAL-NAME      AGE
products-cluster   True    True     products-cluster   23m
```

And, once the cluster is created, we can check the secret containing the connection details:

```bash
kubectl describe secret products-conn  -n crossplane-system
```

The result should be similar to this:

```bash
Name:         products-conn
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
clusterCA:   1107 bytes
endpoint:    72 bytes
kubeconfig:  2367 bytes

```

We need to add this connection to our kubeconfig. As we as creating an EKS cluster, we can get it executing:

```bash
aws eks update-kubeconfig --region eu-west-3 --name products-cluster 
```

Now, we can get the ArgoCD url for this cluster executing:

```
kubectl get service products-argo-helm-release-argocd-server -n argocd -o jsonpath='https://{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[0].port}/'
```

The result should be similar to:

```
https://a3f6a0cac799d43f499454fe15873d0c-976441945.eu-west-3.elb.amazonaws.com:80/
```

And, you can get the admin password, executing:

```
kubectl get secret argocd-initial-admin-secret -n argocd --template={{.data.password}} | base64 -D
```

The result should be similar to:

```
vKIAMHerbUsvIiP2
```

Write it down because we'll need it in the next step

#### Platform tools

Once the base platform is ready, we need to push the next claim to the repository. As we did before, we copy the example file ("claims-example/platformtools-claim.yaml") to the folder "platform-gitops-repositories/claims" and call it "products-platform-tools-claim.yaml", for instance.

So, if we assume that we are going to create the platform tools for the products team, the claim file will have the following content:

```yaml
apiVersion: jaruiz.crossplane.io/v1
kind: platformtoolsclaim
metadata:
  name: products-platform-tools
  labels:
    cluster-owner: products-team
spec:
  name: products
```

As in the previous case, we only have to specify the name of the team ("products"). We push it to the repository and refresh the claims application in ArgoCD. We'll see how new components are being created:

![argocd_cluster_created](doc/pictures/argocd_platform_tools_created.jpg)

Now, if we go to Products ArgoCD url, we'll see the platform tools applications installed:

![argocd_platform_tools_installed](doc/pictures/argocd_platform_tools_installed.jpg)

#### Platform CI/CD and example project

Now, in this PoC, we've decide that the next step is to provide a repository with an example of application and a simple CI/CD process.

In this PoC we are going to use a Github template to create teams repository. This repository is: https://github.com/jaruizes/platform-argo-crossplane_project_template . It will be possible to use other Git provider as Gitlab, for instance.

This is a PoC and we've only implemented one template but, in a real environment, we could have a template for each kind of artifact used in our company. Teams could specify in the claim which kind of artifact they need.

We are also going to use Github and ArgoCD actions to implements a simple CI/CD process:

![cicd](doc/pictures/cicd.jpg)

As you can see in the image, there is a file (build.yaml) implementing the CI pipeline by Github Actions and the CD part is implemented using ArgoCD app. The CI part builds the artifact and upload the new image to Image Registry and then, changes the application deployment file and push it to the Gitops repo. The ArgoCD App pulls those changes and applies them to the K8s team cluster.

For simplicity, in this PoC we are used the same repository for application code and Gitops. Gitops files are in the path "/gitops/manifest" .

The important thing here is that we are used Github Actions and ArgoCD to implement the CI/CD pipeline but, in a real environment, we could provide other CI/CD pipeline to the teams.

## Removing the environment

We have to perform two tasks:

- Remove the claim request to remove the environment for the Team X
- Remove the K8s main cluster

### Removing the claims

If we delete the claim files from the repository (path "platform-gitops-repositories/claims") and we push the changes:

![deleted_claim_team-x](doc/pictures/deleted_claim_team-x.jpg)

the cluster for the team X will be deleted by ArgoCD:

![argocd_delete_claim](doc/pictures/argocd_delete_claim.jpg)

We can also check it using kubectl:

```bash
kubectl get clusters.eks.aws.upbound.io -n crossplane-system
```

we'll notice that the cluster is not ready now:

```bash
NAME                 READY   SYNCED   EXTERNAL-NAME        AGE
k8s-team-x-cluster   False   True     k8s-team-x-cluster   22m
```

And, if we execute the command some minutes later, we'll see that the resource isn't found.

We also can check AWS Console:

![cluster_deleting_aws](doc/pictures/cluster_deleting_aws.jpg)

### Removing the main cluster

Once the team cluster is deleted we can destroy the main cluster executing:

```bash
sh 04.delete-cluster.sh
```

## Next steps

- Crossplane testing: how to test compositions, definitions, etc...
- A dev portal like Backstage
- Why Crossplane (instead of Terraform)?
