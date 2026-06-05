```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-custom-deployment
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: custom
```

This Kubernetes Service, named "my-custom-deployment," allows external access to my application on port 80. It uses a `NodePort` type, making my application reachable through any node in the cluster. The `nodePort` is set to 30008, enabling external access to my application by connecting to any cluster node on that port. The Service directs incoming traffic to pods labeled with "app: custom," which are part of my custom deployment, ensuring seamless external access to my application.