# CP4DCLONE IBM ROKS
Cloning from the Source OCP/CP4D cluster and Reinstate to new OCP cluster has the following steps.

## Pre-requisites for Cloning
Pre-requisites before running cloner utility.

1.	Details of the S3 / ICOS where AIP user can create a bucket and save the clones to
2.	OCP 4.5.6 or higher cluster with OCP/CP4D/DV/DMC services installed.
3.	Run the oc login for the existing OCP/CP4D cluster.
4.	From the oc login you will get the SERVER and TOKEN parameters we need.
5.	Install the docker on the machine host from which you are going to run the commands. This can be a bastion node in the AIP AWS environment or any node from which both the source and target OCP/CP4D clusters are accessible.

## Download the docker image
Always use the latest script docker images as they have latest scripts.

docker pull quay.io/drangar_us/cpc:cp4d.3.5
docker pull quay.io/drangar_us/sincr:v2

If in doubt you can delete and pull again
docker image -a

Remove the old image and pull again.
docker rm <container id>
docker rm <container id> --force (If needed)

## CLONE

Cloning from Existing cluster

To run the cloner, you will use the following syntax

docker run -e ACTION=CLONE -e SERVER=<OC LOGIN SERVER> -e TOKEN=<OC LOGIN TOKEN> -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=<OC PROJECT NAME> -e COSBUCKET=<S3 BUCKET NAME> -e COSAPIKEY=<S3 ACCESS KEY> -e COSSECRETKEY=<S3 SECRET KEY> -e COSREGION=<S3 REGION> -e COSENDPOINT=<S3 END POINT> -it quay.io/drangar_us/cpc:cp4d.3.5 

E.G. Command
docker run -e ACTION=CLONE -e INTERNALREGISTRY=1 -e SERVER=https://c115-e.us-south.containers.cloud.ibm.com:30717 -e TOKEN=P_AsqemimU55zUtQ-ZlgmIJQYVyON9WzcTyirxTEm-w -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=cpd30 -e COSBUCKET=aipclonebackuppravin -e COSAPIKEY=b737054a3ed54cf0812ddbd56c5efe8d -e COSSECRETKEY=55a9a6c6d7fc9e70c4038c54818b88b3f3fed52874cd47e9 -e COSREGION=au-syd -e COSENDPOINT=https://s3.au-syd.cloud-object-storage.appdomain.cloud -it quay.io/drangar_us/cpc:cp4d.3.5

### To check the backups on the S3 /ICOS storage
aws --endpoint-url=https://s3.au-syd.cloud-object-storage.appdomain.cloud s3 ls s3://aipclonebackuppravin/
aws --endpoint-url=https://s3.au-syd.cloud-object-storage.appdomain.cloud s3 ls –summarize s3://aipclonebackuppravin/

## REINSTATE

### Pre-requisites for Reinstate.

Pre-requisites before running the reinstate utility.
1.	Ensure you are running this from the same bastion node where you ran the clone command earlier and it has access to the new cluster.
2.	Ensure you are either using the Internal and External OpenShift registry and they are matching the exact version of the images between the Source and the target cluster/
3.	If using external registry, the target OCP cluster would already have been configured to use the external OpenShift registry.
4.	If using internal registry then you will need to follow the procedure below
5.	You first need to pull/download the images matching with the source cluster images to bastion node.
6.	Then load these images to the targe cluster from bastion node.
7.	Download the matching version of cpd-cli to this bastion node from the following link if not already done https://github.com/IBM/cpd-cli/releases
If using External Registry (preferred / recommended approach)

In this approach the target cluster already has access to registry to download the images directly from this internet registry and hence no need to download them to bastion node.
If using internal Registry to pre-load images

### PRELOAD Images (Optional - Now merged with REINSTATE)

Example steps to preload images to target cluster.
1.	aws --endpoint-url=https://s3.au-syd.cloud-object-storage.appdomain.cloud s3 ls s3://aipclonebackuppravin/
2.	aws --endpoint-url=https://s3.au-syd.cloud-object-storage.appdomain.cloud s3 sync s3://aipclonebackuppravin/specs specs
3.	You should see the specs folder on bastion node
4.	oc create -f 0-projects.json to create the same project as in source cluster in the target cluster. You get the 0-projects.json file from backup.
5.	Use the correct cpd-cli and cpd-dv, dmc repo.yaml file with the correct keys.

bash-3.2$ cat repo.yam
---
fileservers:
  -
    url: "https://raw.github.com/IBM/cloud-pak/master/repo/cpd/3.5"
registry:
  -
    url: cp.icr.io/cp/cpd
    name: base-registry
    namespace: ""
    username: cp
    apikey: eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE2MTM1NjQxMTUsImp0aSI6ImFhYmI4NDNhYmE3NDQ4NGZhNzhiYzRlODJjOWRmMjQzIn0.2KVoSRLrAY522wXJiH0fKIVXohV-ZxECXDHhry9IuIo

6.	Preload the images using the below example script that can be enhanced further.

bash-3.2$ cat preloadimages.sh
---
  !#/bin/ksh
    -
    LITEREPO="/Users/pravinkedia/Downloads/AIP/clone/cpd35/repo.yaml"
    DVREPO="/Users/pravinkedia/Downloads/AIP/clone/cpd35/repo.yaml"
    DMCREPO="/Users/pravinkedia/Downloads/AIP/clone/cpd35/repo.yaml"
    CPDCLI="/Users/pravinkedia/Downloads/AIP/clone/cpd35/cpd-cli"
    NAMESPACE="cpd30"
    REGISTRY=$(oc get route -n openshift-image-registry | grep image-registry | awk '{print $2}')
    IMAGEDIR="/Users/pravinkedia/Downloads/AIP/clone/cpd35/cpd-cli-workspace"

    cd /Users/pravinkedia/Downloads/AIP/clone/cpd35/
    echo "Starting to Download the file (may take long time) Step 0"

    echo "Staring to Download CPD-LITE - Step 1"
    read a
    ${CPDCLI} preload-images --action download -a lite --repo ${LITEREPO} --accept-all-licenses
    echo "Starting to preload CPD-LITE - Step 2"
    read a
    ${CPDCLI} preload-images --assembly lite --action push --load-from ${IMAGEDIR} --transfer-image-to ${REGISTRY}/${NAMESPACE} --target-registry-username $(oc whoami)  --target-registry-password $(oc whoami -t) --accept-all-licenses --insecure-skip-tls-verify

    echo "Staring to Download DV - Step 3"
    read a
    ${CPDCLI} preload-images --action download -a dv --repo ${DVREPO} --accept-all-licenses

    echo "Starting to preload DV - Step 4"
    read a
    ${CPDCLI} preload-images --assembly dv --action push --load-from ${IMAGEDIR} --transfer-image-to ${REGISTRY}/${NAMESPACE} --target-registry-username $(oc whoami) --target-registry-password $(oc whoami -t) --accept-all-licenses --insecure-skip-tls-verify
    echo "Staring to Download DMC - Step 5"
    read a
    ${CPDCLI} preload-images --action download -a dmc --repo ${DVREPO} --accept-all-licenses

    echo "Starting to preload DMC- Step 6"
    read a
    ${CPDCLI} preload-images --assembly dmc --action push --load-from ${IMAGEDIR} --transfer-image-to ${REGISTRY}/${NAMESPACE} --target-registry-username $(oc whoami) --target-registry-password $(oc whoami -t) --accept-all-licenses --insecure-skip-tls-verify

    echo "Complete. All done"

## Reinstate for new OpenShift cluster with OCP (preferred)

1.	If the new cluster has only OCP installed, then the reinstate command with load the images from either internal or external registry.
2.	Then we will issue the re-instate command in the new cluster by provide the new cluster details
    docker run -e ACTION=REINSTATE -e INTERNALREGISTRY=1 -e SERVER=<OC LOGIN SERVER> -e TOKEN=<OC LOGIN TOKEN> -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=<OC PROJECT NAME> -e COSBUCKET=<S3 BUCKET NAME> -e COSAPIKEY=<S3 ACCESS KEY> -e COSSECRETKEY=<S3 SECRET KEY> -e COSREGION=<S3 REGION> -e COSENDPOINT=<S3 END POINT> -it quay.io/drangar_us/cpc:cp4d.3.5
    E.G. Command
    docker run -e ACTION=REINSTATE -e INTERNALREGISTRY=1 -e SERVER=https://c100-e.us-east.containers.cloud.ibm.com:32279 -e TOKEN=CS44Pxl_iYg2yWjMKLWBRH9D_kKiDcH8atnt7w0kdJg -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=cpd30 -e COSBUCKET=aipclonebackuppravin -e COSAPIKEY=b737054a3ed54cf0812ddbd56c5efe8d -e COSSECRETKEY=55a9a6c6d7fc9e70c4038c54818b88b3f3fed52874cd47e9 -e COSREGION=au-syd -e COSENDPOINT=https://s3.au-syd.cloud-object-storage.appdomain.cloud -it quay.io/drangar_us/cpc:cp4d.3.5
3.	Validate target cluster health.
    oc exec -i dv-engine-0 -n <NAMESPCE> -c dv-engine -- bash -c "/opt/dv/current/liveness.sh --verbose"
5.	Validate queries against DV tables and caches from source clusters work without issues.
6.	There are 2 routes returned from:  oc get routes -n zzz
7.	Use the cp4d-route
NAME         HOST/PORT                                             PATH   SERVICES        PORT                   TERMINATION            WILDCARD
cp4d-route   cp4d-route-zzz.apps.ocp452-px255-fips07.cp.fyre.ibm.com   ibm-nginx-svc   ibm-nginx-https-port   passthrough            
zzz-cpd         zzz-cpd-zzz.apps.ocp452-px255-fips07.cp.fyre.ibm.com         ibm-nginx-svc   ibm-nginx-https-port   passthrough/Redirect

## Reinstate for new OpenShift cluster with OCP and CP4D pre-installed.

1.	If you already have the OCP cluster with CP4D pre-installed, then you could choose to reinstate the cluster in the new namespace.
2.	Or you can drop the existing namespace and reinstate the cluster into the same namespace. You may need to wait for 10 minutes for this drop to happen correctly.
3.	Follow the same command as above.
