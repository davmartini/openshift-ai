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
```

3. Configure Authorino deployment with TLS certificate env vars
```
oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt
```

4. Verify the user-workload monitoring stack is enabled
```
oc wait --for=condition=Available deployment/prometheus-operator \
  -n openshift-user-workload-monitoring --timeout=300s
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

6. Annotate the Gateway for Authorino TLS bootstrap:
```
oc annotate gateway maas-default-gateway -n openshift-ingress \
  security.opendatahub.io/authorino-tls-bootstrap="true" --overwrite
```

```
oc wait --for=condition=Programmed gateway/maas-default-gateway \
  -n openshift-ingress --timeout=120s
```

7. Label the redhat-ods-applications namespace where the MaaS API route is created
```
oc label namespace redhat-ods-applications \
  maas.opendatahub.io/gateway-access=true --overwrite
```

8. Test deployement
```
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo $CLUSTER_DOMAIN
curl -vsk https://maas.${CLUSTER_DOMAIN} 2>&1 | grep -E "SSL connection|Connected"
```

> [!WARNING]
> Important step

9. Label namespace where MaaS will be used
```
oc label namespace <your-namespace> maas.opendatahub.io/gateway-access=true --overwrite
```

10. Label the namespace where the MaaS API route is created (RHOAI 3.5+)
```
oc label namespace redhat-ai-gateway-infra maas.opendatahub.io/gateway-access=true --overwrite
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

3. Verify the PostegreSQL deployement
```
oc wait --for=condition=Available deployment/postgres \
  -n redhat-ods-applications --timeout=120s
deployment.apps/postgres condition met
```

### Step 4: Enable Models-as-a-Service

1. Enable 
```
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  labels:
    app.kubernetes.io/name: datasciencecluster
  name: default-dsc
spec:
  components:
    ogx:
      managementState: Managed
    sparkoperator:
      managementState: Removed
    kserve:
      managementState: Managed
      modelCache:
        managementState: Removed
      modelsAsService:
        managementState: Removed
      nim:
        airGapped: false
        managementState: Managed
      rawDeploymentServiceConfig: Headless
      wva:
        managementState: Removed
    modelregistry:
      managementState: Managed
      registriesNamespace: rhoai-model-registries
    feastoperator:
      managementState: Managed
    trustyai:
      eval:
        lmeval:
          permitCodeExecution: deny
          permitOnline: deny
      managementState: Managed
      mcpGuardrailsMode: false
    aipipelines:
      argoWorkflowsControllers:
        managementState: Managed
      managementState: Managed
    ray:
      managementState: Managed
    kueue:
      autoCreateQueues: false
      defaultClusterQueueName: default
      defaultLocalQueueName: default
      managementState: Removed
    workbenches:
      managementState: Managed
      workbenchNamespace: rhods-notebooks
    mlflowoperator:
      managementState: Managed
    dashboard:
      managementState: Managed
    trainer:
      managementState: Managed
    llamastackoperator:
      managementState: Managed
    aigateway:
      managementState: Managed              <<<<--- Change this line
      modelsAsAService:                     <<<<--- Change this line
        managementState: Managed
    trainingoperator:
      managementState: Removed
```

```
apiVersion: opendatahub.io/v1alpha
kind: OdhDashboardConfig
metadata:
  name: odh-dashboard-config
  namespace: redhat-ods-applications
spec:
  dashboardConfig:
    disableTracking: false
    genAiStudio: true
    llmGatewayField: true
    modelAsService: true                      <<<<--- Add this line
    observabilityDashboard: true              <<<<--- Add this line
    vLLMDeploymentOnMaaS: true                <<<<--- Add this line
    maasAuthPolicies: true                    <<<<--- Add this line
  hardwareProfileOrder:
    - default-profile
    - nvidia-l40-profile
  notebookController:
    enabled: true
    notebookNamespace: rhods-notebooks
    pvcSize: 20Gi
  templateDisablement: []
  templateOrder: []
```

2. Verify 
```
oc get crd | grep maas.opendatahub.io
aitenants.maas.opendatahub.io                                                      2026-09-02T13:00:36Z
configs.maas.opendatahub.io                                                        2026-09-02T13:00:37Z
externalmodels.maas.opendatahub.io                                                 2026-09-02T13:00:37Z
maasauthpolicies.maas.opendatahub.io                                               2026-09-02T13:00:37Z
maasmodelrefs.maas.opendatahub.io                                                  2026-09-02T13:00:37Z
maassubscriptions.maas.opendatahub.io                                              2026-09-02T13:00:37Z
maastenantconfigs.maas.opendatahub.io                                              2026-09-02T13:00:37Z
tenants.maas.opendatahub.io                                                        2026-09-02T13:00:37Z
```

## Deploy a model with MaaS

> [!WARNING]
> verify than you have this label on your namespace where you deploy your model **maas.opendatahub.io/gateway-access: 'true'**

![llm-maas](../images/llm-maas1.png)

![llm-maas](../images/llm-maas2.png)

![llm-maas](../images/llm-maas3.png)

![llm-maas](../images/llm-maas4.png)

## Add MaaS subscription


#### What's a subscription

In Red Hat OpenShift AI, you can use Models-as-a-Service (MaaS) subscriptions to manage quotas and token limits for AI model serving. With subscriptions, you can grant specific groups quotas for models with configurable token limits based on user group membership.

When multiple teams share large language models, you can use subscriptions to perform the following tasks:

* Prevent resource exhaustion by enforcing token limits per model
* Provide different access levels for different user groups
* Track and allocate costs based on team consumption
* Control which teams can access high-cost or sensitive models
* Allow users to belong to multiple subscriptions based on their group memberships

MaaS assigns users to subscriptions based on their OpenShift group membership. When a user belongs to multiple groups with different subscriptions, the system uses the subscription with the highest priority level.

1. Create a Authorization policies 

![maas-auth](../images/maas-auth-policy.png)

2. Create a MaaS Production subscription

![maas-auth](../images/maas-prod-subscription.png)

3. Create a MaaS QA subscription

![maas-auth](../images/maas-qa-subscription.png)

![maas-auth](../images/maas-overview-subscription.png)

4. Generate API Key

![api-key](../images/api-key-01.png)

![api-key](../images/api-key-02.png)

![api-key](../images/api-key-03.png)

## Test model access via MaaS

1. Define env variables
```
# MaaS endpoint
DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
HOST="https://maas.${DOMAIN}"

# Personal API key
API_KEY=<YOUR API KEY>
```

2. List available model
```
curl -sk -H "Authorization: Bearer ${API_KEY}" "${HOST}/v1/models" |jq .
{
  "data": [
    {
      "id": "publishers/dmartini-ai/models/redhataiqwen25-7b-instruct-maa",
      "created": 1788416766,
      "object": "model",
      "owned_by": "dmartini-ai/redhataiqwen25-7b-instruct-maa",
      "kind": "LLMInferenceService",
      "url": "https://maas.apps.cluster-p42wn.p42wn.sandbox1365.opentlc.com/",
      "ready": true,
      "modelDetails": {
        "displayName": "RedHatAI/Qwen2.5-7B-Instruct-MaaS"
      },
      "subscriptions": [
        {
          "name": "prodution-subscription",
          "displayName": "prodution-subscription"
        }
      ]
    }
  ],
  "object": "list"
}
```

```
# Env model
MODEL=$(curl -sk -H "Authorization: Bearer ${API_KEY}" "${HOST}/v1/models" | jq -r '.data[0].id')
```

3. Request available model via MaaS
```
for i in $(seq 1 16); do
  curl -sk -H "Authorization: Bearer ${API_KEY}" \
    -H "Content-Type: application/json" \
    -X POST "${HOST}/v1/chat/completions" \
    -d '{"model": "publishers/dmartini-ai/models/redhataiqwen25-7b-instruct-maa","messages": [{"role": "user", "content": "Bonjour, peux-tu te présenter ?"}],"max_tokens":50}'
    "${MODEL_URL}/v1/chat/completions"
done
```


## Enable MaaS Observability (metriques)

### Prerequisites

* Tempo Operator
* Red Hat build of OpenTelemetry Operator
* Cluster Observability Operator

### Configuration

1. Enable Monitoring for MaaS
```
oc patch dsci default-dsci --type=merge -p '{
  "spec": {
    "monitoring": {
      "namespace": "redhat-ods-monitoring",
      "metrics": {
        "replicas": 1,
        "storage": {
          "size": "5Gi",
          "retention": "90d"
        }
      },
      "traces": {
        "sampleRatio": "0.1",
        "storage": {
          "backend": "pv",
          "retention": "2160h"
        }
      }
    }
  }
}'
```

2. Check deployment
```
oc get pods -n redhat-ods-monitoring
NAME                                                          READY   STATUS    RESTARTS   AGE
alertmanager-data-science-monitoringstack-0                   2/2     Running   0          105s
alertmanager-data-science-monitoringstack-1                   2/2     Running   0          105s
data-science-collector-collector-0                            1/1     Running   0          107s
data-science-collector-collector-1                            1/1     Running   0          107s
data-science-collector-targetallocator-674cf6bf9d-pnf5m       1/1     Running   0          106s
data-science-perses-0                                         1/1     Running   0          107s
data-science-prometheus-cluster-proxy-7955589bdb-zw4zp        1/1     Running   0          107s
data-science-prometheus-namespace-proxy-56fd5f657c-hz8p7      2/2     Running   0          107s
prometheus-data-science-monitoringstack-0                     3/3     Running   0          105s
tempo-data-science-tempomonolithic-0                          3/3     Running   0          105s
thanos-querier-data-science-thanos-querier-7857b974bb-2mz8n   1/1     Running   0          107s
```

3. Apply Gateway Telemetry if no exist
```
apiVersion: extensions.kuadrant.io/v1alpha1
kind: TelemetryPolicy
metadata:
  name: maas-telemetry
  namespace: openshift-ingress
  labels:
    app.kubernetes.io/part-of: maas-observability
spec:
  metrics:
    default:
      labels:
        model: responseBodyJSON("/model")
        user: auth.identity.userid
        # Subscription metadata for usage attribution and billing
        subscription: auth.identity.selected_subscription
        # Before RHOAI 3.4.4 (RHOAIENG-79318) organization_id and cost_center must be omitted
        # organization_id: auth.identity.subscription_info.organizationId
        # cost_center: auth.identity.subscription_info.costCenter
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: maas-default-gateway
```

```
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: latency-per-subscription
  namespace: openshift-ingress
spec:
  selector:
    matchLabels:
      gateway.networking.k8s.io/gateway-name: maas-default-gateway
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_DURATION
        mode: CLIENT_AND_SERVER
      tagOverrides:
        subscription:
          operation: UPSERT
          value: request.headers["x-maas-subscription"]
```

### Monitoring Dashboaerds

#### Cluster dashboard

![maas-monitoring](../images/maas-monitoring1.png)

#### Model dashboard

![maas-monitoring](../images/maas-monitoring2.png)

#### LLM Trafic dashboard

![maas-monitoring](../images/maas-monitoring3.png)

#### MaaS subscription usage dashboard

![maas-monitoring](../images/maas-monitoring4.png)

#### LLM Performance dashboard

![maas-monitoring](../images/maas-monitoring5.png)


## Enable MaaS Observability (logs)


### Configruation

1. Install loki operator

2. Deploy a S3 if you don't have it

```
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
  namespace: redhat-ods-monitoring
stringData:
  access_key_id: minio
  access_key_secret: minio123
  bucketnames: loki
  endpoint: http://minio.redhat-ods-monitoring.svc:9000
  region: us-east-1
type: Opaque
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data
  namespace: redhat-ods-monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: redhat-ods-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: quay.io/minio/minio:latest
          command:
            - minio
            - server
            - /data
            - --console-address
            - ":9001"
          env:
            - name: MINIO_ROOT_USER
              value: minio
            - name: MINIO_ROOT_PASSWORD
              value: minio123
          ports:
            - containerPort: 9000
              name: api
            - containerPort: 9001
              name: console
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-data
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: redhat-ods-monitoring
spec:
  selector:
    app: minio
  ports:
    - port: 9000
      targetPort: 9000
      name: api
    - port: 9001
      targetPort: 9001
      name: console
---
apiVersion: batch/v1
kind: Job
metadata:
  name: minio-create-bucket
  namespace: redhat-ods-monitoring
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: mc
          image: quay.io/minio/mc:latest
          env:
            - name: HOME
              value: /tmp
          command:
            - /bin/sh
            - -c
            - |
              until mc alias set local http://minio.redhat-ods-monitoring.svc:9000 minio minio123; do
                echo "Waiting for MinIO..."
                sleep 5
              done
              mc mb local/loki --ignore-existing
              echo "Bucket 'loki' ready"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
  backoffLimit: 5
```

3. Deploy loki stack
```
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: usage
  namespace: redhat-ods-monitoring
spec:
  limits:
    global:
      otlp:
        streamLabels:
          resourceAttributes:
            - name: service.name
            - name: subscription
            - name: model
            - name: response_type
            - name: kubernetes_namespace_name
  managementState: Managed
  size: 1x.demo
  storage:
    schemas:
      - effectiveDate: "2024-10-01"
        version: v13
    secret:
      credentialMode: static
      name: minio-secret
      type: s3
  storageClassName: gp3-csi
  tenants:
    mode: openshift-logging
```