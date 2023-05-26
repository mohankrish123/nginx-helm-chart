This repository creates the following kubernetes resources:
- `nginx` namespace
- `nginx-deployment` deployment
- `nginx-svc` service

# How to build docker image in Mac
```
docker build --platform=linux/amd64 . -t nginx:1.0.0
```


# How to deploy the helm chart
```
cd helm-chart
helm install nginx .
```

# How to upgrade
```
cd helm-chart
helm upgrade nginx .
```
