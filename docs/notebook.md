# Notebook

## 1. Create a project
![notebook](../images/project-creation1.png)

![notebook](../images/project-creation2.png)

## 2. Create a workbench
![notebook](../images/workbench-creation1.png)

![notebook](../images/workbench-creation2.png)

![notebook](../images/workbench-creation3.png)

![notebook](../images/workbench-creation4.png)

## 3. Access to your workbench
![notebook](../images/workbench-creation5.png)

![notebook](../images/workbench-creation6.png)

## 4. Workbench = Pod in OpenShift
![notebook](../images/workbench-pod.png)

## 5. Check GPU usage
```
oc describe node | egrep 'Name:|Capacity|nvidia.com/gpu:|Allocatable:'
Name:               ip-10-0-11-46.us-east-2.compute.internal
Capacity:
Allocatable:
Name:               ip-10-0-14-232.us-east-2.compute.internal
Capacity:
Allocatable:
Name:               ip-10-0-3-90.us-east-2.compute.internal
Capacity:
Allocatable:
Name:               ip-10-0-31-95.us-east-2.compute.internal
Capacity:
Allocatable:
Name:               ip-10-0-38-13.us-east-2.compute.internal
Capacity:
  nvidia.com/gpu:     1
Allocatable:
  nvidia.com/gpu:     1
Name:               ip-10-0-63-4.us-east-2.compute.internal
Capacity:
Alloc