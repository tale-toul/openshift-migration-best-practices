[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Running the migration <](./running-the-migration.md) Troubleshooting
---
# Troubleshooting

This section describes common troubleshooting procedures.

* **[MTC data model](#mtc-data-model)**
* **[Debugging tips](#debugging-tips)**
* **[Using `must-gather`](#using-must-gather)**
* **[Performance metrics](#performance-metrics)**
* **[Cleaning up a failed migration](#cleaning-up-a-failed-migration)**
* **[Deleting the MTC Operator and resources](#deleting-the-mtc-operator-and-resources)**

Upstream doc for improving debug experience
* https://github.com/konveyor/enhancements/tree/master/enhancements/debug

Debug flowchart (in progress)
* [MTC Debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0)

# MTC data model

The following is a diagram of the CRDs that make up the MTC data model for context.
Each of these objects are standard Kubernetes CRDs, and the user can therefore use the
normal REST CRUD operations for acting on then via the kubectl and oc clients,
or even the HTTP interface directly:

TODO: Need to update diagram with MigAnalytic and MigHook

![CRD Architecture](./images/CRDArch.png)

# Debugging tips

You can view migration plan resources in the MTC web console:

1. Click the Options menu beside a migration plan and select **View migration plan resources**.

    The migration plan resources are displayed as a tree.
   
2. Click the Copy button to copy the `oc get <resource>` command.

3. Click **View Raw** to inspect the resource.

![Resource Debug Kebab Option](./images/ResourceDebugKebabOption.png)

![Debug Tree UI](./images/DebugTree.png)

Typically, the objects that you are interested in depend on the stage at which the
migration failed. The [MTC debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0) provides information about what objects are relevant depending on this failure stage.

> NOTE: Stage migrations will only have a single pair of Backup and Restore objects,
> while a Final migration will have *two* pairs of Backup and Restore objects.
> This can be considered a low level implementation detail, but the initial
> backup is performed to capture the original, unaltered state of the application
> and its Kubernetes objects. It is the source of truth. The application is then
> quiesced and a stage backup is performed to back up the storage-
> related resources (PV, PVC, the data itself). These storage objects are restored
> on the target cluster, followed by a final restore to restore from the original
> application's source of truth.

## Querying the cli

The migration debug tree can be viewed and traced via label selectors. For example,
to retrieve all migmigrations that are associated with a particular plan:

```
$ oc get migmigration -l 'migration.openshift.io/migplan-name=test'
NAME                                  READY  PLAN  STAGE  ITINERARY  PHASE
09a8bf20-fdc5-11ea-a447-cb5249018d21         test  false  Final      Completed
```

Notice the columns show select useful information about the migration, such as
the associated plan name, itinerary step, and phase.

TODO: Need to update with Backup/Restore queries

# Using `must-gather`

You can use the `must-gather` tool to collect information for troubleshooting or for opening a customer support case on the [Red Hat Customer Portal](https://access.redhat.com/). The `openshift-migration-must-gather-rhel8` image collects migration-specific logs and Custom Resource data that are not collected by the default `must-gather` image.

Run the `must-gather` command on your cluster:
````
$ oc adm must-gather --image=registry.redhat.io/rhcam-1-2/openshift-migration-must-gather-rhel8
````

## Previewing metrics on local Prometheus server

`must-gather` can be used to produce a dump of the last day of metrics data,
which can then be viewed with a local Prometheus instance following the
instructions in the [konveyor must-gather repo](https://github.com/konveyor/must-gather#preview-metrics-on-local-prometheus-server).

# Performance metrics

Details about the metrics that are recorded by the MTC controller can be found
in the [mig-operator documentation](https://github.com/konveyor/mig-operator/blob/master/docs/usage/Metrics.md#accessing-mig-controller-prometheus-metrics),
including a [set of useful queries](https://github.com/konveyor/mig-operator/blob/master/docs/usage/Metrics.md#useful-queries) that are available for performance monitoring.

# Cleaning up a failed migration

## Deleting resources

Ensure stage pods have been cleaned up. If a migration fails during stage or copy,
the 'stage' pods will be retained to allow debugging. Before proceeding with a
reattempt of the migration the stage pods need to be manually removed.

## Unquiescing an application

If your application was quiesced during migration, you should unquiesce it by scaling it back to its initial replica count.

This can be done manually by editing the deployment primitive (Deployment, DeploymentConfig, etc.) and setting the `spec.replicas` field back to its original, non-zero value:

```
$ oc edit deployment <deployment_name>
```

Alternatively, you can scale your deployment with the oc scale cmd:

```
$ oc scale deployment <deployment_name> --replicas=<desired_replicas>
```

## Note on labels applied to help track what was migrated

When quiescing a source application, MTC will annotate the original replica
count on the deployment object for reference:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    migration.openshift.io/preQuiesceReplicas: "1"
```

# Deleting the MTC Operator and resources

The following procedure removes the MTC Operator and cluster-scoped resources:

1. Delete the Migration Controller and its resources:
    ```` 
    $ oc delete migrationcontroller <resource_name>
    ````
    Wait for the MTC Operator to finish deleting the resources.

2. Uninstall the MTC Operator:
    * OpenShift 4: Uninstall the Operator in the [web console](https://docs.openshift.com/container-platform/4.5/operators/olm-deleting-operators-from-cluster.html) or by running the following command: 
    ````
    $ oc delete ns openshift-migration
    ````
    * Openshift 3: Uninstall the operator by deleting it:
    ````
    $ oc delete -f operator.yml
    ````

4. Delete the cluster-scoped resources:
    * Migration custom resource definition:
    ````
    $ oc get crds | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete crd {}
    ````  
    * Velero custom resource definition:
    ````
    $ oc get crds | grep velero | awk '{print $1}' | xargs -I{} oc delete crd {}
    ````  
    * Migration cluster role:
    ````
    $ oc get clusterroles | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
    ````  
    * Velero cluster role:
    ````
    $ oc get clusterroles | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
    ````  
    * Migration cluster role bindings:
    ````
    $ oc get clusterrolebindings | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
    ````  
    * Velero cluster role bindings:
    ````
    $ oc get clusterrolebindings | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
    ```` 

