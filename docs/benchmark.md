# Model Benchmark

1. Start job
```
cat << 'EOF' | oc apply -f-
apiVersion: batch/v1
kind: Job
metadata:
  name: benchmark-llm
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: guidellm
          image: ghcr.io/vllm-project/guidellm:latest
          env:
            - name: GUIDELLM_TARGET
              value: "http://llama-vllm-scaled-lb:8000/v1"
            - name: GUIDELLM_PROFILE
              value: concurrent
            - name: GUIDELLM_RATE
              value: "32,64"
            - name: GUIDELLM_OUTPUTS
              value: json,csv
            - name: GUIDELLM_MAX_SECONDS
              value: "300"
            - name: GUIDELLM_DATA
              value: "/data/prompts.csv"
            - name: GUIDELLM_PROCESSOR
              value: "RedHatAI/Meta-Llama-3.1-8B-Instruct-FP8-dynamic"
            - name: HF_HOME
              value: /tmp/hf-cache
            - name: TRANSFORMERS_CACHE
              value: /tmp/hf-cache
            - name: XDG_CACHE_HOME
              value: /tmp
          volumeMounts:
            - name: cache
              mountPath: /tmp/hf-cache
            - name: results
              mountPath: /results
            - name: data
              mountPath: /data
              readOnly: true
      volumes:
        - name: cache
          emptyDir: {}
        - name: results
          emptyDir: {}
        - name: data
          persistentVolumeClaim:
            claimName: benchmark-data
EOF

oc wait --for=condition=ready pod -l job-name=guidellm-vllm-scaled --timeout=120s
oc logs -f job/guidellm-vllm-scaled
```