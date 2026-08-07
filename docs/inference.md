# LLM Inference on OpenShift AI




## llm-d and distributed inference

**Official documentation:** https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/deploy_models_using_distributed_inference_with_llm-d/index

### Configuration

1. Create a GatewayClass
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1
```

2. Create a Gateway
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openshift-ai-inference
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-default
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: router-certs-default
```

3. Enable Gateway discovery in the dashboard configuration
```
oc patch odhdashboardconfig odh-dashboard-config \
-n redhat-ods-applications \
--type merge \
-p '{"spec":{"dashboardConfig":{"llmGatewayField":true}}}'
```
### Model deployement with llm-d via OpenShift AI WebUI

![llmd](../images/llmd1.png)

![llmd](../images/llmd2.png)

![llmd](../images/llmd3.png)

![llmd](../images/llmd4.png)

![llmd](../images/llmd5.png)

![llmd](../images/llmd6.png)

![llmd](../images/llmd7.png)


### llm-d observability

1. Enable enableUserWorkload for OpenShift Observability
```
cat << EOF | oc create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
configmap/cluster-monitoring-config created
```