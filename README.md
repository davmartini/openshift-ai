# Openshit / OpenShift AI Lab

## Description

This repository resume all tests performed on a OpenShift cluster.

## Lab environment

* **Type:** Self-Managed OpenShift on AWS
* **OpenSift Version:** 4.22
* **OpenShift AI Version:** 3.4
* **NVIDIA GPU used** L40 (g6.2xlarge)

# Official docs

* **OpenShift:** https://docs.redhat.com/en/documentation/openshift_container_platform/4.22
* **OpenShift AI:** https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/
* **NVIDIA:** https://docs.nvidia.com/datacenter/cloud-native/openshift/26.3/index.html

## Parts

[1. OpenShift prerequisites](./docs/openshift-install.md)  
[2. GPU MachineSet and GPU configuration](./docs/gpu-configuration.md)  
[3. NVIDIA MIG](./docs/mig.md)  
[4. Hardware profiles](./docs/hardware-profiles.md)   
[5. NoteBook](./docs/notebook.md)  
[6. LLM inference](./docs/inference.md)  
[7. Gen AI playground](./docs/playground.md)  
[8. Models-as-a-Service (MaaS)](./docs/maas.md)  
[Red Hat Build of Kueue](./docs/kueue)  
[DRA](./docs/dra.md)  