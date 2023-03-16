# GitOps for organizations - Part I: Provisioning Openshift clusters using GitOps with ACM

In this first part, ACM is used to provision Openshift clusters using Gitops. The users fill in the clusters parameters in a form, which are written to a yaml/json object and pushed to git. ArgoCD synchronizes these objects into the ACM cluster. The cluster is provisioned with ACM, which automatically adds the clusters to Openshift GitOps for Day 2.

![Openshift Gitops Overview](../img/gitops-for-organization-overview.png)

## Cluster Parameters

We need a frontend to create the objects needed by ACM to provision the clusters. This frontend can be AAP (Tower), Jenkins, or any custom web application with a form. Although ACM can be used as frontend to provision the clusters, there are some drawbacks: ACM stores the objects locally, not in git; ACM has a fixed form for cluster creation, which cannot be customized.

Instead of creating all the objects, we just need to create a 2 objects:
- conf.yaml: contains all the configuration parameters of the cluster
- provision.yaml: contains the parameters needed to provision the cluster, like intall-config.yaml


In our example repository, we can see these files for the cluster example [zamora.dev.redhat.com](clusters/dev/zamora.dev.redhat.com):

```
clusters
└── dev
    └── zamora.dev.redhat.com
        ├── conf.yaml
        ├── provision.yaml
        └── overlay
            └── kustomization.yaml
```

## Cluster-provisioning ApplicationSet

We’re using an ApplicationSet to search for provision.yaml files, and it’ll create an Argo Application for each file. This application will use a Helm chart to deploy all the ACM objects needed:

- ClusterDeployment
- KlusterletAddonConfig
- MachinePool
- ManagedCluster
- Namespace
- Secrets: pull-secret, install-config, ssh-private-key and creds

The Helm [ACM provision chart](provision/openshift-provisioning/templates) has these templates: 

```
base
└── provision
    ├── Chart.yml
    └── templates
        ├── ClusterDeployment.yml
        ├── KlusterletAddonConfig.yml
        ├── MachinePool.yml
        ├── ManagedCluster.yml
        ├── Namespace.yml
        └── Secrets.yml
```

The ApplicationSet creates an Argo Application for each provision.yaml under clusters folder using the path: `base/provision/openshift-provisioning` and the config files:

* `/conf/{{cluster.environment}}/provision.yaml`
* `/clusters/{{cluster.environment}}/{{cluster.fqdn}}/provision.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
 name: cluster-provisioning
 namespace: openshift-gitops
spec:
 generators:
   - git:
       repoURL: https://github.com/albertogd/gitops-for-organizations.git
       revision: main
       files:
         - path: "clusters/**/provision.yaml"
 template:
   metadata:
     name: "provision-{{cluster.environment}}-{{cluster.name}}"
     labels:
       environment: "{{cluster.environment}}"
       cluster: "{{cluster.fqdn}}"
       region: "{{cluster.region}}"
       cloud: "{{cluster.cloud}}"
   spec:
     project: default
     source:
       repoURL: https://github.com/albertogd/gitops-for-organizations.git
       targetRevision: main
       path: base/provision/openshift-provisioning
       helm:
         valueFiles:
           - /conf/{{cluster.environment}}/provision.yaml
           - /clusters/{{cluster.environment}}/{{cluster.fqdn}}/provision.yaml
     destination:
       server: 'https://kubernetes.default.svc'
       namespace:
     syncPolicy:
       syncOptions:
         - preserveResourcesOnDeletion=true
```

### Use of preserveResourcesOnDeletion

Using an ApplicationSet to dynamically generate the provision applications has a risk involved: if the ApplicationSet is deleted. the following occurs (in rough order):

The ApplicationSet resource itself is deleted
Any Application resources that were created from this ApplicationSet (as identified by owner reference)
Any deployed resources (Deployments, Services, ConfigMaps, etc) on the managed cluster, that were created from that Application resource (by Argo CD), will be deleted.
  
To prevent the deletion of the resources of the Application, such as ManagedCluster, ClusterDeployment, etc, set .syncPolicy.preserveResourcesOnDeletion to true in the ApplicationSet. This syncPolicy parameter prevents the finalizer from being added to the Application.

## Cluster Provision

Once the ACM objects are synchronized to the cluster, ACM will start to provision the new cluster. 


## Day 2 Configuration

To add each new cluster to GitOps, we need to create this 3 objects in ACM:
- ManagedClusterSetBinding
- Placement
- GitOpsCluster

A ManagedClusterSetBinding resource allows us to bind a ManagedClusterSet resource to a namespace. In our example, we want to bind the clusterSet vmware to the openshift-gitops namespace:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
 name: vmware
 namespace: openshift-gitops
spec:
 clusterSet: vmware
```


The Placement allows us to select clusters based on labels, in the case, platform = vmware:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
 name: vmware
 namespace: openshift-gitops
spec:
 predicates:
 - requiredClusterSelector:
     labelSelector:
       matchExpressions:
       - key: platform
         operator: "In"
         values:
         - vmware
```



The GitOpsCluster custom resource allows us to register the clusters selected with a Placement to an ArgoCD server:

```yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
 name: argo-acm-clusters
 namespace: openshift-gitops
spec:
 argoServer:
   cluster: local-cluster
   argoNamespace: openshift-gitops
 placementRef:
   kind: Placement
   apiVersion: cluster.open-cluster-management.io/v1beta1
   name: vmware
   namespace: openshift-gitops
```