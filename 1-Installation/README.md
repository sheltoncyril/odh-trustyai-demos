# Installing ODH/RHOAI and TrustyAI
This guide will walk through installing Open Data Hub/Red Hat Openshift AI and TrustyAI into your cluster. Starting from a completely
blank cluster, you will be left with:
1) An Open Data Hub/Red Hat Openshift AI installation
2) A namespace to deploy models into
3) A TrustyAI Operator, to manage all instances of the TrustyAI Service
4) A TrustyAI Service, to monitor and analyze all the models deployed into your model namespace.

## Cluster Setup
1) Make sure you are `oc login`'d to your OpenShift cluster
2) Create two projects, `opendatahub` and `model-namespace`. These names are arbitrary, but I'll be using them throughout the rest of this demo:
   1) `oc new-project opendatahub`
   2) `oc new-project model-namespace`
  
## Enable User-Workload-Monitoring
To get enable ODH's monitoring stack , user-workload-monitoring must be configured:
1) Enable user-workload-monitoring: `oc apply -f resources/enable_uwm.yaml`
2) Configure user-workload-monitoring to hold metric data for 15 days: `oc apply -f resources/uwm_configmap.yaml`

Depending on how your cluster was created, you may need to enable a User Workload Monitoring setting from 
your cluster management UI (for example, on console.redhat.com)

## ODH/RHOAI Prerequisties
1) Install the Red Hat OpenShift Serverless operator.
2) Install the Red Hat OpenShift Service Mesh operator.

## Install ODH/RHOAI Operator
1) From the OpenShift Console, navigate to "Operators" -> "OperatorHub", and search for "Open Data Hub" or "Red Hat Openshift AI"
   ![ODH in OperatorHub](images/odh_operator_install.png)
2) Click on "Open Data Hub Operator" or "Red Hat Openshift AI". 
   1) If the "Show community Operator" warning opens, hit "Continue"
   2) Hit "Install". 
3) From the "Install Operator" screen:
   1) Make sure "All namespaces on the cluster" in selected as the "Installation Mode":
   2) Hit install
4) Wait for the Operator to finish installing

#### ❗NOTE: These demos were last verified against *RHOAI 2.25* 

## Install ODH/RHOAI 
1) Navigate to your `opendatahub` project
2) From "Installed Operators", select "Open Data Hub Operator".
3) Navigate to the "DSC Initialization" tab and hit "Create DSCInitialization", then install the default DSCI. Once the DSCI reports "Ready", move on to step 4. 
4) Navigate to the "Data Science Cluster" tab and hit "Create DataScienceCluster"
5) In the YAML view Make sure `trustyai` is set to `Managed`:
![ODH V2 YAML](images/odh_V2.png)
6) Hit the "Create" button
7) Within the "Pods" menu, you should begin to see various ODH components being created, including the `trustyai-service-operator-controller-manager-xxx`

## Set up TLS configuration
In order to ensure that TrustyAI can receive encrypted model payloads, we need to add TrustyAI's CA bundle to your model controller. Make sure to pick the right command
for your chosen operator:

### ODH Command
```bash
NAMESPACE=opendatahub
oc patch configmap inferenceservice-config -n $NAMESPACE --type merge -p '{"metadata": {"annotations": {"opendatahub.io/managed": "false"}}}'
IMAGE=$(oc get configmap inferenceservice-config -n $NAMESPACE -o json | jq -r '.data.agent | fromjson | .image')
oc patch configmap inferenceservice-config \
    -n "$NAMESPACE" \
    --type json \
    -p="[{
      \"op\": \"add\",
      \"path\": \"/data/logger\",
      \"value\": \"{\\\"image\\\" : \\\"$IMAGE\\\",\\\"memoryRequest\\\": \\\"100Mi\\\",\\\"memoryLimit\\\": \\\"1Gi\\\",\\\"cpuRequest\\\": \\\"100m\\\",\\\"cpuLimit\\\": \\\"1\\\",\\\"defaultUrl\\\": \\\"http://default-broker\\\",\\\"caBundle\\\": \\\"kserve-logger-ca-bundle\\\",\\\"caCertFile\\\": \\\"service-ca.crt\\\",\\\"tlsSkipVerify\\\": false}\"
    }]"
```

### RHOAI Command
```bash
NAMESPACE=redhat-ods-applications
oc patch configmap inferenceservice-config -n $NAMESPACE --type merge -p '{"metadata": {"annotations": {"opendatahub.io/managed": "false"}}}'
IMAGE=$(oc get configmap inferenceservice-config -n $NAMESPACE -o json | jq -r '.data.agent | fromjson | .image')
oc patch configmap inferenceservice-config \
    -n "$NAMESPACE" \
    --type json \
    -p="[{
      \"op\": \"add\",
      \"path\": \"/data/logger\",
      \"value\": \"{\\\"image\\\" : \\\"$IMAGE\\\",\\\"memoryRequest\\\": \\\"100Mi\\\",\\\"memoryLimit\\\": \\\"1Gi\\\",\\\"cpuRequest\\\": \\\"100m\\\",\\\"cpuLimit\\\": \\\"1\\\",\\\"defaultUrl\\\": \\\"http://default-broker\\\",\\\"caBundle\\\": \\\"kserve-logger-ca-bundle\\\",\\\"caCertFile\\\": \\\"service-ca.crt\\\",\\\"tlsSkipVerify\\\": false}\"
    }]"
```
# Install TrustyAI
```bash
oc project model-namespace
oc apply -f resources/trustyai.yaml
``` 
This will install the TrustyAI Service
into your `model-namespace` project, which will then provide TrustyAI features to all subsequent models deployed into that project, such as explainability, fairness monitoring, and data drift monitoring,
