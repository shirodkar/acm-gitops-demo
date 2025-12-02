# ACM GitOps Demo

This is a GitOps repo which demonstrates the features of Red Hat Advanced Cluster Management for Kubernetes (RHACM).

## Prerequisites

- Hub Cluster - Openshift Container Platform 4.20
- 1 or more Managed Clusters - Openshift Container Platform 4.20

## Setup

1. Install Advanced Cluster Management for Kubernetes (ACM) Operator on the Hub cluster
2. Install Openshift GitOps (ArgoCD) Operator on the Hub cluster.
3. Make sure you are logged into the OCP cluster as a cluster admin.
4. Give ArgoCD access to the OCP cluster:

```oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller --rolebinding-name gitops-role-binding```

## Install the Platform components using GitOps

1. ```git clone https://github.com/shirodkar/acm-gitops-demo.git```
2. ```cd acm-gitops-demo```
3. Make sure you are logged into the OCP cluster as a cluster admin.
4. Run the command: 
```oc apply -f gitops/platform/app-of-apps/applications.yaml```

**Note:** ACM Policies and GitOps manifests will be automatically applied to the Hub cluster. It could take about 10-20 minutes for the installation to complete. Wait until all apps in ArgoCD are Healthy and Synced.

## Import Managed Clusters into RHACM

1. In the Openshift Console, switch to ACM using the 'Fleet Management' prespective.
2. Import the Managed cluster(s) into ACM via 'Clusters=>Import Cluster'
  - Make sure to add the cluster to one of the following ClusterSets: dev, qa, uat, prod
  - Make sure to provide the 'env' label with one of the following values dev, qa, uat, prod. Example: ```env=dev```
  - You can provide the server url and token of your cluster(s) to ACM.

**Note:** ACM Policies and GitOps manifests will be automatically applied to the imported Managed clusters. It could take about 10-20 minutes for the installation to complete on the managed clusters. Wait until all apps in ArgoCD are Healthy and Synced.

## Demo

### Deploying an application Manually using the ACM Console

1. TBD

### Deploying an application using GitOps and visualizing it in ACM

1. TBD

### Deploying an Application using Progressive Delivery

1. TBD