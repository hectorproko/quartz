```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-deployment
  labels:
    app: custom
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom
  template:
    metadata:
      labels:
        app: custom
    spec:
      containers:
      - name: custom
        image: docker6767/image
        ports:
        - containerPort: 80
```
This Kubernetes Deployment creates and manages two replicas of pods, each running a container based on the "docker6767/image" *(name we give when we build the Dockerfile)* Docker image, and exposes port 80 within those pods. The pods are labeled with "app: custom" for easy selection and management.

