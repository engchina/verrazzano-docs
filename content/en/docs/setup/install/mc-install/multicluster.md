---
title: "Install Multicluster"
description: "How to set up a multicluster Verrazzano environment"
weight: 1
draft: false
---

## Prerequisites

- Before you begin, read this document, [Verrazzano in a multicluster environment]({{< relref "/docs/concepts/VerrazzanoMultiCluster.md" >}}).
- To set up a multicluster Verrazzano environment, you will need two or more Kubernetes clusters. One of these clusters
will be *admin* cluster; the others will be *managed* clusters. For instructions on preparing Kubernetes platforms for installing Verrazzano, see [Platform Setup]({{< relref "/docs/setup/platforms/_index.md" >}}).

{{< alert title="NOTE" color="primary" >}}
If Rancher is not enabled, then refer to [Verrazzano multicluster installation without Rancher]({{< relref "docs/setup/install/mc-install/multicluster-no-rancher.md" >}})
because additional steps are required to register a managed cluster.
{{< /alert >}}

The following instructions assume an admin cluster and a single managed cluster. For each additional managed
cluster, simply repeat the managed cluster instructions.

## Install Verrazzano

To install Verrazzano on each Kubernetes cluster, complete the following steps:

1. On one cluster, install Verrazzano using the `dev` or `prod` profile; this will be the *admin* cluster.
2. On the other cluster, install Verrazzano using the `managed-cluster` profile; this will be a managed cluster. The `managed-cluster` profile contains only the components that are required for a managed cluster.
<br>**NOTE**: You also can use the `dev` or `prod` profile.

For detailed instructions on how to install and customize Verrazzano on a Kubernetes cluster using a specific profile,
see the [Installation Guide]({{< relref "/docs/setup/install/installation.md" >}}) and [Installation Profiles]({{< relref "/docs/setup/install/profiles.md" >}}).

## Register the managed cluster

To register the managed cluster using the VerrazzanoManagedCluster resource, complete the following steps:

1. Create the environment variables, `KUBECONFIG_ADMIN`, `KUBECONTEXT_ADMIN`, `KUBECONFIG_MANAGED1`, and
  `KUBECONTEXT_MANAGED1`, and point them to the kubeconfig files and contexts for the admin and managed cluster,
  respectively. You will use these environment variables in subsequent steps when registering the managed cluster. The
  following shows an example of how to set these environment variables.
{{< clipboard >}}
<div class="highlight">

   ```
   $ export KUBECONFIG_ADMIN=/path/to/your/adminclusterkubeconfig
   $ export KUBECONFIG_MANAGED1=/path/to/your/managedclusterkubeconfig

   # Lists the contexts in each kubeconfig file
   $ kubectl --kubeconfig $KUBECONFIG_ADMIN config get-contexts -o=name
   $ kubectl --kubeconfig $KUBECONFIG_MANAGED1 config get-contexts -o=name

   # Choose the right context name for your admin and managed clusters from the output shown and set the KUBECONTEXT
   # environment variables
   $ export KUBECONTEXT_ADMIN=<admin-cluster-context-name>
   $ export KUBECONTEXT_MANAGED1=<managed-cluster-context-name>
   ```

</div>
{{< /clipboard >}}

2. To begin the registration process for a managed cluster named `managed1`, apply the VerrazzanoManagedCluster resource on the admin cluster.
{{< clipboard >}}
<div class="highlight">

   ```
   # On the admin cluster
   $ kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
       apply -f <<EOF -
   apiVersion: clusters.verrazzano.io/v1alpha1
   kind: VerrazzanoManagedCluster
   metadata:
     name: managed1
     namespace: verrazzano-mc
   spec:
     description: "Test VerrazzanoManagedCluster resource"
   EOF
   ```

</div>
{{< /clipboard >}}

3. Wait for the VerrazzanoManagedCluster resource to reach the `Ready` status. At that point, it will have generated a YAML
   file that must be applied on the managed cluster to complete the registration process.
{{< clipboard >}}
<div class="highlight">

   ```
   # On the admin cluster
   $ kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
       wait --for=condition=Ready \
       vmc managed1 -n verrazzano-mc
   ```

</div>
{{< /clipboard >}}

4. Export the YAML file created to register the managed cluster.
{{< clipboard >}}
<div class="highlight">

   ```
   # On the admin cluster
   $ kubectl --kubeconfig $KUBECONFIG_ADMIN --context $KUBECONTEXT_ADMIN \
       get secret verrazzano-cluster-managed1-manifest \
       -n verrazzano-mc \
       -o jsonpath={.data.yaml} | base64 --decode > register.yaml
   ```

</div>
{{< /clipboard >}}

5. Apply the registration file exported in the previous step, on the managed cluster.
{{< clipboard >}}
<div class="highlight">

   ```
   # On the managed cluster
   $ kubectl --kubeconfig $KUBECONFIG_MANAGED1 --context $KUBECONTEXT_MANAGED1 \
       apply -f register.yaml

   # After the command succeeds, you may delete the register.yaml file
   $ rm register.yaml
   ```

</div>
{{< /clipboard >}}

## Next steps

- Verify your multicluster Verrazzano environment set up by following the instructions at [Verify Multicluster Installation]({{< relref "/docs/setup/install/mc-install/verify-install.md" >}}).
- Deploy multicluster example applications. See [Examples of using Verrazzano in a multicluster environment]({{< relref "/docs/samples/multicluster/_index.md" >}}).

{{< alert title="NOTE" color="primary" >}}
To deregister a managed cluster, see [Deregister a Managed Cluster]({{< relref "/docs/setup/install/mc-install/deregister-install.md" >}}).
{{< /alert >}}
