## BNPP CPD upgrade 5.0.3 to 5.2.2

## Author: Alex Kuan (alex.kuan@ibm.com)

From:

```
CPD: 5.0.3
OCP: 4.18.27
Storage: FDF 2.9.1
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate,dmc
```

To:

```
CPD: 5.2.2
OCP: 4.18.27
Storage: FDF 2.9.1
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate,dmc
```

Upgrade flow and steps

```
1. CPD 5.0.3 precheck
2. Update cpd-cli and environment variables script for 5.2.2
3. Prepare to run upgrades in a restricted network using a private container registry
4. Backup CPD 5.0.3 CRs, cp4data, and cpd-inst-operators namespaces
5. Upgrade shared cluster components (ibm-cert-manager,ibm-licensing,scheduler)
6. Prepare to upgrade an instance of IBM Software Hub
7. Upgrade an instance of IBM Software Hub
8. Upgrade CPD services (db2oltp,datagate,dmc)
9. Potential Issues
10. Validate CPD upgrade (customer acceptance test)
```


## 1. CPD 5.0.3 pre-check

### Use a client workstation with internet (bastion or infra node) to download OCP and CPD images, and confirm the OS level, ensuring the OS is RHEL 8/9

```
cat /etc/redhat-release
```

Test internet connection, and make sure the output from the target URL and it can be connected successfully:

```
curl -v https://github.com/IBM
```

### Prepare your IBM entitlement key

Log in to <https://myibm.ibm.com/products-services/containerlibrary> with the IBMid and password that are associated with the entitled software.

On the Get entitlement key tab, select Copy key to copy the entitlement key to the clipboard.

Save the API key in a text file.

### Make sure free disk space more than 500 GB (to download images and pack the images into a tar ball)

```
df -lh
```

### Collect OCP and CPD cluster information

Log into OCP cluster from bastion node

```
oc login $(oc whoami --show-server) -u kubeadmin -p <kubeadmin-password>
```

### Review OCP version

```
oc get clusterversion
```

### Review storage classes

```
oc get sc
```

### Review OCP cluster status and make sure all nodes are in ready status

```
oc get nodes
```

### Make sure all mc are in correct status, UPDATED all True, UPDATING all False, DEGRADED all False

```
oc get mcp
```

### Make sure all co are in correct status, AVAILABLE all True, PROGRESSING all False, DEGRADED all False

```
oc get co
```

### Make sure there are no unhealthy pods, if there are, please open an IBM support case to fix them.

```
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed' 
```

### Get CPD installed projects

```
oc get pod -A | grep zen
```

### Get CPD version and installed components

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

### Check the scheduling service, if it is installed but not in ibm-common-services project, uninstall it

```
oc get scheduling -A
```

### Check install plan is automatic

```
oc get ip -A
```

### Check the spec of each CPD custom resource, remove any patches before upgrading

```
oc project ${PROJECT_CPD_INST_OPERANDS}
```

```
for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}'); do echo "************* $i *************"; for x in $(oc get $i --no-headers | awk '{print $1}'); do echo "--------- $x ------------"; oc get $i $x -o jsonpath={.spec} | jq; done ; done
```

### Probe IBM registry (if required)

```
podman login cp.icr.io -u cp -p ${IBM_ENTITLEMENT_KEY}
```

Repair IAM Migration

### Verfication Step 1

Check if the infrastructure is affected

If ibm-iam-request is Running, continue with Verification Step 2

Review operand request details

```
oc get opreq
```

Example of failed output

```
NAME                            AGE     PHASE     CREATED AT
cloud-native-postgresql-opreq   4d18h   Running   2025-12-25T01:23:45Z
ibm-iam-request                 4d18h   Pending   2025-12-25T01:23:45Z
ibm-iam-service                 4d18h   Failed    2025-12-25T01:23:45Z
postgresql-operator-request     4d18h   Running   2025-12-25T01:23:45Z
zen-ca-operand-request          4d18h   Running   2025-12-25T01:23:45Z
zen-service                     4d18h   Running   2025-12-25T01:23:45Z
```

Identify ODLM pod and search logs for messages related to 'ibm-license-key-applied'

```
oc get pods -n cpd-inst-operators |grep operand-deployment-lifecycle-manager
```

```
operand-deployment-lifecycle-manager-8696fb7bbb-5sk7p             1/1     Running
```

```
oc logs operand-deployment-lifecycle-manager-8696fb7bbb-5sk7p  | grep ibm-license-key-applied
```

Example of concerning log messages

```
E0924 06:39:32.327436 1 reconcile_operand.go:508] Failed to reconcile k8s resource -- Kind: Cluster, NamespacedName: /common-service-db with error: failed to parse
path .metadata.annotations.ibm-license-key-applied from Secret cpd-inst-operators/postgresql-operator-controller-manager-config: ibm-license-key-applied is not found
- failed to parse path .metadata.annotations.ibm-license-key-applied from Secret cpd-inst-operators/postgresql-operator-controller-manager-config: ibm-license-key-applied
is not found
```

If the above error message is found, the infrastructure is affected by [Known Issue: DT401031](https://www.ibm.com/mysupport/s/defect/aCIKe000000g0riOAA/dt401031?language=fr)

Proceed with the following steps

Delete the create-postgres-license-config job

```
oc delete job -n cpd-inst-operators create-postgres-license-config
```

Delete the ODLM pod

```
oc delete pod -n cpd-inst-operators operand-deployment-lifecycle-manager-8696fb7bbb-5sk7p
```

Wait until the OperandRequest ibm-iam-request is Running

```
watch oc get operandrequest
```

Confirm that the ibm-iam operandrequests are in Running status

```
NAME                            AGE     PHASE     CREATED AT
ibm-iam-request                 4d18h   Running   2025-12-25T01:23:45Z
ibm-iam-service                 4d18h   Running   2025-12-25T01:23:45Z
```

Wait for the common-service-db pods to be created

```
watch "oc get pods -n cp4data |grep common-service-db"
```
common-service-db-1              1/1 Running   0  14m
common-service-db-2              1/1 Running   0  12m


### Verfication Step 2

Check if the mongodb service is still running

```
oc get service -n cp4data | grep -i mongodb
```

Delete the icp-mongodb and mongodb service

```
oc delete service -n cp4data icp-mongodb 
oc delete service -n cp4data mongodb
```

If the deletion does not complete, edit the service yaml and remove the "finalizer" section

Find the ibm-iam pod

```
oc get pod -n cpd-inst-operators | grep ibm-iam-operator 
```

Delete the ibm-iam pod

```
oc delete pod -n cpd-inst-operators ibm-iam-operator-7b755fc96d-t49hz
```

### Save the existing configuration

```
cd /apps/bastion/backup
```

```
oc get datagateinstanceservices dg1734441701891261 -o yaml > dg1734441701891261.yaml 
oc get dmc data-management-console -o yaml > data-management-console.yaml 
oc get db2uclusters db2oltp-1734440804666892 -o yaml > db2oltp-1734440804666892.yaml 
oc get cm -n cp4data c-db2oltp-1734440804666892-db2dbconfig -o yaml > c-db2oltp-1734440804666892-db2dbconfig.yaml 
oc get cm -n cp4data c-db2oltp-1734440804666892-db2dbmconfig -o yaml > c-db2oltp-1734440804666892-db2dbmconfig.yaml
```


## 2. Update cpd-cli and environment variables script for 5.2.2

### Prepare cpd-cli and environment variables

Connect to the GRAFANA server

DEV - s02vl9925125

QUALIFICATION - S02VL9925965

PROD - S02VL9926323

Retrieve the tar file containing the latest version of cpd-cli and the offline sources as prepared in a previous step

Prepare the File System

```
lvcreate -n lv_cpdcli -L 10G /dev/vg_apps
mke2fs -j -i 1024 /dev/vg_apps/lv_cpdcli
```

```
vi /etc/fstab
```

/dev/vg_apps/lv_cpdcli /apps/cpdcli ext4 defaults 1 2

```
systemctl daemon-reload
mkdir /apps/cpdcli
mount /apps/cpdcli
```

Unzip the file

```
cd /apps/cpdcli
tar -xvf cpd-cli-linux-SE-14.2.2-2727_with_sources.tar
chmod -R 777 /apps/cpdcli/cpd-cli-linux-SE-14.2.2-2727
```

Optionally, add the cpd-cli to your PATH variable, for example

```
export PATH=/root/cpd-cli-linux-SE-14.2.2-2727:$PATH
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

### Test connection to the artifactory

Install Podman and Docker

```
yum install docker -y
```

The username and password are in the directory

```
podman login cp-icr.artifactory-dogen.group.echonet.net.intra
podman login icr.artifactory-dogen.group.echonet.net.intra
podman login quay-io-oc.artifactory-dogen.group.echonet.net.intra
podman login redhat-io-docker-oc.artifactory-dogen.group.echonet.net.intra
```

If there is a problem, check the registries in /etc/containers/registries.conf

Launch the olm-utils-play-v3 container

```
cd /apps/cpdcli/cpd-cli-linux-SE-14.2.2-2727
```

```
cpd-cli manage restart-container
```

Confirm the olm container is running

```
podman ps
```


## 3. Prepare to run upgrades in a restricted network using a private container registry

### Obtaining the olm-utils-v3 image (skip, if not required)

If your cluster is in a restricted network, you must ensure that a supported version of the olm-utils-v3 image is on the client workstation from which you will run the installation commands. The latest version of the image is recommended

The steps that you complete depend on whether you plan to use the same workstation in the cluster network

Option 1 - You plan to use the same workstation inside the cluster network:

When the workstation is connected to the internet, run the following command to update the olm-utils-v3 image on the workstation:

```
cpd-cli manage restart-container
```

Wait for the cpd-cli to return the following messages:

```
[SUCCESS] ... Successfully pulled the container image icr.io/cpopen/cpd/olm-utils-v3:latest
[SUCCESS] ... Successfully started the container olm-utils-play
[SUCCESS] ... Container olm-utils-play has been re-created
```

Option 2 - You plan to use a different workstation inside the cluster network:

Before you begin, you must have the cpd-cli installed on a workstation in the cluster network

From a workstation that can connect to the internet, ensure that Docker or Podman is running on the workstation

Run the following command to save the olm-utils-v3 image to the client workstation:

s390x clusters:

```
cpd-cli manage save-image \
--from=icr.io/cpopen/cpd/olm-utils-v3:${VERSION}.s390x
```

This command saves the image as a compressed TAR file named icr.io_cpopen_cpd_olm-utils-v3_${VERSION}.s390x.tar.gz in the cpd-cli-workspace/olm-utils-workspace/work/offline directory

Transfer the compressed file to a client workstation that can connect to the cluster. Ensure that you place the TAR file in the work/offline directory:

s390x clusters:

```
icr.io_cpopen_cpd_olm-utils-v3_${VERSION}.s390x.tar.gz
```

From the workstation that can connect to the cluster, ensure that Docker or Podman is running on the workstation

Run the following command to load the olm-utils-v3 image on the client workstation:

s390x clusters:

```
cpd-cli manage load-image \
--source-image=icr.io/cpopen/cpd/olm-utils-v3:${VERSION}.s390x
```

The command returns the following message when the image is loaded:

```
Loaded image: icr.io/cpopen/cpd/olm-utils:latest.s390x
```

Set the OLM_UTILS_IMAGE environment variable to ensure that the cpd-cli uses the version of the image on the client workstation

s390x clusters:

```
export OLM_UTILS_IMAGE=icr.io/cpopen/cpd/olm-utils-v3:${VERSION}.s390x
```

Now that you've made the olm-utils-v3 image available on the client workstation, you're ready to download CASE packages

### Downloading CASE packages

If you will run the upgrade commands in a restricted network, you must have the CASE packages for the components that you plan to upgrade on the client workstation from which you will run the upgrade commands

To prepare to run upgrades in a restricted network, download the CASE packages for the components that you plan to install

Run the appropriate command depending on the site from which you plan to download the CASE packages:

GitHub:

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```

IBM Cloud Pak Open Container Initiative:

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--from_oci=true
```

The CASE packages for the specified components are downloaded to the work directory

**Important: If you transfer the CASE packages to other workstations, ensure that you complete the following steps on each workstation**

Change to the directory that contains the work directory

Set the following permissions on the work directory:

```
chown -R 1001 ./work
chmod -R 775 ./work
```

Restart the container:

```
cpd-cli manage restart-container
```

### Mirroring images to a private container registry

If you mirrored the images for IBM Cloud Pak® for Data Version 5.0 to a private container registry, you must mirror the images for IBM® Software Hub 5.2 to the private container registry before you upgrade your installation

Mirroring images directly to the private container registry:

Log in to the IBM Entitled Registry registry:

```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

Log in to the private container registry, the following command assumes that you are using private container registry that is secured with credentials:

```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PUSH_USER} \
${PRIVATE_REGISTRY_PUSH_PASSWORD}
```

If your private registry is not secured omit the following arguments:
- ${PRIVATE_REGISTRY_PUSH_USER}
- ${PRIVATE_REGISTRY_PUSH_PASSWORD}

Confirm that you have access to the images that you want to mirror from the IBM Entitled Registry and inspect the IBM Entitled Registry

You already have the CASE packages on the client workstation:

```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--inspect_source_registry=true
```

Download CASE packages from GitHub (github.com/IBM):

```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--inspect_source_registry=true
```

Download CASE packages from the IBM Cloud Pak Open Container Initiative repository:

```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--inspect_source_registry=true \
--from_oci=true
```

The output is saved to the list_images.csv file in the work/offline/${VERSION} directory

Check the output for errors, this command returns images that failed because of authorization errors or network errors:

```
grep "level=fatal" list_images.csv
```

The following applies to EDB Postgres Standard users only. If you purchased EDB Postgres Standard, run the following command to remove the EDB Postgres Enterprise images from the list of images that will be mirrored to the private container registry

Workstations that use the default cpd-cli-workspace/olm-utils-workspace/work directory:

```
sed -i -e '/edb-postgres-advanced/d' ./cpd-cli-workspace/olm-utils-workspace/work/offline/${VERSION}/.ibm-pak/data/cases/ibm-cpd-edb/*/ibm-cpd-edb-*-images.csv
```

Workstations that use the CPD_CLI_MANAGE_WORKSPACE environment variable:

```
sed -i -e '/edb-postgres-advanced/d' ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/${VERSION}/.ibm-pak/data/cases/ibm-cpd-edb/*/ibm-cpd-edb-*-images.csv
```

Mirror the images to the private container registry

You need to mirror models or optional images:

```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--groups=${IMAGE_GROUPS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

You don't need to mirror models or optional images:

```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

For each component, the command generates a log file in the work directory

**Tip: Run the following command to print out any errors in the log files:**

```
grep "error" mirror_*.log
```

Confirm that the images were mirrored to the private container registry

Inspect the contents of the private container registry:

```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false
```

The output is saved to the list_images.csv file in the work/offline/${VERSION} directory

Check the output for errors, the command returns images that are missing or that cannot be inspected:

```
grep "level=fatal" list_images.csv
```


## 4. Backup CPD 5.2.1 CRs, cp4data and cpd-inst-operators namespaces

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


## 5. Upgrade shared cluster components (ibm-cert-manager,ibm-licensing)

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


## 6. Prepare to upgrade an instance of IBM Software Hub (est. 1-2 minutes)

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

Apply entitlements for CP4D

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-standard
```

Disable Bedrock customiztion

```
oc get sts -n cp4data |grep db2u
```

Observe the output

```
c-db2oltp-1736767772824325-db2u
```

Run the set volume command

```
oc set volume statefulset/c-db2oltp-1736767772824325-db2u -n cp4data --remove --name=db2u-entrypoint-sh

oc set volume statefulset/c-db2oltp-1764799139398922-db2u -n cp4data --remove --name=db2u-entrypoint-sh
```

If you see this error message, you can proceed normally

```
error: statefulsets/c-db2oltp-1736767772824325-db2u volume 'db2u-entrypoint-sh' not found
```


## 7. Upgrade an instance of IBM Software Hub

Before you upgrade to IBM Software Hub, check whether the following common core services pods are running in this instance of IBM Cloud Pak for Data:

Check whether the global search pods are running:

```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep elasticsea-0ac3
```

If the command returns an empty response, proceed to the next step.

If the command returns a list of pods, review [Upgrades fail when global search is configured incorrectly](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=tiu-upgrades-fail-when-global-search-is-configured-incorrectly) to determine whether you have any configurations that could cause issues during upgrade.

Check whether the catalog-api pods are running:

```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep catalog-api
```

If the command returns an empty response, you are ready to upgrade IBM Software Hub.

If the command returns a list of pods, review the following guidance to determine how long the catalog-api service will be down during upgrade.

When you upgrade the common core services to IBM Software Hub Version 5.2, the underlying storage for the catalog-api service is migrated to PostgreSQL.

During the final stages of the migration, the catalog-api service is offline, and services that are dependent on the service are not available. The duration of the migration depends on the number of assets and relationships that are stored in the instance. The duration of the outage depends on the number of databases (projects, catalogs, and spaces) in the instance. In a typical upgrade scenario, the outage should be significantly shorter than the overall migration.

To determine how many databases will be migrated follow these steps:

Set the INSTANCE_URL environment variable to the URL of IBM Software Hub

```
export INSTANCE_URL=<URL>
```

To get the URL of the web client, run the following command:

```
cpd-cli manage get-cpd-instance-details \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Get the credentials for the wdp-service:

```
TOKEN=$(oc get -n ${PROJECT_CPD_INST_OPERANDS} secrets wdp-service-id -o yaml | grep service-id-credentials | cut -d':' -f2- | sed -e 's/ //g' | base64 -d)
```

Get the number of catalogs in the instance:

```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=catalogs&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Get the number of projects in the instance:

```curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=projects&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Get the number of spaces in the instance:

```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=spaces&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Add up the number of catalogs, projects, and spaces returned by the previous commands. Then, use the following table to determine approximately how long the service will be offline during the migration:

```
Databases                   Downtime for migration (approximate)
Up to 1,000 databases	    6 minutes
1,001 - 10,000 databases	20 minutes
10,001 - 70,000 databases	60 minutes
```

Save the following script on the client workstation as a file named precheck_migration.sh:

```
#!/bin/bash

# Default ranges for couchdb size
SMALL=30
LARGE=200

echo "Performing pre-migration checks"

check_resources(){
        scale_config=$1
        pvc_size=$(oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} database-storage-wdp-couchdb-0 --no-headers | awk '{print $4}')
        size=$(awk '{print substr($0, 1, length($0)-2)}' <<< "$pvc_size")

        if [[ "$size" -le "$SMALL" ]];then
          echo "The system is ready for migration. Upgrade your cluster as usual."
        elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$LARGE" ] && [[ $scale_config == "small" ]];then
          echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 6,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "3", "ephemeral-storage": "10Mi", "memory": "4Gi"},
    "limits": {"cpu": "8", "ephemeral-storage": "4Gi", "memory": "8Gi"}}
}}'
EOF
          echo
          echo "The system is ready for migration. Upgrade your cluster as usual."
        elif [[ $scale_config == "medium" ]];then
          echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 8,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "6", "ephemeral-storage": "10Mi", "memory": "6Gi"},
    "limits": {"cpu": "10", "ephemeral-storage": "6Gi", "memory": "10Gi"}}
}}'
EOF
          echo
          echo "Before you can start the upgrade, you must prepare the system for migration."
        fi
}

check_upgrade_case(){     
        scale_config=$(oc get ccs -n ${PROJECT_CPD_INST_OPERANDS} ccs-cr -o json | jq -r '.spec.scaleConfig')

        # Default case, scale config is set to small
        if [[ -z "${scale_config}" ]];then
          scale_config=small
        fi

        if [[ $scale_config == "large" ]];then
          echo "Before you can start the upgrade, you must prepare the system for migration."
        elif [[ $scale_config == "small" ]] || [[ $scale_config == "medium" ]];then
          check_resources $scale_config
        fi
}

check_upgrade_case
```

Run the precheck_migration.sh to determine whether you can run an automatic migration of the common core services or whether you need to configure common core services to run a semi-automatic migration:

```
./precheck_migration.sh
```

Take the appropriate action based on the message returned by the script:

Option 1 - "The system is ready for migration" -> Upgrade your cluster as usual

Option 2 - "Run the following command to increase the CPU and memory" -> Run the patch command returned by the script, upgrade IBM Software Hub

Option 3 - "The script returns both of the following messages: Run the following command to increase the CPU and memory AND Before you can start the upgrade, you must prepare the system for migration." -> Run the patch command returned by the script, then run the following command to enable semi-automatic migration:

```
oc patch ccs ccs-cr \
-n ${PROJECT_CPD_INST_OPERANDS} \
--type merge \
--patch '{"spec": {"use_semi_auto_catalog_api_migration": true}}'
```

Proceed with the upgrade of IBM Software Hub

***Important: After you upgrade the services in your environment, ensure that you complete Completing the [catalog-api service migration](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=services-completing-catalog-api-migration)***

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

Retrieve the common-service-db-superuser password

```
oc get secret common-service-db-superuser -o jsonpath='{.data.password}' | base64 -d
```

For example

```
8ANCuxIfpXvlOwPnaySmkesrxtiit4GrYtAFxameJB2qH6ILOxk39G5T1UF65xze
```

Connect to common-storage-db and when prompted for the password use the password you retrieved in the previous step

```
oc exec -it common-service-db-1 -- /bin/bash
```

```
psql -h common-service-db-rw -p 5432 -U postgres -d im
```

Enter the password and then run the following command

```
set search_path = platformdb; SELECT *
FROM information_schema.table_constraints WHERE constraint_name = 'fk_useratt_fk' AND table_schema = 'platformdb' AND table_name = 'users_attributes';
```

If it returns no lines, execute the following command

```
ALTER TABLE platformdb.users_attributes ADD CONSTRAINT fk_useratt_fk FOREIGN KEY (user_uid) REFERENCES platformdb.users (uid) ON DELETE CASCADE; quit exit
```

Otherwise, use `quit` followed by `exit`

### Upgrade the required operators and custom resources for the instance (est. 60 minutes):

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
oc get ZenService lite-cr -n cp4data -o yaml
```

```
oc get ibmcpd ibmcpd-cr -n cp4data -o yaml
```

**Pay close attention to the Zen operator pod logs and the lite-cr and ibmcpd-cr yaml for any errors**

In TS020176018, the setup-instance command failed and the following action needed to be taken

Retrieve the common-service-db superuser password

```
oc get secret common-service-db-superuser -o jsonpath='{.data.password}' | base64 -d
```

For example

```
8ANCuxIfpXvlOwPnaySmkesrxtiit4GrYtAFxameJB2qH6ILOxk39G5T1UF65xze
```

Take note of the primary common-service-db pod name

```
oc get cluster.postgresql.k8s.enterprisedb.io -n cp4data
```

For example

```
common-service-db
```

Remote into the common-service-db pod

```
oc exec -it common-service-db-3 -- /bin/bash
```

```
psql -h common-service-db-rw -p 5432 -U postgres -d im
```

Enter the password you obtained from the secret

Then, run the following command

```
set search_path = platformdb; SELECT * FROM information_schema.table_constraints WHERE constraint_name = 'fk_useratt_fk' AND table_schema = 'platformdb' AND table_name = 'users_attributes';
```

If the previous command returns no rows, run the following command

```
ALTER TABLE platformdb.users_attributes ADD CONSTRAINT fk_useratt_fk FOREIGN KEY (user_uid) REFERENCES platformdb.users (uid) ON DELETE CASCADE; quit exit
```


### Upgrade the operators in the operators project (est. 15 minutes):

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


## 8. Upgrade CPD services (db2oltp,datagate,dmc)

In order to have more visibility into each service upgrade, it is recommended to upgrade the services in sequential order, as follows:

### Upgrade Db2 custom resource (est. 10 minutes)

**Important: Before you proceed, Stop Data Gate synchronization in the UI**

Before you upgrade the Db2 custom resource, check the status of the zen-metastore-edb pods

```
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -l k8s.enterprisedb.io/cluster=zen-metastore-edb
```

For example

```
NAME                  READY   STATUS    RESTARTS      AGE
zen-metastore-edb-1   1/1     Running   0             XdXh
zen-metastore-edb-2   1/1     Running   0             XdXh
```

If there are no issues with zen-metastore-edb pods, proceed with upgrading the Db2 custom resource

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
--upgrade=true \
--case_download=false
```

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=db2oltp
```

You can also monitor the status of the Db2 upgrade with the following command:

```
oc get db2oltpservice.databases.cpd.ibm.com -n cp4data -o yaml
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
    commands](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cli-creating-cpd-profile)
	

**Potential issue during the upgrade of Db2 service instance related to tempspace1**

During the Db2 instance upgrade you may observe any issue with Tempspace1, "Table space access is not allowed." 

This problem is related to the usage of local storage and results in missing files and missing folder structures within the Db2u pod. 

To address this, you will need to create a specific directory structure inside the Db2u pod and copy the container tag to this location. 

RSH into the Db2u pod and run the following commands to create the folder structure and copy the container tag to the same location. 

Please keep in mind that the container name will change.

Start by transfer the SQLTAG.NAM file into the Db2 pod

```
oc cp SQLTAG.NAM c-db2oltp-1736767772824325-db2u-0:/tmp
```

Recreate the directory (Note: the SQLTAG.NAM file is located in /apps/bastion/install_files of bastion DEV ME)

```
cd /tmp
chmod 777 SQLTAG.NAM
su - db2inst1
mkdir -p /mnt/tempts/systemp/db2inst1/NODE0000/BCEFDB/T0000001/C0000000.TMP
```

Copy the container tag and update permissions

```
cd /tmp
cp SQLTAG.NAM /mnt/tempts/systemp/db2inst1/NODE0000/BCEFDB/T0000001/C0000000.TMP
chmod 600 /mnt/tempts/systemp/db2inst1/NODE0000/BCEFDB/T0000001/C0000000.TMP/SQLTAG.NAM
chown db2inst1:db2iadm1 /mnt/tempts/systemp/db2inst1/NODE0000/BCEFDB/T0000001/C0000000.TMP/SQLTAG.NAM
```

In the Db2 terminal run the following command

```
CREATE TEMPORARY TABLESPACE "TEMPSPACE2" IN DATABASE PARTITION GROUP IBMTEMPGROUP PAGESIZE 16384 MANAGED BY AUTOMATIC STORAGE USING STOGROUP "IBMDB2UTEMPSG1" EXTENTSIZE 4 PREFETCHSIZE AUTOMATIC BUFFERPOOL "IBMDEFAULTBP" OVERHEAD INHERIT TRANSFERRATE INHERIT FILE SYSTEM CACHING DROPPED TABLE RECOVERY OFF

drop tablespace TEMPSPACE1

quit
```


After these steps, confirm that Db2 can start and then once you can connect to Db2, run the below commands as Db2 instance owner to take the backup of the container tags, as this backup will be used to restore the container tags when the pod starts again:

```
sudo rsync -rdgop --numeric-ids --checksum --exclude '*TLB' --exclude '*TDA' --exclude '*TBA' /mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/ /mnt/blumeta0/local-backup\
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

**Before you upgrade Db2 instance, remove any taints in the Db2 node**

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

Continue with the upgrade of Data Gate custom resource

### Upgrade the Data Gate custom resource (est. 10 minutes):

Complete the following tasks before you run the actual Data Gate upgrade:

1. [Activating the Db2 Connect Unlimited Edition license](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrade-activating-db2-connect-unlimited)

As a prerequisite for all upgrades, the license for your JDBC driver must be activated


2. [Cleaning up the data-gate-api container](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrade-cleaning-up-data-gate-api)

Open a shell in the data-gate-api container of the Data Gate pod:

```
oc exec -it -c data-gate-api ${DG_POD_ID} -- bash
```

Determine the ID of the Data Gate instance you want to upgrade by running the oc get dginstance command, as in this example:

```
oc get dginstance -n ${PROJECT_CPD_INST_OPERANDS}
```

For example:

```
NAME                 VERSION   BUILD      STATUS      RECONCILED   AGE
dg1699914520773847   5.0.3     5.0.3      Completed   5.0.3        6h7m

The instance ID in this example is this example is dg1699914520773847
```

Determine the ID of the Data Gate instance's server pod by running the following command:

```
DG_POD_ID=$(oc get pod -l icpdsupport/app=dg-instance-server,\
icpdsupport/serviceInstanceId=`echo ${DG_INSTANCE_ID} | 
sed 's/^dg//'` -o jsonpath='{.items[0].metadata.name}')
```

DG_INSTANCE_ID is the instance ID you identified in step 1

Run the following command:

```
rm -rf /head/clone-api/work/jetty-0_0_0_0-8188-clone-api_war-_clone_system-any-/webapp/* 2> /dev/null
```

Exit the container shell and proceed with the upgrade

Upgrade the custom resource for Data Gate:

```
cpd-cli manage apply-cr \
--components=datagate \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true \
--case_download=false
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

**Potential issue after the upgrade of Data Gate service instances**

After Data Gate instances have been upgraded, it is possible that the configuration settings of the Db2 target database are not correctly migrated during the upgrade

Missing Db2 configuration settings can cause a variety of issues, in which the 'DB2COMM=TCPIP,SSL' was missing

To resolve this, we had to add the missing Db2 setting by using 'db2set DB2COMM=TCPIP,SSL'

These configurations should persist through a Db2u pod recycle

Keep track of your Db2 configuration settings prior to upgrading to ensure you have a list of settings which you can restore

For example

```
oc rsh c-db2oltp-1764799139398922-db2u-0 su - db2inst1
```

```
db2set
```

```
DB2_WAITFORDATA_LIBNAME=/mnt/blumeta0/home/db2inst1/config_db2u/waitfordata/libs/libwaitForDataSharedLib.so
DB2_ENABLE_MULTIPLE_CCSIDS_FOR_LOB_COLUMNS=YES [DB2_WORKLOAD]
DB2_REMOTE_EXTTAB_PIPE_PATH=/db2u/tmp
DB2_ENABLE_LOCAL_DATE_FORMAT=YES [DB2_WORKLOAD]
DB2_4K_DEVICE_SUPPORT=ON
DB2_ENABLE_CCSID_SEMANTICS=YES [DB2_WORKLOAD]
DB2_ENABLE_MULTIPLE_CCSIDS=YES [DB2_WORKLOAD]
DB2_FMP_RUN_AS_CONNECTED_USER=NO
DB2_STATISTICS=USCC:0;DISCOVER:ON;CGS_SAMPLE_RATE_ADJUST:0;RAND_COLGROUPID:Y;LEN21_COLGROUPNAME:Y;AUTO_SAMPLING_IMPRV:ON
DB2_OVERRIDE_THREADING_DEGREE=2
DB2_OVERRIDE_NUM_CPUS=3
DB2_RESTORE_GRANT_ADMIN_AUTHORITIES=ON
DB2_ATS_ENABLE=YES
DB2_WORKLOAD=ANALYTICS_ACCELERATOR
DB2RSHCMD=/bin/ssh
DB2AUTH=OSAUTHDB,ALLOW_LOCAL_FALLBACK,PLUGIN_AUTO_RELOAD
DB2_USE_ALTERNATE_PAGE_CLEANING=ON [DB2_WORKLOAD]
DB2_APPENDERS_PER_PAGE=1
DB2_LOAD_COPY_NO_OVERRIDE=COPY YES TO /mnt/bludata0/db2/copy
DB2_SELECTIVITY=ALL,AJSEL,UNIQUE_COL_FF ON
DB2_ANTIJOIN=EXTEND [DB2_WORKLOAD]
DB2MAXFSCRSEARCH=1
DB2CHECKCLIENTINTERVAL=100
DB2LIBPATH=/mnt/blumeta0/home/db2inst1/config_db2u/waitfordata/libs
DB2LOCK_TO_RB=STATEMENT
DB2COMM=TCPIP,SSL
```

You can also [change configuration settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings) after you deploy your instance

Before you proceed, make sure the CPD profile is set up:

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


### Upgrade the Db2 Data Management Console custom resource (est. 10 minutes):

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
--upgrade=true \
--case_download=false
```

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=dmc
```

You can also monitor the status of the Db2 upgrade with the following command:

```
oc get db2oltpservice.databases.cpd.ibm.com -n cp4data -o yaml
```

DMC is upgraded when the apply-cr command returns:

```
[SUCCESS]... The apply-cr command ran successfully
```

### Complete the catalog-api service migration:

After you upgrade the common core services to IBM® Software Hub Version 5.2, the back-end database for the catalog-api service is migrated from CouchDB to PostgreSQL

1. Checking the migration method used

If you ran an automatic migration, the common core services waits for the migration jobs to complete before upgrading the components associated with the common core services

If you ran a semi-automatic migration, the common core services runs the migration jobs while upgrading the components associated with the common core services

Run the following command to determine which migration method was used:

```
oc describe ccs ccs-cr \
--namespace ${PROJECT_CPD_INST_OPERANDS} \
| grep use_semi_auto_catalog_api_migration
```

Take the appropriate action based on the response returned by the oc describe command:

The command returns an empty response:
```
Automatic -> Proceed to 4. Collecting statistics about the migration
```

The command returns true:
```
Semi-automatic	-> Proceed to 2. Checking the status of the migration jobs.
```


2. Checking the status of the migration jobs

This step is required only for semi-automatic migrations. If you completed an automatic migration, proceed to 4. Collecting statistics about the migration

Check the migration status periodically. The following jobs might take some time to complete depending on number of assets to be migrated:
-cams-postgres-migration-job
-jobs-postgres-upgrade-migration

To check the status of the jobs:

```
oc get job cams-postgres-migration-job jobs-postgres-upgrade-migration \
--namespace ${PROJECT_CPD_INST_OPERANDS} \
-o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[0].type,COMPLETIONS:.status.succeeded
```

The command returns output with the following format:

```
NAME                             STATUS    COMPLETIONS
cams-postgres-migration-job      Complete   1/1       
jobs-postgres-upgrade-migration  Complete   1/1      
```

Take the appropriate action based on the status of the jobs:

Option 1 - The status of either job is Failed -> Contact IBM Support for assistance resolving the error

Option 2 - The status of either job is InProgress -> Wait several minutes before checking the status of the jobs again

Option 3 - The status of both jobs is Complete -> Proceed to 3. Completing the migration


3. Completing the migration

This step is required only for semi-automatic migrations. If you completed an automatic migration, proceed to 4. Collecting statistics about the migration.

After both of the following jobs complete, you can complete the migration to PostgreSQL:
-cams-postgres-migration-job
-jobs-postgres-upgrade-migration

To complete the migration to PostgreSQL:

Run the following command to continue the semi-automatic migration:

```
oc patch ccs ccs-cr \
--namespace ${PROJECT_CPD_INST_OPERANDS} \
--type merge \
--patch '{"spec": {"continue_semi_auto_catalog_api_migration": true}}'
```

Wait for common core services custom resource to be Completed. This process takes at least 10 minutes. However, it might be significantly longer if any assets were changed during the common core services upgrade.

To check the status of the custom resource, run:

```
oc get ccs ccs-cr \
--namespace ${PROJECT_CPD_INST_OPERANDS}
```

The command returns output with the following format:

```
NAME     VERSION   RECONCILED   STATUS      PERCENT   AGE
ccs-cr   11.0.0    11.0.0       Completed   100%      1d
```

Take the appropriate action based on the status of the jobs:

Option 1 - The status of either job is Failed -> Contact IBM Support for assistance resolving the error

Option 2 - The status of either job is InProgress -> WWait several minutes before checking the status of the custom resource again

Option 3 - The status of both jobs is Complete -> 	Proceed to 4. Collecting statistics about the migration


4. Collecting statistics about the migration

Save the following script on the client workstation as a file named migration_status.sh:

```
#!/bin/bash

# Set postgres connection parameters
postgres_password=$(oc get secret -n ${PROJECT_CPD_INST_OPERANDS} ccs-cams-postgres-app -o json 2>/dev/null | jq -r '.data."password"' | base64 -d)
postgres_username=cams_user
postgres_db=camsdb
postgres_migrationdb=camsdb_migration

echo -e "======MIGRATION STATUS==========="

# Total migrated database(s)
databases=$(oc -n ${PROJECT_CPD_INST_OPERANDS} -c postgres exec ccs-cams-postgres-1 -- psql -t postgresql://$postgres_username:$postgres_password@localhost:5432/$postgres_migrationdb -c "select count(*) from migration.status where state='complete'" 2>/dev/null)
if [ -n "$databases" ];then
  databases_no_space=$(echo "$databases" | tr -d ' ')
  echo "Total catalog-api databases migrated: $databases_no_space"
else
  echo "Unable to fetch migration information for databases"
fi

# Total migrated assets
assets=$(oc -n ${PROJECT_CPD_INST_OPERANDS} -c postgres exec ccs-cams-postgres-1 -- psql -t postgresql://$postgres_username:$postgres_password@localhost:5432/$postgres_db -c "select count(*) from cams.asset" 2>/dev/null)
if [ -n "$assets" ];then
  assets_no_space=$(echo "$assets" | tr -d ' ')
  echo -e "Total catalog-api assets migrated: $assets_no_space\n"
else
  echo "Unable to fetch migration information for assets"
fi
```

Run the migration_status.sh script:

```
./migration_status.sh
```

Proceed to step 5. Backing up the PostgreSQL database


5. Backing up the PostgreSQL database

Back up the new PostgreSQL database

Save the following script on the client workstation as a file named backup_postgres.sh:

```
#!/bin/bash

# Make sure PROJECT_CPD_INST_OPERANDS is set
if [ -z "$PROJECT_CPD_INST_OPERANDS" ]; then
  echo "Environment variable PROJECT_CPD_INST_OPERANDS is not defined. This environment variable must be set to the project where IBM Software Hub is running."
  exit 1
fi

echo "PROJECT_CPD_INST_OPERANDS namespace is: $PROJECT_CPD_INST_OPERANDS"

# Step 1: Find the replica pod
REPLICA_POD=$(oc get pods -n $PROJECT_CPD_INST_OPERANDS -l app=ccs-cams-postgres -o jsonpath='{range .items[?(@.metadata.labels.role=="replica")]}{.metadata.name}{"\n"}{end}')

if [ -z "$REPLICA_POD" ]; then
  echo "No replica pod found."
  exit 1
fi

echo "Replica pod: $REPLICA_POD"

# Step 2: Extract JDBC URI from a secret
JDBC_URI=$(oc get secret ccs-cams-postgres-app -n $PROJECT_CPD_INST_OPERANDS -o jsonpath="{.data.uri}" | base64 -d)

if [ -z "$JDBC_URI" ]; then
  echo "JDBC URI not found in secret."
  exit 1
fi

#  Set path on the pod to save the dump file 
TARGET_PATH="/var/lib/postgresql/data/forpgdump"

# Step 3: Run pg_dump with nohup inside the pod
oc exec "$REPLICA_POD" -n $PROJECT_CPD_INST_OPERANDS -- bash -c "
  TARGET_PATH=\"$TARGET_PATH\"
  JDBC_URI=\"$JDBC_URI\"
  echo \"TARGET_PATH is $TARGET_PATH\"
  mkdir -p $TARGET_PATH &&
  chmod 777 $TARGET_PATH &&
  nohup bash -c '
    pg_dump $JDBC_URI -Fc -f $TARGET_PATH/cams_backup.dump > $TARGET_PATH/pgdump.log 2>&1 &&
    echo \"Backup succeeded. Please copy $TARGET_PATH/cams_backup.dump file from this pod to a safe place and delete it on this pod to save space.\" >> $TARGET_PATH/pgdump.log
  ' &
  echo \"pg_dump started in background. Logs: $TARGET_PATH/pgdump.log\"
"
```

Run the backup_postgres.sh script:

```
./backup_postgres.sh
```

The script starts the backup in a separate terminal session

Set the REPLICA_POD environment variable:

```
REPLICA_POD=$(oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -l app=ccs-cams-postgres -o jsonpath='{range .items[?(@.metadata.labels.role=="replica")]}{.metadata.name}{"\n"}{end}')
```

Open a remote shell in the replica pod:

```
oc rsh ${REPLICA_POD}
```

Change to the /var/lib/postgresql/data/forpgdump/ directory:

```
cd /var/lib/postgresql/data/forpgdump/
```

Run the following command to monitor the list of files in the directory:

```
ls -lat
```

Wait for the backup to complete. (This process can take several hours if the database is large.)

Option 1 - In progress	-> During the backup, the size of the pgdump.log file increases.

Option 2 - Complete	-> The backup is complete when the script writes the following message to the pgdump.log file: Backup succeeded. Please copy /var/lib/postgresql/data/forpgdump/cams_backup.dump file from this pod to a safe place and delete it on this pod to save space.

Option 3 - Failed -> If the backup fails, the pgdump.log file will include error messages. If the backup fails, contact IBM Support. Append the pgdump.log file to your support case.

***Do not proceed to the next step unless the backup is complete***

Set the POSTGRES_BACKUP_STORAGE_LOCATION environment variable to the location where you want to store the backup:

```
export POSTGRES_BACKUP_STORAGE_LOCATION=<directory>
```

***Important: Ensure that you choose a location where the file will not be accidentally deleted***

Copy the backup to the POSTGRES_BACKUP_STORAGE_LOCATION:

```
oc cp ${REPLICA_POD}:/var/lib/postgresql/data/forpgdump/cams_backup.dump \
$POSTGRES_BACKUP_STORAGE_LOCATION/cams_backup.dump
```

Delete the backup from the replica pod:

```
oc rsh $REPLICA_POD rm -f /var/lib/postgresql/data/forpgdump/cams_backup.dump
```

Proceed to step 6. Consolidating the PostgreSQL database


6. Consolidating the PostgreSQL database

After you back up the new PostgreSQL database, you must consolidate all of the existing copies of identical data across governed catalogs into a single record so that all identical data assets share a set of common properties

Set the INSTANCE_URL environment variable to the URL of IBM Software Hub:

```
export INSTANCE_URL=https://<URL>
```

Tip: To get the URL of the web client, run the following command:

```
cpd-cli manage get-cpd-instance-details \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Get the name of a catalog-api-jobs pod:

```
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} \
| grep catalog-api-jobs
```

Set the CAT_API_JOBS_POD environment variable to the name a pod returned by the preceding command:

```
export CAT_API_JOBS_POD=<pod-name>
```

Open a Bash prompt in the pod:

```
oc exec ${CAT_API_JOBS_POD} -n ${PROJECT_CPD_INST_OPERANDS} -it -- bash 
```


Run the following command to set the AUTH_TOKEN environment variable:

```
AUTH_TOKEN=$(cat /etc/.secrets/wkc/service_id_credential)
```

Start the consolidation:

```
curl -k -X PUT "${INSTANCE_URL}/v2/shared_assets/initialize_content?bss_account_id=999" \
     -H "Authorization: Basic $AUTH_TOKEN"
```

The command returns a transaction ID

***Important: Save the transaction ID so that you can refer to it later for debugging, if needed***

Get the name of a catalog-api pod:

```
oc get pods \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep catalog-api \
| grep -v catalog-api-jobs
```

Set the CAT_API_POD environment variable to the name a pod returned by the preceding command:

```
export CAT_API_POD=<pod-name>
```

Check the catalog-api pod logs to determine the status of the consolidation

Check for the following success message:

```
oc logs ${CAT_API_POD} \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep "Initial consolidation with bss account 999 complete"
```

If the command returns a response the consolidation was successful, proceed to [What to do if the consolidation completed successfully](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=services-completing-catalog-api-migration#catalog-api-migration__success)

If the command returns an empty response, proceed to the next step

Check for the following failure message:

```
oc logs ${CAT_API_POD} \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep "Error running initial consolidation with resource key"
```

If the command returns a response the consolidation failed, try to consolidate the database again. If the problem persists, contact [IBM Support](https://www.ibm.com/mysupport/s/topic/0TO50000000IYkUGAW/cloud-pak-for-data?language=en_US)

If the command returns an empty response, proceed to the next step

Check for the following failure message:

```
oc logs ${CAT_API_POD} \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep "Error executing initial consolidation for bss 999"
```

If the command returns a response the consolidation failed, try to consolidate the database again. If the problem persists, contact [IBM Support](https://www.ibm.com/mysupport/s/topic/0TO50000000IYkUGAW/cloud-pak-for-data?language=en_US)

If the command returns an empty response, proceed to the next step

If the preceding commands returned empty responses, wait 10 minutes before checking the pod logs again


7. What to do if the consolidation completed successfully

If the PostgreSQL database consolidation was successful, wait several weeks to confirm that the projects, catalogs, and spaces in your environment are working as expected.

After you confirm that the projects, catalogs, and spaces are working as expected, run the following commands to clean up the migration resources:

Delete the pods associated with the migration:

```
oc delete pod \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the jobs associated with the migration:

```
oc delete job \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the config maps associated with the migration:

```
oc delete cm \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the secrets associated with the migration:

```
oc delete secret \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the persistent volume claim associated with the migration:

```
oc delete pvc cams-postgres-migration-pvc \
-n ${PROJECT_CPD_INST_OPERANDS}
```

***This marks the completion of the catalog-api service migration***


## 9. All Potential Issues

1. Setup-instance command will fail with 'imagepullbackoff' errors if the storage test images are missing. Mirror the storage test images ahead of time or exclude the '--run_storage_tests' flag

2. For imagepullbackoff errors, ensure that all required images are mirrored to the private registry, for example:

```
skopeo copy docker://icr.io/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d docker://lpdza532.phx.aexp.com:5000/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d \--all \--remove-signatures \--authfile=/var/registry/oc4.7/installer/pullsecret/merged_pullsecret.json
```

```
skopeo copy docker://icr.io/cpopen/cpd/olm-utils-v3@sha256:5a5b9b563756b3a41b22b2c9b6e940d3ceed6e12100e351809eb7f09ed058905 docker://lppza417.gso.aexp.com:5000/cpopen/cpd/olm-utils-v3@sha256:5a5b9b563756b3a41b22b2c9b6e940d3ceed6e12100e351809eb7f09ed058905 --all --remove-signatures --authfile=/root/merged_pullsecret.json
```

3. For insufficient CPU, insufficient memory, untolerated taint issues, ensure there are enough resources available on the cluster. Perform a pre-upgrade health check to ensure the cluster is in a healthy state prior to the upgrade

4. If the Db2 license is not upgraded to the compatible version you may run into issues during Db2 instance upgrade. Follow the steps to "[Upgrade the license before you deploy Db2](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=setup-upgrading-license-before-you-deploy-db2)"

5. During the Db2 instance upgrade, the Db2ckupgrade.sh utility runs a job that must be scheduled to the same node as the Db2 engine pod. To resolve this issue, restrict the job to run on the same node as the engine pod. Use a cordon, to ensure that this job is scheduled to the node hosting Db2. 

6. During the Db2 instance upgrade you may observe issues related to Tempspace1, "Table space access is not allowed." This problem is related to the usage of local storage and results in missing files and missing folder structures within the Db2 pod. To address this, you will need to create a specific directory structure inside the Db2u pod and copy the container tag to this location.

7. During Data Gate instance upgrade a dg-1750101262664465-backup-head-job pod runs, but it should be scheduled to any node where Db2 is not running. For this, cordon the node hosting Db2, and after the backup-head-job pod is completed, uncordon the same node

8. After Data Gate instances have been upgraded, it is possible that the configuration settings of the Db2 target database are not correctly migrated during the upgrade. Missing Db2 configuration settings can cause a variety of issues, in which the DB2COMM=TCPIP,SSL was missing. To resolve this, we had to add the missing Db2 setting by using 'db2set DB2COMM=TCPIP,SSL'. These configurations should persist through a Db2u pod recycle. Keep track of your Db2 configuration settings prior to upgrading to ensure you have a list of settings which you can restore. You can also [change configuration settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings) after you deploy your instance

This marks the end of the installation of IBM Software Hub, Db2oltp, Datagate, and Db2 Data Management Console

## 10. Validate CPD upgrade (customer acceptance test)

End of document
