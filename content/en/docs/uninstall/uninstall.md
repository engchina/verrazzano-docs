---
title: "Uninstall"
linkTitle: "Uninstall"
description: "Learn how to uninstall Verrazzano"
weight: 3
draft: false
---

## Uninstall considerations
Before uninstalling Verrazzano, you should delete your Verrazzano applications because they may not function properly after the uninstall is done.

When you uninstall Verrazzano:
* All of the Verrazzano components are uninstalled
* The CRDs installed by Verrazzano are not deleted
* Any applications that were deployed will still exist, but they may not be functional

## Perform the uninstall

You can uninstall Verrazzano using the [Verrazzano CLI]({{< relref "/docs/setup/install/installation.md" >}}) or with [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/).
See the following respective sections.

{{< tabs tabTotal="2" >}}
{{< tab tabName="vz" >}}
<br>

1. Uninstall Verrazzano.
{{< clipboard >}}

 ```shell
  $ vz uninstall
  ```
{{< /clipboard >}}

2. Wait for the uninstall to complete.
   The uninstall logs from the Verrazzano platform operator will be streamed to the command window until the uninstall has completed or until the default timeout (20m) has been reached.

   The following is an example of the output:
{{< clipboard >}}
   ```shell
   Uninstalling Verrazzano
   2022-11-22T16:31:20.377Z info Reconciling Verrazzano resource default/verrazzano, generation 2, version 1.4.2
   2022-11-22T16:31:20.377Z info Deleting Verrazzano installation
   2022-11-22T16:31:20.418Z info Uninstalling components
   2022-11-22T16:31:20.418Z info Uninstalling Verrazzano default/verrazzano
   ...
   ```
{{< /clipboard >}}

{{< /tab >}}
{{< tab tabName="kubectl" >}}
<br>

To delete a Verrazzano installation, delete the Verrazzano custom resource you used to
install it into your cluster.

The following example starts a deletion of a Verrazzano installation in the background and then
uses the `kubectl logs -f` command to tail the output of the pod performing the uninstall.

1. Get the name of the Verrazzano custom resource.
{{< clipboard >}}

   ```shell
   $ MYVZ=$(kubectl  get vz -o jsonpath="{.items[0].metadata.name}")
   ```
{{< /clipboard >}}
2. Delete the Verrazzano custom resource.  Once the delete is done, the Verrazzano uninstall will be complete.
{{< clipboard >}}

   ```shell
   $ kubectl delete verrazzano $MYVZ
   ```
{{< /clipboard >}}

If you want to see the uninstall logs during the deletion, you can view them from the `Verrazzano platform operator` with the following command:
{{< clipboard >}}
<div class="highlight">

```
$ kubectl logs -n verrazzano-install \
    -f $(kubectl get pod \
    -n verrazzano-install \
    -l app=verrazzano-platform-operator \
    -o jsonpath="{.items[0].metadata.name}") | grep '^{.*}$' \
    | jq -r '."@timestamp" as $timestamp | "\($timestamp) \(.level) \(.message)"'
```
</div>
{{< /clipboard >}}

{{< /tab >}}
{{< /tabs >}}

For troubleshooting help, see [Analysis Advice]({{< relref "/docs/troubleshooting/diagnostictools/analysisadvice/" >}}).
