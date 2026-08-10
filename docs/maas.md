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

> [!WARNING]
> Thses steps are specific to OCP on AWS. Other steps are required for OCP on BM.

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
apps.cluster-r8w6p.r8w6p.sandbox721.opentlc.com
```

```
export CERT_NAME=$(oc get ingresscontroller default \
  -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate.name}' 2>/dev/null)
export CERT_NAME="${CERT_NAME:-router-certs-default}"
echo $CERT_NAME
cert-manager-ingress-cert
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

6. Annotate the Gateway for Authorino TLS bootstrap:
```
oc annotate gateway maas-default-gateway -n openshift-ingress \
  security.opendatahub.io/authorino-tls-bootstrap="true" --overwrite
```

```
oc wait --for=condition=Programmed gateway/maas-default-gateway \
  -n openshift-ingress --timeout=120s
gateway.gateway.networking.k8s.io/maas-default-gateway condition met
```

7. Label the redhat-ods-applications namespace where the MaaS API route is created
```
oc label namespace redhat-ods-applications \
  maas.opendatahub.io/gateway-access=true --overwrite
namespace/redhat-ods-applications labeled
```

8. Test deployement
```
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo $CLUSTER_DOMAIN
curl -vsk https://maas.${CLUSTER_DOMAIN} 2>&1 | grep -E "SSL connection|Connected"
```

### Step 3: Create a PostgreSQL BDD for Models-as-a-Service

1. Create the PostgreSQL secret and configuration
```
# Generate a random password
POSTGRES_PASSWORD=$(openssl rand -base64 16 | tr -d '=+/')

# Create postgres-creds (consumed by the PostgreSQL deployment)
oc create secret generic postgres-creds \
  -n redhat-ods-applications \
  --from-literal=POSTGRES_USER=maas \
  --from-literal=POSTGRES_PASSWORD="${POSTGRES_PASSWORD}" \
  --from-literal=POSTGRES_DB=maas

# Create maas-db-config (consumed by maas-api)
oc create secret generic maas-db-config \
  -n redhat-ods-applications \
  --from-literal=DB_CONNECTION_URL="postgresql://maas:${POSTGRES_PASSWORD}@postgres:5432/maas?sslmode=disable"
```

2. Deploy a PostgreSQL instance
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: redhat-ods-applications
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: redhat-ods-applications
  labels:
    app: postgres
    purpose: poc
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: redhat-ods-applications
  labels:
    app: postgres
    purpose: poc
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: registry.redhat.io/rhel9/postgresql-16:latest
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRESQL_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-creds
                  key: POSTGRES_USER
            - name: POSTGRESQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-creds
                  key: POSTGRES_PASSWORD
            - name: POSTGRESQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: postgres-creds
                  key: POSTGRES_DB
            - name: PGDATA
              value: /var/lib/pgsql/data/userdata
          readinessProbe:
            exec:
              command:
                - /usr/libexec/check-container
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U "$POSTGRESQL_USER" -d "$POSTGRESQL_DATABASE"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgsql/data
              subPath: userdata
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
```