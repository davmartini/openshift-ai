# Models-as-a-Service (MaaS)

**Official Documentation:** https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/deploy-and-manage-models-as-a-service_maas

**RHOAI Models-as-a-Service Guide:** https://github.com/rh-aiservices-bu/rhoai-maas-guide

## Enable MaaS

### Step 1: Kuadrant and Authorino

1. Create the kuadrant-system namespace
```
kind: Namespace
apiVersion: v1
metadata:
  name: kuadrant-system
```

2. Create Authorino Service
```
apiVersion: v1
kind: Service
metadata:
  name: authorino-authorino-authorization
  namespace: kuadrant-system
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: authorino-server-cert
spec:
  ports:
    - name: grpc
      port: 50051
      protocol: TCP
      targetPort: 50051
    - name: http
      port: 5001
      protocol: TCP
      targetPort: 5001
  selector:
    authorino-resource: authorino
  type: ClusterIP
  ```

3. Create kuadrant CRD
```
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
spec:
  # Enable observability so the Kuadrant operator creates its own
  # Limitador/Authorino PodMonitor/ServiceMonitor resources that are
  # scraped by user-workload monitoring. Required for MaaS observability.
  observability:
    enable: true
```

### Step 2: Configure TLS for Models-as-a-Service

1. Check authorino cert
```
oc get secret authorino-server-cert -n kuadrant-system
NAME                    TYPE                DATA   AGE
authorino-server-cert   kubernetes.io/tls   2      2m29s
dmartini@dmartini-mac ~ % 
```

2. Patch the Authorino CR to enable the TLS listener
```
oc patch authorino authorino -n kuadrant-system --type=merge --patch '{
  "spec": {
    "listener": {
      "tls": {
        "enabled": true,
        "certSecretRef": {
          "name": "authorino-server-cert"
        }
      }
    }
  }
}'
authorino.operator.authorino.kuadrant.io/authorino patched
```

3. Configure Authorino deployment with TLS certificate env vars
```
oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt
deployment.apps/authorino updated
```

4. Verify the user-workload monitoring stack is enabled
```
oc wait --for=condition=Available deployment/prometheus-operator \
  -n openshift-user-workload-monitoring --timeout=300s
deployment.apps/prometheus-operator condition met
```

5. Create the MaaS Gateway
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: maas-gateway-options
  namespace: openshift-ingress
data:
  deployment: |
    spec:
      template:
        spec:
          containers:
          - name: istio-proxy
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: "2"
                memory: 2Gi
```

```
export CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster \
  -o jsonpath='{.spec.domain}')
echo $CLUSTER_DOMAIN
```

```
export CERT_NAME=$(oc get ingresscontroller default \
  -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate.name}' 2>/dev/null)
export CERT_NAME="${CERT_NAME:-router-certs-default}"
echo $CERT_NAME
```

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"
    security.opendatahub.io/authorino-tls-bootstrap: "true"
spec:
  gatewayClassName: openshift-default
  infrastructure:
    parametersRef:
      group: ""
      kind: ConfigMap
      name: maas-gateway-options
  listeners:
   - name: http
     hostname: maas.${CLUSTER_DOMAIN}                     <<<--- Change value
     port: 80
     protocol: HTTP
     allowedRoutes:
       namespaces:
         from: Selector
         selector:
           matchLabels:
             maas.opendatahub.io/gateway-access: "true"
   - name: https
     hostname: maas.${CLUSTER_DOMAIN}                     <<<--- Change value
     port: 443
     protocol: HTTPS
     allowedRoutes:
       namespaces:
         from: Selector
         selector:
           matchLabels:
             maas.opendatahub.io/gateway-access: "true"
     tls:
       certificateRefs:
       - group: ""
         kind: Secret
         name: ${CERT_NAME}                              <<<--- Change value
       mode: Terminate
```