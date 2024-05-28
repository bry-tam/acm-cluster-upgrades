## Introduction
The resources in this repository are used to demonstrate how to use RHACM Policy to upgrade OpenShift clusters.  You can see a video of this demonstration at https://www.youtube.com/watch?v=d7Gw-oVy1i4


There are two upgrades present; the `policy` directory contains resources to perform an upgrade.  The `talm` directory contains the same resource, plus the additional configuration to perform the upgrade using the [TALM Operator](https://docs.openshift.com/container-platform/4.12/scalability_and_performance/ztp_far_edge/cnf-talm-for-cluster-upgrades.html).

Each directory creates the policies with use of the [PolicyGenerator](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html-single/governance/index#policy-generator) and makes use of `PolicySets`.

## Upgrade a cluster
There are two resources needed to execute an upgrade.  First is the acknowledgment of the api deprecations for the specific release.  This is required when upgrading from 4.11->4.12, 4.12->4.13 and so on - but would not be needed for a z-stream release.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: admin-acks
  namespace: openshift-config
data:
  ack-4.11-kube-1.25-api-removals-in-4.12: "true"
  ack-4.12-kube-1.26-api-removals-in-4.13: "true"
```

The second resource is the `ClusterVersion` which specifies details of the version and upgrade.
```
apiVersion: config.openshift.io/v1
kind: ClusterVersion
metadata:
  name: version
spec:
  channel: fast-4.13
  upstream: https://api.openshift.com/api/upgrades_info/v1/graph
  desiredUpdate:
    force: false
    version: 4.13.15
    image: ''
status:
  history:
    - version: '{{ (lookup "config.openshift.io/v1" "ClusterVersion" "" "version").spec.desiredUpdate.version }}'
      state: "Completed"
```

Details of the values in the `ClusterVersion` explained:

- `channel` - Specifies the upgrade channel to use for the version graph
- `upstream` - URL pointing to the upgrade graph.  Connected cluster URL shown, in a disconnected environment this could point at a locally hosted [OpenShift Update Service](https://docs.openshift.com/container-platform/4.12/architecture/architecture-installation.html#update-service-about_architecture-installation).  Setting this allows us to specify the version and not the release image to use, making management much easier.
- `desiredUpdate.version` - Specifies the version to upgrade the cluster to.  Version must be available in the `channel` and must have a valid upgrade edge from the current version.  Upgradeability can be verified in the Customer Portal [Upgrade Graph Lab](https://access.redhat.com/labs/ocpupgradegraph/update_path/)
- `desiredUpdate.image` - Set to an empty value to remove any images specified during previous upgrades.  This is set by the OpenShift Console when performing an upgrade with the UI.
- `status` - Ensures the policy remains noncompliant until the upgrade has completed.


The `talm` directory makes use of the same policies, except they are created with the `remediationAction` set to "inform".  This prevents RHACM from applying the policy.  The TALM Operator will use to `ClusterGroupUpgrade` to make a copy of the policy, set the `remediationAction` to "enforce", create a Placement based on the clusters defined; once TALM has created these objects RHACM will enforce them to begin the upgrade.

