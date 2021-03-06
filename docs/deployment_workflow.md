# Deployment workflow

## Deploying the controllers

The following controllers need to be deployed :

* CAPI
* CAPBK or alternative
* CACPK or alternative
* CAPM3
* Baremetal Operator, with Ironic setup

## Requirements

The cluster should either :

* be deployed with an external cloud provider, that would be deployed as part of
  the userData and would set the providerIDs based on the BareMetalHost UID.
* Be deployed with the ProviderID set to be the BareMetalHostUUID
* be deployed with each node with the label "metal3.io/uuid" set to the
  BareMetalHost UID that is provided by ironic to cloud-init through the
  metadata `ds.meta_data.uuid`. This can be achieved by setting the following in
  the KubeadmConfig :

```yaml
nodeRegistration:
  name: '{{ ds.meta_data.name }}'
  kubeletExtraArgs:
    node-labels: 'metal3.io/uuid={{ ds.meta_data.uuid }}'
```

## Deploying the CRs

You can deploy the CRs all at the same time, the controllers will take care of
following the correct flow.
An outline of the workflow is below.

1. The CAPI controller will set the OwnerRef on the BaremetalCluster referenced
   by the Cluster, on the KubeadmControlPlane, and all machines, KubeadmConfig
   and BareMetalMachines created by the user or by a MachineDeployment.
1. The CAPM3 controller will verify the controlPlaneEndpoint field and populate
   the status with ready field set to true.
1. The CAPI controller will set infrastructureReady field to true on the Cluster
1. The CAPI controller will set the OwnerRef
1. The KubeadmControlPlane controller will wait until the cluster has
   infrastructureReady set to true, and generate the first machine,
   BareMetalMachine and KubeadmConfig.
1. CABPK will generate the cloud-init output for this machine and create a
   secret containing it.
1. The CAPI controller will copy the userData secret name into the machine
   object and set the bootstrapReady field to true.
1. Once the machine has userdataSecretName, OwnerRef and bootstrapReady properly
   set, the CAPM3 controller will select, if possible, a BareMetalHost that
   matches the criteria, or wait until one is available. If matched, the CAPM3
   controller will create a secret with the userData. and set the BareMetalHost
   spec accordingly to the BareMetalMachine specs.
1. The BareMetal Operator will then start the deployment.
1. After deployment, the BaremetalHost will be in provisioned state. However,
   initialization is not complete. If deploying without cloud provider, CAPM3
   will wait until the target cluster is up and the node appears. It will fetch
   the node by matching the label `metal3.io/uuid=<bmh-uuid>`. It will set the
   providerID to `metal3://<bmh-uuid>`. The BareMetalMachine ready status will
   be set to true and the providerID will be set to `metal3://<bmh-uuid>` on the
   BareMetalMachine.
1. CAPI will access the target cluster and compare the providerID on the node to
   the providerID of the Machine, copied from the BaremetalMachine. If matching,
   the control plane initialized status will be set to true and the machine
   state to running.
1. CACPK will then do the same for each further controller node until reaching
   the desired replicas number, one by one, triggering the same workflow.
   Meanwhile, as soon as the controlplane is initialized, CABPK will generate
   the user data for all other machines, triggering their deployment.

## Deletion

Deleting the cluster object will trigger the deletion of all related objects
except for KubeadmConfigTemplates, BareMetalMachineTemplates and BareMetalHost,
and the related secrets.
