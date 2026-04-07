# 2. Kubernetes Deployment — Production-Grade Orchestration

---

## ELI5

Docker runs ONE container on ONE machine. Kubernetes (K8s) runs MANY containers across MANY machines, handles failures automatically, scales up/down based on load, does rolling updates with zero downtime, and heals itself.

Think of Kubernetes as an "operating system for your cluster." You declare what you want (3 API pods, always), and K8s makes it happen and keeps it that way.

---

## Project Structure

```
k8s/
├── namespace.yaml
├── configmaps/
│   ├── qdrant-config.yaml
│   └── prometheus-config.yaml
├── secrets/              # managed by External Secrets Operator (not checked in)
├── qdrant/
│   ├── statefulset.yaml  # Qdrant cluster
│   ├── service.yaml
│   └── pdb.yaml          # PodDisruptionBudget
├── api/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml          # HorizontalPodAutoscaler
│   └── pdb.yaml
├── ingestion/
│   ├── deployment.yaml
│   └── hpa.yaml
├── redis/
│   ├── statefulset.yaml
│   └── service.yaml
└── ingress/
    ├── ingress.yaml
    └── certificate.yaml  # cert-manager
```

---

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vector-search
  labels:
    name: vector-search
    environment: production
```

---

## Qdrant StatefulSet (Production-Grade)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: vector-search
spec:
  serviceName: qdrant-headless
  replicas: 3
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "6333"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: qdrant

      # Never run 2 Qdrant pods on the same node
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: [qdrant]
              topologyKey: kubernetes.io/hostname

      # Spread pods across AZs
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: qdrant

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      initContainers:
        # Ensure storage directory has correct permissions
        - name: fix-permissions
          image: busybox:1.36
          command: ["sh", "-c", "chown -R 1000:1000 /qdrant/storage"]
          volumeMounts:
            - name: qdrant-storage
              mountPath: /qdrant/storage

      containers:
        - name: qdrant
          image: qdrant/qdrant:v1.11.3
          ports:
            - containerPort: 6333
              name: http
            - containerPort: 6334
              name: grpc
            - containerPort: 6335
              name: p2p
          env:
            - name: QDRANT__CLUSTER__ENABLED
              value: "true"
            - name: QDRANT__CLUSTER__P2P__PORT
              value: "6335"
            - name: QDRANT__SERVICE__API_KEY
              valueFrom:
                secretKeyRef:
                  name: qdrant-secrets
                  key: api_key
          resources:
            requests:
              memory: "4Gi"
              cpu: "2"
            limits:
              memory: "8Gi"
              cpu: "4"
          volumeMounts:
            - name: qdrant-storage
              mountPath: /qdrant/storage
            - name: qdrant-config
              mountPath: /qdrant/config/production.yaml
              subPath: production.yaml
          livenessProbe:
            httpGet:
              path: /healthz
              port: 6333
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: 6333
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                # Graceful shutdown: wait for in-flight requests
                command: ["/bin/sh", "-c", "sleep 10"]

      volumes:
        - name: qdrant-config
          configMap:
            name: qdrant-config

  volumeClaimTemplates:
    - metadata:
        name: qdrant-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: encrypted-gp3
        resources:
          requests:
            storage: 200Gi
```

---

## Qdrant Services

```yaml
# Headless service for cluster peer discovery
apiVersion: v1
kind: Service
metadata:
  name: qdrant-headless
  namespace: vector-search
spec:
  clusterIP: None   # headless: returns all pod IPs, not a virtual IP
  selector:
    app: qdrant
  ports:
    - name: http
      port: 6333
    - name: grpc
      port: 6334
    - name: p2p
      port: 6335

---
# Load-balanced service for client connections
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: vector-search
spec:
  selector:
    app: qdrant
  ports:
    - name: http
      port: 6333
    - name: grpc
      port: 6334
  type: ClusterIP   # internal only; never expose Qdrant externally
```

---

## API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: vector-search
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # zero downtime: never reduce below desired replicas
      maxSurge: 1            # allow 1 extra pod during rollout
  template:
    metadata:
      labels:
        app: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: api-service

      # Anti-affinity: spread API pods across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: api
                topologyKey: kubernetes.io/hostname

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        readOnlyRootFilesystem: true   # container can't write to filesystem

      containers:
        - name: api
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/vector-search:${IMAGE_TAG}
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          env:
            - name: QDRANT_URL
              value: "http://qdrant.vector-search.svc.cluster.local:6333"
            - name: QDRANT_API_KEY
              valueFrom:
                secretKeyRef:
                  name: qdrant-secrets
                  key: api_key
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-secrets
                  key: url
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-secrets
                  key: api_key
            - name: LOG_LEVEL
              value: "INFO"
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: tmp
              mountPath: /tmp   # writable temp dir (readOnlyRootFilesystem=true)

      volumes:
        - name: tmp
          emptyDir: {}

      terminationGracePeriodSeconds: 60   # wait 60s for in-flight requests

---
# API Service
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: vector-search
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

---

## Horizontal Pod Autoscaler

```yaml
# Scale API based on CPU and custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: vector-search
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale up when avg CPU > 70%

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # wait 60s before another scale-up
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60    # add max 2 pods per 60s
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5min before scale-down (prevent flapping)
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120   # remove max 1 pod per 2min

---
# HPA for ingestion workers (scale on Kafka consumer lag)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ingestion-worker-hpa
  namespace: vector-search
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ingestion-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumer_lag_sum
          selector:
            matchLabels:
              topic: vector-ingestion
        target:
          type: AverageValue
          averageValue: 1000   # scale up when consumer lag > 1000 messages
```

---

## PodDisruptionBudget

```yaml
# Qdrant: always keep at least 2 pods running during maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: qdrant-pdb
  namespace: vector-search
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: qdrant

---
# API: always keep 2 pods for availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: vector-search
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

---

## Ingress with TLS

```yaml
# cert-manager: ClusterIssuer (Let's Encrypt)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourcompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx

---
# Ingress: TLS termination + routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: vector-search
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "100"
spec:
  tls:
    - hosts:
        - api.yourcompany.com
      secretName: api-tls
  rules:
    - host: api.yourcompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

---

## Useful kubectl Commands

```bash
# Deploy / update
kubectl apply -f k8s/ -n vector-search

# Rolling restart (pick up new secrets, configmaps)
kubectl rollout restart deployment/api -n vector-search

# Watch rollout
kubectl rollout status deployment/api -n vector-search

# Check pod status
kubectl get pods -n vector-search -w

# Check events (useful for debugging scheduling issues)
kubectl get events -n vector-search --sort-by='.lastTimestamp'

# View logs
kubectl logs -l app=api -n vector-search --tail=100 -f

# Exec into pod (for debugging)
kubectl exec -it deployment/api -n vector-search -- bash

# Scale manually (override HPA temporarily)
kubectl scale deployment/api --replicas=10 -n vector-search

# Port-forward for local testing
kubectl port-forward svc/qdrant 6333:6333 -n vector-search
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| StatefulSet | For Qdrant (ordered, stable pod names, persistent volumes) |
| Deployment | For API and workers (stateless, rolling updates) |
| Anti-affinity | No two Qdrant pods on same node (required for HA) |
| TopologySpread | Spread pods across AZs |
| PDB | minAvailable=2 prevents K8s from evicting too many pods |
| HPA | Scale on CPU% for API; Kafka consumer lag for workers |
| RollingUpdate | maxUnavailable=0 ensures zero-downtime deployments |
| readOnlyRootFilesystem | Security hardening; use emptyDir for /tmp |
| cert-manager | Auto-renews Let's Encrypt certs before expiry |
