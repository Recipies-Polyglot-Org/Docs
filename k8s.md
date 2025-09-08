# Kubernetes Best Practices

## Ingress
I have implemented the ingress-service through nordport and exposed my backened and frontend through ingress, by this only ingress can access my frontend and backend.

## Frontend(React)

### Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: akshat1414/frontend-react:latest
        imagePullPolicy: Always
        securityContext:
          allowPrivilegeEscalation: false
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "50Mi"
          limits:
            cpu: "200m"
            memory: "100Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 20
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3

```

1. **Using appropriate limits and probes for health checks** and reasonable **initialDelaySeconds** so that it waits for it to start before probing.
2. **Added security context**, This is a strong security measure that prevents a process from gaining higher privileges within the container, which is a good practice for minimizing risk.


### Network Policy

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-net-pol
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```


1. **Using Ingress rule** : The policy allows traffic to the frontend pod on port 80, but only from pods within the ingress-nginx namespace. This effectively isolates my frontend application, ensuring it only receives traffic from my designated ingress controller.

2. **Using egress rule** :  By allowing outbound traffic only to the kube-dns pod on port 53 (UDP), i am implementing the principle of least privilege. This prevents the frontend pod from initiating connections to unexpected or potentially malicious external services, which is a critical security practice.


## Java services

### Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment-apigate
  labels:
    app: backend-apigate
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend-apigate
  template:
    metadata:
      labels:
        app: backend-apigate
    spec:
      containers:
      - name: backend-apigate
        image: akshat1414/api-gate-microservice:latest
        imagePullPolicy: Always
        securityContext:
          allowPrivilegeEscalation: false
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
          limits:
            cpu: "500m"
            memory: "400Mi"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 20
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
```
1. Using appropriate limits and probes for **health checks** and reasonable **initialDelaySeconds** so that it waits for it to start before probing. also added actuator depenedency for health checks.

2. Added **security context**, This is a strong security measure that prevents a process from gaining higher privileges within the container, which is a good practice for minimizing risk.


### HPA

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apigate-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment-apigate
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

1. **Added hpa and metrics server** to monitor the resource usage, and to increase the replicas for load distribution.

### Network policy

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gate-net-pol
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend-apigate
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend-userdata
    ports:
    - protocol: TCP
      port: 8081
  - to:
    - podSelector:
        matchLabels:
          app: backend-suggest
    ports:
    - protocol: TCP
      port: 8083

  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

```

1. **Ingress:** It strictly allows incoming TCP traffic on port 8080, but only from the ingress-nginx namespace. This is a powerful security measure, as it ensures my API gateway is only accessible through the designated entry point and not from other pods within the cluster.

2. I have explicitly allowed the API gateway to connect to my other microservices (backend-userdata on port 8081 and backend-suggest on port 8083). This prevents the API gateway from connecting to any other unintended service, which is a great way to enforce a controlled service mesh and maintain a high level of security between components.

3. By allowing traffic to the DNS service in the kube-system namespace on port 53, i am ensuring that my API gateway can correctly perform DNS lookups to find the other services it needs to communicate with.


### DB
- **used stateful-set instead of deployments for db, also used network policy for db to only allowed it to access through my userdata service.**


### Secrets
- **Using Github Secret for creating my secrets that will be used by my suggest service.**