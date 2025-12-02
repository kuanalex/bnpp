## BNPP CPD upgrade 5.0.3 to 5.2.2

## Author: Alex Kuan (alex.kuan@ibm.com)

From:

```
CPD: 5.0.3
OCP: 4.17.31
Storage: FDF 2.9.1
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate
```

To:

```
CPD: 5.2.2
OCP: 4.17.31
Storage: FDF 2.9.1
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate
```

Upgrade flow and steps

```
1. CPD 5.0.3 precheck
2. Update cpd-cli and env variables script for 5.2.2
3. Backup CPD 5.0.3 CRs, cpd-instance, and cpd-operators namespaces
4. Upgrade shared cluster components (ibm-cert-manager,ibm-licensing,scheduler)
5. Prepare to upgrade an instance of IBM Software Hub
6. Upgrade an instance of IBM Software Hub
7. Upgrade CPD services (db2oltp,datagate)
8. Potential Issues
9. Validate CPD upgrade (customer acceptance test)
```


## 1. CPD 5.2.2 pre-check

Use a client workstation with internet (bastion or infra node) to download OCP and CPD images, and confirm the OS level, ensuring the OS is RHEL 8/9

```
cat /etc/redhat-release
```

Test internet connection, and make sure the output from the target URL and it can be connected successfully:

```
curl -v https://github.com/IBM
```

Prepare customer\'s IBM entitlement key

Log in to <https://myibm.ibm.com/products-services/containerlibrary> with the IBMid and password that are associated with the entitled software.

On the Get entitlement key tab, select Copy key to copy the entitlement key to the clipboard.

Save the API key in a text file.

Make sure free disk space more than 500 GB (to download images and pack the images into a tar ball)

```
df -lh
```

Collect OCP and CPD cluster information

Log into OCP cluster from bastion node

```
oc login $(oc whoami --show-server) -u kubeadmin -p <kubeadmin-password>
```

Review OCP version

```
oc get clusterversion
```

Review storage classes

```
oc get sc
```

Review OCP cluster status

Make sure all nodes are in ready status

```
oc get nodes
```

Make sure all mc are in correct status, UPDATED all True, UPDATING all False, DEGRADED all False

```
oc get mcp
```

Make sure all co are in correct status, AVAILABLE all True, PROGRESSING all False, DEGRADED all False

```
oc get co
```

Make sure there are no unhealthy pods, if there are, please open an IBM support case to fix them.

```
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed' 
```

Get CPD installed projects

```
oc get pod -A | grep zen
```

Get CPD version and installed components

```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

or

```
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check the scheduling service, if it is installed but not in ibm-common-services project, uninstall it

```
oc get scheduling -A
```

Check install plan is automatic

```
oc get ip -A
```

Check the spec of each CPD custom resource, remove any patches before upgrading

```
oc project ${PROJECT_CPD_INST_OPERANDS}
```

```
for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}'); do echo "************* $i *************"; for x in $(oc get $i --no-headers | awk '{print $1}'); do echo "--------- $x ------------"; oc get $i $x -o jsonpath={.spec} | jq; done ; done
```

Probe IBM registry (if required)

```
podman login cp.icr.io -u cp -p ${IBM_ENTITLEMENT_KEY}
```


## 2. Update cpd-cli and environment variables script for 5.2.2

Download and unpack the latest cpd-cli release

```
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.2/cpd-cli-linux-EE-14.2.2.tgz && gzip -d cpd-cli-linux-EE-14.2.2.tgz && tar -xvf cpd-cli-linux-EE-14.2.2.tar && rm -rf cpd-cli-linux-EE-14.2.2.tar
```

Add the cpd-cli to your PATH variable, for example

```
export PATH=/root/cpd-cli-linux-EE-14.2.2-2727:$PATH
```

Update your environment variables script

```
vi cpd_vars.sh
```

Update the Version field and save your changes

```
VERSION=5.2.2
```

Source your environment variables

```
source cpd_vars.sh
```

Launch olm-utils-play-v3 container

```
cpd-cli manage restart-container
```


## 3. Backup CPD 5.2.1 CRs, cp4d and cp4d-operators namespaces

Create a new directory and store the output of the following commands in that directory

```
mkdir cpdbackup && cd cpdbackup && oc project ${PROJECT_CPD_INST_OPERANDS}
for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}'); do echo "************* $i *************"; for x in $(oc get $i --no-headers | awk '{print $1}'); do echo "--------- $x ------------"; oc get $i $x -oyaml > bak-$x.yaml; done ; done
```

**Note: The following 'oc adm' commands can be time-consuming and should be collected during the pre-upgrade Health Check activity**

Backup the current state of operands in your backup folder of choice:

```
mkdir operandsbackup && cd operandsbackup && oc adm inspect -n ${PROJECT_CPD_INST_OPERANDS} --dest-dir=source-${PROJECT_CPD_INST_OPERANDS} $(oc api-resources --namespaced=true --verbs=get,list --no-headers -o name | tr '\n' ',' | sed 's/,$//')
```

Backup the current state of operators in your backup folder of choice:

```
mkdir operatorsbackup && cd operatorsbackup && oc adm inspect -n ${PROJECT_CPD_INST_OPERATORS} --dest-dir=source-${PROJECT_CPD_INST_OPERATORS} $(oc api-resources --namespaced=true --verbs=get,list --no-headers -o name | tr '\n' ',' | sed 's/,$//')
```


## 4. Upgrade shared cluster components (ibm-cert-manager,ibm-licensing)

Determine which project the License Service is in

```
oc get deployment -A | grep ibm-licensing-operator
```

Upgrade the Certificate manager and License Service

The License Service is in the ${PROJECT_LICENSE_SERVICE} or ${PROJECT_CS_CONTROL} project

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster

```
${CPDM_OC_LOGIN}
```

Upgrade shared cluster components

```
./cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_CS_CONTROL} \
--case_download=false
```

Wait for the cpd-cli to return the following message before proceeding to the next step:

[SUCCESS] ... The apply-cluster-components command ran successfully

Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running or Completed:

```
oc get pods --namespace=${PROJECT_CERT_MANAGER}
```

Confirm that the License Service pods are Running or Completed

```
oc get pods --namespace=${PROJECT_CS_CONTROL}
```


## 5. Prepare to upgrade an instance of IBM Software Hub (est. 1-2 minutes)

Validate the health of your cluster, nodes, operators, and operands before proceeding with the upgrade:

```
cpd-cli health operators \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS}
```

Results should read "SUCCESS..."

```
cpd-cli health operands \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--include_ns=${PROJECT_CPD_INST_OPERATORS}
```

Results should read "SUCCESS..."

```
cpd-cli health cluster
```

Results should read "SUCCESS..."

```
cpd-cli health nodes
```

Results should read "SUCCESS..."

[Next, apply entitlements](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=puish-applying-your-entitlements-3) (est. 1-2 minutes):

Run the apply-entitlement command for each solution that is installed or that you plan to install in this instance

Apply entitlements for CP4D:

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-standard
```

## 6. Upgrade an instance of IBM Software Hub

Upgrade the required operators and custom resources for the instance:

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster

```
${CPDM_OC_LOGIN}
```

Review the license terms for the software that you plan to install:

```
cpd-cli manage get-license \
--release=${VERSION} \
--license-type=SE
```

Upgrade the required operators and custom resources for the instance (est. 60 minutes):

```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--file_storage_class=${STG_CLASS_FILE} \
--run_storage_tests=false \
--case_download=false
```

Monitor the upgrade progress of the custom resources with the following commands:

```
oc get ZenService lite-cr -n cpd-instance -o yaml
```

```
oc get ibmcpd ibmcpd-cr -n cpd-instance -o yaml
```

Upgrade the operators in the operators project (est. 15 minutes):

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true \
--case_download=false
```

Wait for the cpd-cli to return the following message before proceeding to the next step:

[SUCCESS]... The apply-olm command ran successfully

Confirm that the operator pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Proceed with upgrading the services next


## 7. Upgrade CPD services (db2oltp,datagate,dmc)

From the BNPP runbook, all CPD services are upgraded at the same time:

```
cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=${COMPONENTS} \
--license_acceptance=true \
--upgrade=true
```

In order to have more visibility into each service upgrade, you can optionally upgrade the services in sequential order, as follows:

### Upgrade Db2 custom resource (est. 10 minutes)

**Before you proceed, Stop Data Gate synchronization in the UI**

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster:

```
${CPDM_OC_LOGIN}
```

Update the custom resource for Db2:

```
cpd-cli manage apply-cr \
--components=db2oltp \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=db2oltp
```

You can also monitor the status of the Db2 upgrade with the following command:

```
oc get db2oltpservice.databases.cpd.ibm.com -n cpd-instance -o yaml
```

Db2 is upgraded when the apply-cr command returns:

```
[SUCCESS]... The apply-cr command ran successfully
```


### Upgrade Db2 instances (est. 10 minutes)

**Before you proceed, ensure your Db2 license is upgraded and CPD profile are set up**
-   [Upgrading the license before you
    deploy Db2](https://www.ibm.com/docs/en/SSNFH6_5.2.x/svc-db2/db2-update-lic.html)

-   [Creating a profile to use the cpd-cli management
    commands](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cli-creating-cpd-profile)
	

**Potential issue during the upgrade of Db2 service instance with db2ckupgrade.sh utility**

Db2ckupgrade.sh utility runs a job that must be scheduled to the same node as the Db2 engine pod. To work around this issue, restrict the job to run on the same node as the engine pod (using a cordon).

**Potential issue during the upgrade of Db2 service instance related to tempspace1**

During the Db2 instance upgrade you may observe issues related to tempspace1, "Table space access is not allowed." This problem is related to the usage of local storage and results in missing files and missing folder structures within the Db2u pod. To address this, you will need to create a specific directory structure inside the Db2u pod and copy the container tag to this location. RSH into the Db2u pod and run the following commands to create the folder structure and copy the container tag to the same location. Please keep in mind that the container name will change and that these steps assume that the file paths are identical across both systems, IPC1 and IPC2.

```
mkdir -p /mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP

```

```
cp SQLTAG.NAM /mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP
```

```
chmod 600 /mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP/SQLTAG.NAM
```

```
chown db2inst1:db2iadm1 /mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP/SQLTAG.NAM
```


After these steps, confirm that Db2 can start and then once you can connect to Db2, run the below commands as Db2 instance owner to take the backup of the container tags, as this backup will be used to restore the container tags when the pod starts again:

```
sudo rsync -rdgop --numeric-ids --checksum --exclude '*TLB' --exclude '*TDA' --exclude '*TBA' /mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/ /mnt/blumeta0/local-backup
```

Confirm the folder structure within /mnt/blumeta0/local-backup:

```
ls -laR /mnt/blumeta0/local-backup
```

Make sure the folder structure is:

```
 /mnt/blumeta0/local-backup/NODE000/*
```

Once the folder structure is confirmed, proceed with the Db2 instance upgrade accordingly

Prepare for the Db2 instance upgrade by obtaining a list of Db2 service instances:

To obtain value for --profile run cat $HOME/.cpd-cli/config as root:

```
cpd-cli service-instance list \
--service-type=db2oltp \
--profile=${CPD_PROFILE_NAME}
```

Set the INSTANCE_NAME to the name of the service instance:

```
export INSTANCE_NAME=<instance-name>
```

Check whether your Db2 service instances are in running state:

```
cpd-cli service-instance status ${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

**Before you upgrade Db2 instance, cordon all nodes other than the Db2 node. Remove any taints in the Db2 node as well**

Upgrade the db2oltp service instance (est. 10 minutes)

```
cpd-cli service-instance upgrade \
--service-type=db2oltp \
--instance-name=${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME}
```

If the Db2 instance upgrade starts looping, with error messages related to licensing, you may need to update the db2u-lic secret with the base64 encoded license key

Update the db2u-lic secret with the new Db2 license:

```
oc get secret | grep db2u-lic
```

```
oc edit secret db2u-lic
```

Verify the service instance upgrade by running the following command and waiting for the status to change to Ready:

```
oc get db2ucluster <instance_id> -o jsonpath='{.status.state} {"\n"}'
```

Run the following command to check the status of your Db2 service instances:

```
cpd-cli service-instance status ${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

Run the following command to check that the service instances have updated:

```
cpd-cli service-instance list \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

After upgrading Db2, you need to update the configmap

Run the following commands to patch the instance.json configmap

Run the following command to retrieve the values for the variables, CM_NAME, NAMESPACE and NEW_VERSION:

```
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME} --service-type=db2oltp
```

CM_NAME: Db2u instance configmap name. CM_NAME is obtained like this: <service_type>-<instance_id>-<service_type>-cm

For example:

```
db2oltp-1689782702423826-db2oltp-cm
```

NAMESPACE: Cluster namespace where Db2 instance and the configmap are installed

For example:

```
cpd-instance
```

NEW_VERSION: Upgraded Db2 instance version

For example:

```
11.5.9.0-cn5-amd64
```

[Copy the bash script](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=u-upgrading-from-version-51-36) to your local workstation and update the permissions of the bash script using chmod

Update the values for the CM_NAME, NAMESPACE, and NEW_VERSION variables in the bash script and run the bash script.

Continue with the upgrade of Data Gate custom resource

### Update Data Gate custom resource (est. 10 minutes):

Update the custom resource for Data Gate:

```
cpd-cli manage apply-cr \
--components=datagate \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true
```

Validating the upgrade: Data Gate is upgraded when the apply-cr command returns:

[SUCCESS]... The apply-cr command ran successfully

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=datagate
```

**Potential issue during the upgrade of Data Gate service instances**

During Data Gate instance upgrade a dg-1750101262664465-backup-head-job pod runs, but it should be scheduled to any node where Db2 is not running. For this, cordon the node hosting Db2, and after the backup-head-job pod is completed, uncordon the same node

### Upgrade the Data Gate service instances (est. 18 minutes):

Ensure the CPD profile is set up:

-   [Creating a profile to use the cpd-cli management
    commands](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cli-creating-cpd-profile)

```
cpd-cli service-instance upgrade \
--all \
--service-type=dg \
--profile=${CPD_PROFILE_NAME} \
--watch
```

Verifying the service instance upgrade

Run the following command and wait for the status to change to Completed:

```
oc get dginstance <instance_id> -o jsonpath='{.status.datagateInstanceStatus} {"\n"}'
```

Run the following command and see if the Provision status has changed to UPGRADED:

```
watch cpd-cli service-instance list --profile=${CPD_PROFILE_NAME} --service-type dg
```

**Potential issue after the upgrade of Data Gate service instances**

After Data Gate instances have been upgraded, it is possible that the configuration settings of the Db2 target database are not correctly migrated during the upgrade

Missing Db2 configuration settings can cause a variety of issues, as experienced most recently during the E3-IPC1 upgrade, in which the DB2COMM=TCPIP,SSL was missing

To resolve this, we had to add the missing Db2 setting by using 'db2set DB2COMM=TCPIP,SSL'

These configurations should persist through a Db2u pod recycle

Keep track of your Db2 configuration settings prior to upgrading to ensure you have a list of settings which you can restore

You can also [change configuration settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings) after you deploy your instance

### Update Db2 Data Management Console custom resource (est. 10 minutes):

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster:

```
${CPDM_OC_LOGIN}
```

Update the custom resource for Db2:

```
cpd-cli manage apply-cr \
--components=dmc \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=dmc
```

You can also monitor the status of the Db2 upgrade with the following command:

```
oc get db2oltpservice.databases.cpd.ibm.com -n cpd-instance -o yaml
```

DMC is upgraded when the apply-cr command returns:

```
[SUCCESS]... The apply-cr command ran successfully
```

## 8. All Potential Issues

1. Setup-instance command will fail with 'imagepullbackoff' errors if the storage test images are missing. Mirror the storage test images ahead of time or exclude the '--run_storage_tests' flag

2. For imagepullbackoff errors, ensure that all required images are mirrored to the private registry, for example:

```
skopeo copy docker://icr.io/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d docker://lpdza532.phx.aexp.com:5000/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d \--all \--remove-signatures \--authfile=/var/registry/oc4.7/installer/pullsecret/merged_pullsecret.json
```

```
skopeo copy docker://icr.io/cpopen/cpd/olm-utils-v3@sha256:5a5b9b563756b3a41b22b2c9b6e940d3ceed6e12100e351809eb7f09ed058905 docker://lppza417.gso.aexp.com:5000/cpopen/cpd/olm-utils-v3@sha256:5a5b9b563756b3a41b22b2c9b6e940d3ceed6e12100e351809eb7f09ed058905 --all --remove-signatures --authfile=/root/merged_pullsecret.json
```

3. For insufficient CPU, insufficient memory, untolerated taint issues, ensure there are enough resources available on the cluster. Perform a pre-upgrade health check to ensure the cluster is in a healthy state
prior to the upgrade

4. If the Db2 license is not upgraded to the compatible version you may run into issues during Db2 instance upgrade. Follow the steps to "[Upgrade the license before you deploy Db2](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=setup-upgrading-license-before-you-deploy-db2)"

5. During the Db2 instance upgrade, the Db2ckupgrade.sh utility runs a job that must be scheduled to the same node as the Db2 engine pod. To resolve this issue, restrict the job to run on the same node as the
engine pod. Use a cordon, to ensure that this job is scheduled to the node hosting Db2. In previous upgrades, we've cordoned nodes lpqzo504, lpqzo505, and lpqzo506, such that the job would be scheduled to lpqzo507

6. During the Db2 instance upgrade you may observe issues related to tempspace1, "Table space access is not allowed." This problem is related to the usage of local storage and results in missing files and missing
folder structures within the Db2 pod. To address this, you will need to create a specific directory structure inside the Db2u pod and copy the container tag to this location.

7. During Data Gate instance upgrade a dg-1750101262664465-backup-head-job pod runs, but it should be scheduled to any node where Db2 is not running. For this cordon the node hosting Db2, and after the backup-head-job pod is completed, uncordon the same node

8. After Data Gate instances have been upgraded, it is possible that the configuration settings of the Db2 target database are not correctly migrated during the upgrade. Missing Db2 configuration settings can
cause a variety of issues, as experienced most recently during the E3-IPC1 upgrade, in which the DB2COMM=TCPIP,SSL was missing. To resolve this, we had to add the missing Db2 setting by using 'db2set
DB2COMM=TCPIP,SSL'. These configurations should persist through a Db2u pod recycle. Keep track of your Db2 configuration settings prior to upgrading to ensure you have a list of settings which you can restore.
You can also [change configuration settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings) after you deploy your instance

This marks the end of the installation of IBM Software Hub, db2oltp and Data Gate

## 9. Validate CPD upgrade (customer acceptance test)

End of document
