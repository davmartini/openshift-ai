# Models monitoring

## Prerequisites

* Tempo Operator
* Red Hat build of OpenTelemetry Operator
* Cluster Observability Operator

## Configuration

1.
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