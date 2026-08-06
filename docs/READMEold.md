# OCP GPU Setup

Complete setup guide for enabling NVIDIA GPU support in OpenShift Container Platform (OCP) clusters running on AWS.

## Overview





## Clone the Repository

```bash
git clone https://github.com/rh-aiservices-bu/ocp-gpu-setup.git
cd ocp-gpu-setup
```



## Step 6: Create a new accelerator profile

**Via YAML:**
```
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  name: nvidia-l40s-private
  namespace: redhat-ods-applications
spec:
  identifiers:
    - defaultCount: 2
      displayName: CPU
      identifier: cpu
      maxCount: 4
      minCount: 1
      resourceType: CPU
    - defaultCount: 4Gi
      displayName: Memory
      identifier: memory
      maxCount: 8Gi
      minCount: 2Gi
      resourceType: Memory
    - defaultCount: 1
      displayName: GPU
      identifier: nvidia.com/gpu
      maxCount: 2
      minCount: 1
      resourceType: Accelerator
  scheduling:
    node:
      nodeSelector: {}
      tolerations:
        - effect: NoSchedule
          key: nvidia.com/gpu
          operator: Equal
          value: NVIDIA-L40S-PRIVATE
    type: Node
```

**Via RHOAI Web UI:**

![image](images/hardware-profile.png)

## Step 7: Deploy a model with RHOAI and Kserv

![image](images/model-deployement.png)

![image](images/model-deployement2.png)

## Step 8: Deploy a distributed model with RHOAI and llm-d

**Doc:** https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html/deploying_models/deploying_models#deploying-models-using-distributed-inference_rhoai-user 

**Requirements:**

1. Installing OpenShift AI.
2. Enabling the model serving platform.
3. Configuring authentication with Red Hat Connectivity Link.
4. Enabling Distributed Inference with llm-d on a Kubernetes cluster.
5. Creating an LLMInferenceService Custom Resource (CR).
6. Deploying a model.

### Steps

1. Create a new GatewayClass:
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
 name: openshift-ai-inference
spec:
 controllerName: openshift.io/gateway-controller/v1
```
2. Create a new Gateway:
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
 name: openshift-ai-inference
 namespace: openshift-ingress
spec:
 gatewayClassName: openshift-ai-inference
 listeners:
   - name: https
     port: 443
     protocol: HTTPS
     hostname: inference-gateway.apps.test-rc3.rhoai.rh-aiservices-bu.com
     tls:
       mode: Terminate
       certificateRefs:
         - kind: Secret
           name: default-gateway-tls
     allowedRoutes:
       namespaces:
         from: Selector
         selector:
           matchExpressions:
             - key: kubernetes.io/metadata.name
               operator: In
               values:
                 - openshift-ingress
                 - redhat-ods-applications
                 - dmartini
```

3. Create a LLMInferenceService:

```
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: granite-llm-d-inference-service
  namespace: dmartini
spec:
  replicas: 2
  model:
    uri: oci://registry.redhat.io/rhelai1/modelcar-granite-3-1-8b-lab-v1:1.4.0
    name: RedHatAI/granite-3-1-8b
  router:
    route: {}
    gateway: {}
    scheduler: {}
  template:
    tolerations:
      - key: nvidia.com/gpu
        operator: Equal
        value: NVIDIA-L40S-PRIVATE
        effect: NoSchedule
    containers:
    - name: main
      resources:
        limits:
          cpu: '4'
          memory: 32Gi
          nvidia.com/gpu: "1"
        requests:
          cpu: '2'
          memory: 16Gi
          nvidia.com/gpu: "1"
```