# nginx-deployment-with-efs
# 📦 Kubernetes Deployment Explained — NGINX on EKS with EFS and Security Context

This document breaks down the components of a Kubernetes `Deployment` manifest that:

- Deploys an NGINX web server
- Uses a Persistent Volume (EFS)
- Implements secure, non-root containers
- Adds resource limits, probes, and environment variables
- Utilizes a custom ServiceAccount
- Sets permissions via an init container

---

## 📄 Full Manifest (For Reference)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
  namespace: luit
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      securityContext:
        fsGroup: 101
      serviceAccountName: luitsa
      initContainers:
      - name: init-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 101:101 /mnt/efs"]
        volumeMounts:
        - name: efs-pv
          mountPath: /mnt/efs
        securityContext:
          runAsUser: 0
          runAsGroup: 0
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx:stable-perl
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          runAsUser: 101
          runAsGroup: 101
          runAsNonRoot: true
          allowPrivilegeEscalation: false
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
        env:
        - name: NGINX_HOST
          value: "nginx.example.com"
        - name: NGINX_PORT
          value: "80"
        volumeMounts:
        - name: efs-pv
          mountPath: /usr/share/nginx/html
          readOnly: true
      volumes:
      - name: efs-pv
        persistentVolumeClaim:
          claimName: efs-pvc
