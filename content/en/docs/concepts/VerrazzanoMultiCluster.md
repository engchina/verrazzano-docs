---
title: "Verrazzano in a multicluster environment"
linkTitle: Verrazzano in a multicluster environment
weight: 1
draft: false
---

Multicluster Verrazzano consists of an _admin_ Kubernetes cluster and optionally, one or more _managed_ clusters.

Review the following key concepts to understand multicluster Verrazzano:
- [Admin Kubernetes cluster](#admin-kubernetes-cluster)
- [Managed Kubernetes clusters](#managed-kubernetes-clusters)
- [The VerrazzanoProject resource](#the-verrazzanoproject-resource)
- [Verrazzano multicluster resources](#verrazzano-multicluster-resources)
- [Managed cluster registration](#managed-cluster-registration)

For more information, see:
- [Detailed view of multicluster Verrazzano](#detailed-view-of-multicluster-verrazzano)
- [Try out multicluster Verrazzano](#try-out-multicluster-verrazzano)

The following diagram shows a high-level overview of how multicluster Verrazzano works. For a more
detailed view, see the diagram [here](#detailed-view-of-multicluster-verrazzano).

![](../../images/multicluster/MCConceptsHighLevel.png)

### Admin Kubernetes cluster
A Verrazzano admin cluster is a central management point for:
- Deploying and undeploying applications to the managed clusters registered with the admin cluster.
- Viewing logs and metrics for both Verrazzano components and applications that reside in the managed clusters.

You may register one or more managed clusters with the admin cluster by creating a `VerrazzanoManagedCluster`
resource in the `verrazzano-mc` namespace of an admin cluster.

**Note:** The admin cluster has a fully functional Verrazzano installation. You can locate applications on the admin
cluster as well as on managed clusters.

### Managed Kubernetes clusters
A Verrazzano managed cluster has a minimal footprint of Verrazzano, installed using the `managed-cluster`
installation profile. A managed cluster has the following additional characteristics:
- It is registered with an admin cluster with a unique name.
- Logs for Verrazzano system components and Verrazzano multicluster applications are sent to
  Elasticsearch running on the admin cluster, and viewable from that cluster.
- A Verrazzano multicluster Kubernetes resource, created on the admin cluster, will be retrieved and deployed to a
  managed cluster if all of the following are true:
  - The resource is in a namespace governed by a `VerrazzanoProject`.
  - The `VerrazzanoProject` is located in this managed cluster.
  - The resource itself is located in this managed cluster.

### The VerrazzanoProject resource
A [`VerrazzanoProject`](../../reference/api/multicluster/verrazzanoproject "api docs") provides a way to group application namespaces that are owned or administered by the
same user or group of users.
- For multicluster applications to work correctly, _first_ a VerrazzanoProject containing the application's namespace must be created.
- A `VerrazzanoProject` resource is created by a Verrazzano administrative user, and specifies the following:
  - A list of namespaces that the project governs.
  - A user or group that is designated as the `Project Admin` of the VerrazzanoProject. Project admins may deploy
    or delete applications and related resources in the namespaces in the project.
  - A user or group that is designated as `Project Monitor` of the VerrazzanoProject. Project monitors may view
    the resources in the namespaces in the project, but not modify or delete them.
  - A list of network policies to apply to the namespaces in the project.
- If those namespaces do not already exist, then the creation of a `VerrazzanoProject` results in the creation of the
  specified namespaces in the project.
- It also results in the creation of a Kubernetes `RoleBinding` in each of the namespaces, to set up the appropriate
  permissions for the project admins and project monitors of the project.

### Verrazzano multicluster resources
Verrazzano includes several multicluster resource definitions for resources that may be targeted for placement in one
or more clusters: [MultiClusterApplicationConfiguration](../../reference/api/multicluster/multiclusterapplicationconfiguration "api docs"),
[MultiClusterComponent](../../reference/api/multicluster/multiclustercomponent "api docs"),
[MultiClusterConfigMap](../../reference/api/multicluster/multiclusterconfigmap "api docs"),
and [MultiClusterSecret](../../reference/api/multicluster/multiclustersecret "api docs").


- Each multicluster resource type serves as a wrapper for an underlying resource type.
- A multicluster resource additionally allows the `placement` of the underlying resource to be specified as a list of
  names of the clusters in which the resource must be placed.
- Multicluster resources are created in the admin cluster, in a namespace that is part of a `VerrazzanoProject`,
  and targeted for `placement` in either the local admin cluster or a remote managed cluster.
- A multicluster resource is said to be part of a `VerrazzanoProject` if it is in a namespace that is governed
  by that `VerrazzanoProject`.

### Managed cluster registration
A managed cluster may be registered with an admin cluster using a two-step process:

**Step 1:** Create a [`VerrazzanoManagedCluster`](../../reference/api/multicluster/verrazzanomanagedcluster "api docs") resource in the `verrazzano-mc` namespace of the admin cluster.

**Step 2:** Retrieve the Kubernetes manifest file generated in the `VerrazzanoManagedCluster` resource and apply it on
   the managed cluster to complete the registration.

When a managed cluster is registered, the following will happen:

- Immediately after the first registration step, the admin cluster begins scraping Prometheus metrics from the newly
   registered managed cluster.
- After both steps of the registration are complete, the managed cluster begins polling the admin cluster for
   `VerrazzanoProject` resources and multicluster resources, which specify a `placement` in this managed cluster.
    -  Any `VerrazzanoProject` resources placed in this managed cluster are retrieved, and the corresponding namespaces
   and security permissions (`RoleBindings`) are created in the managed cluster.
    - Any multicluster resources that are placed in this managed cluster, and are in a `VerrazzanoProject` that is
       also placed in this managed cluster, are retrieved, and created or updated on the managed cluster. The
       underlying resource represented by the multicluster resource is unwrapped, and created or updated on the managed
       cluster. The managed cluster namespace of the multicluster resource and its underlying resource matches
       the admin cluster namespace of the multicluster resource.
- For `MultiClusterApplicationConfigurations` retrieved and unwrapped on a managed cluster, the application logs are
   sent to Elasticsearch on the admin cluster, and may be viewed from the Verrazzano-installed Kibana UI on the
   admin cluster. Likewise, application metrics will be scraped by the admin cluster and available from
   Verrazzano-installed Prometheus on the admin cluster.

### Detailed view of multicluster Verrazzano

This diagram shows a more detailed view of how multicluster Verrazzano works.

![](../../images/multicluster/MCConcepts.png)

### Try out multicluster Verrazzano

For more information, see the [API Documentation](../../reference/api/) for the resources described here.
To try out multicluster Verrazzano, see the [Hello World Helidon multicluster example application](https://github.com/verrazzano/verrazzano/tree/master/examples/multicluster/hello-helidon).