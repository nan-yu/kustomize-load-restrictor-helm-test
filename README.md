## Folder structure

```
.
├── base
├── components
│   ├── 1.2.3
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   ├── 1.3.0
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   ├── 1.3.5
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   └── coredns
│       ├── Chart.yaml
│       ├── README.md
│       ├── templates
│       │   ├── clusterrole-autoscaler.yaml
│       │   ├── clusterrolebinding-autoscaler.yaml
│       │   ├── clusterrolebinding.yaml
│       │   ├── clusterrole.yaml
│       │   ├── configmap-autoscaler.yaml
│       │   ├── configmap.yaml
│       │   ├── deployment-autoscaler.yaml
│       │   ├── deployment.yaml
│       │   ├── _helpers.tpl
│       │   ├── hpa.yaml
│       │   ├── NOTES.txt
│       │   ├── poddisruptionbudget.yaml
│       │   ├── podsecuritypolicy.yaml
│       │   ├── serviceaccount-autoscaler.yaml
│       │   ├── serviceaccount.yaml
│       │   ├── service-metrics.yaml
│       │   ├── servicemonitor.yaml
│       │   └── service.yaml
│       └── values.yaml
└── overlays
    └── dev
        ├── kustomization.yaml
        └── values.yaml
```

## Kustomizaiton configurations

There are four kustomization configuration files, each referencing a values.yaml outside its root kustomization folder.

- `overlays/dev/values.yaml` sets the image tag and overwrites the image pull policy to `Always`.
- Other values.yaml files in the component directories set the image tag and resource requirements.

## Kustomize build

1. Build error with the default `LoadRestrictionsRootOnly` option:

```
$ kustomize build --enable-helm overlays/dev/
Error: security; file '/usr/local/home/kustomize-load-restrictor-test/components/1.3.0/values.yaml' is not in or below '/usr/local/home/kustomize-load-restrictor-test/overlays/dev'
```

2. Override the load restrictor:

```
$ kustomize build --enable-helm --load-restrictor=LoadRestrictionsNone overlays/dev/
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: coredns
    helm.sh/chart: coredns-1.16.4
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: coredns-coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: coredns
    helm.sh/chart: coredns-1.16.4
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: coredns-coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: coredns-coredns
subjects:
- kind: ServiceAccount
  name: default
  namespace: coredns-system
---
apiVersion: v1
data:
  Corefile: |-
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus 0.0.0.0:9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: coredns
    helm.sh/chart: coredns-1.16.4
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: coredns-coredns
  namespace: coredns-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: coredns
    helm.sh/chart: coredns-1.16.4
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: coredns-coredns
  namespace: coredns-system
spec:
  ports:
  - name: udp-53
    port: 53
    protocol: UDP
  - name: tcp-53
    port: 53
    protocol: TCP
  selector:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/name: coredns
    k8s-app: coredns
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: coredns
    app.kubernetes.io/version: 1.8.4
    helm.sh/chart: coredns-1.16.4
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: coredns-coredns
  namespace: coredns-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: coredns
      app.kubernetes.io/name: coredns
      k8s-app: coredns
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      annotations:
        checksum/config: 4bcdaa70b3ae3e79eb77abcc33200594a50195363ba12e705b7f3571f0241a10
        scheduler.alpha.kubernetes.io/critical-pod: ""
        scheduler.alpha.kubernetes.io/tolerations: '[{"key":"CriticalAddonsOnly",
          "operator":"Exists"}]'
      labels:
        app.kubernetes.io/instance: coredns
        app.kubernetes.io/name: coredns
        k8s-app: coredns
    spec:
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        image: coredns/coredns:1.8.5
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: coredns
        ports:
        - containerPort: 53
          name: udp-53
          protocol: UDP
        - containerPort: 53
          name: tcp-53
          protocol: TCP
        readinessProbe:
          failureThreshold: 5
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        resources:
          limits:
            cpu: 130m
            memory: 130Mi
          requests:
            cpu: 130m
            memory: 130Mi
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
      dnsPolicy: Default
      serviceAccountName: default
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          items:
          - key: Corefile
            path: Corefile
          name: coredns-coredns
        name: config-volume
```



