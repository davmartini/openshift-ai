# Gen AI playground

**Official Documentation:** https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/experimenting_with_models_in_the_gen_ai_playground/playground-overview_rhoai-user

## Enable Gen AI playground

1. Enable GenAI in OpenShift AI Dashboard
```
apiVersion: opendatahub.io/v1alpha
kind: OdhDashboardConfig
metadata:
  name: odh-dashboard-config
spec:
  dashboardConfig:
    disableTracking: false
    llmGatewayField: true
    genAiStudio: true           <<<--- Add this line
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