# OpenShift Cluster with no GPUs

## LLMInferenceServiceConfig for CPU Only

```
apiVersion: serving.kserve.io/v1alpha2
kind: LLMInferenceServiceConfig
metadata:
  annotations:
    opendatahub.io/runtime-version: "0.7"
    opendatahub.io/supported-topologies: '["workload-single-node"]'
    openshift.io/description: DM - vLLM CPU x86 only LLMInferenceServiceConfig
    openshift.io/display-name: DM - vLLM CPU x86 only LLMInferenceServiceConfig
  name: dm-vllm-cpu-x86-only-llminferenceserviceconfig
  namespace: redhat-ods-applications
  labels:
    opendatahub.io/config-type: accelerator
    opendatahub.io/dashboard: "true"
spec:
  model:
    uri: ""
  template:
    containers:
      - image: registry.redhat.io/rhaii/vllm-cpu-rhel9@sha256:db9adba1176fea8d2e6bbc8621ca3e8aa0d69b92036f846af2ac692355114af1
        name: main
        resources: {}
```