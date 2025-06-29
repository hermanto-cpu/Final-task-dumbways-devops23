# Challenge Kubernetes

**Instructions**

- No 5 (Deployment) app runs inside a kubernetes cluster (include frontend, backend, and database) for production Environment
- Apps running 100% with backend and db integration
- CICD integration for production
- Ingress Nginx or others.
  - **Domain**
    - <name>.studentdumbways.my.id - App
    - api.<name>.studentdumbways.my.id - Backend API
- Persistent Volume

---

1. Buat Cluster Kubernetes

```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
k3d cluster create dumbmerch \
  --agents 1 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"

```

> menjalankan Kubernetes cluster kecil dengan 1 node, dan otomatis membuka port 80 dan 443 agar bisa dipakai Ingress. 2. Buat file YAML untuk K8s

2. Deploy NGINX Ingress Controller agar bisa routing ke domain hermanto.studentdumbways.my.id dan api.hermanto.studentdumbways.my.id

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/k3s/deploy.yaml
```

3. Buat namespace `kubectl create namespace dumbmerch` atau bisa langsung nanti di bagian metadata file .yaml yang akan dibuat

```
metadata:
  namespace: dumbmerch

```

4. Siapkan YAML File

- Struktur File Kubernetes

```
k8s/
├── backend/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── frontend/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── postgres/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── pvc.yaml
```

- frontend/deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
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
      - name: fe-dumbmerch
        image: registry.hermanto.studentdumbways.my.id/fe-dumbmerch:production
        ports:
        - containerPort: 80

```

- frontend/service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP

```

- frontend/ingress.yaml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hermanto.studentdumbways.my.id
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

- (Backend dan PostgreSQL serupa, tinggal sesuaikan port dan imagenya)

5. Apply Semua YAML

```
kubectl apply -f k8s/postgres/ -n dumbmerch
kubectl apply -f k8s/backend/ -n dumbmerch
kubectl apply -f k8s/frontend/ -n dumbmerch
```

3. Buat Jenkins Pipeline Tambahan (Deploy ke K8s)

```groovy
def secret = 'finaltask-totywan-app-server'
def server = 'finaltask-totywan@103.127.137.206'
def sshPort = '1234'
def directory = '/home/finaltask-totywan/k8s/'

stage('Deploy to Kubernetes Production') {
  steps {
    sshagent(['secret']) {
      sh """
        ssh -p ${sshPort} -o StrictHostKeyChecking=no ${server} << EOF
                            kubectl apply -f ${directory} -n dumbmerch
                            echo "Sukses"
                            exit
                        EOF
      """
    }
  }
}

```
