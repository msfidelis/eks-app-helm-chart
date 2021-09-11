<h1 align="center">Welcome to EKS Application Helm Chart 👋</h1>
<p>
  <img alt="Version" src="https://img.shields.io/badge/version-v1-blue.svg?cacheSeconds=2592000" />
  <a href="/" target="_blank">
    <img alt="Documentation" src="https://img.shields.io/badge/documentation-yes-brightgreen.svg" />
  </a>
  <a href="/LICENSE" target="_blank">
    <img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg" />
  </a>
  <a href="https://twitter.com/fidelissauro" target="_blank">
    <img alt="Twitter: fidelissauro" src="https://img.shields.io/twitter/follow/fidelissauro.svg?style=social" />
  </a>
</p>

> Helm Chart to Grant High Availability for Any Applications Running on EKS


### 🏠 [Homepage](.)

### ✨ [Demo](/)

## Install

```sh
git clone git@github.com:msfidelis/eks-app-helm-chart.git
cd eks-app-helm-chart/
```

## Usage Simple

```sh
helm install simple chart/ -f examples/simple/values.yaml -n simple
helm update  simple chart/ -f examples/simple/values.yaml -n simple
```

## Run tests

```sh
curl simple.k8s.raj.ninja/version -i
HTTP/1.1 200 OK
content-type: application/json; charset=utf-8
date: Mon, 06 Sep 2021 23:20:58 GMT
content-length: 23
x-envoy-upstream-service-time: 3
server: istio-envoy

{"version":"simple-v1"}
```

## Features 

### Deployment with AWS Zones Anti-Affinity (Multi-AZ Distribuition)

By default Kubernetes (EKS) don't provide multi-az deployments for pods, only for nodes and control plane. 

This chart solve this using node label `failure-domain.beta.kubernetes.io/zone` to sugest to scheduler where place the pod. You can use `availability.multiaz.mode` with `soft` or `hard` value

```yaml
    spec:
      affinity:

        {{ if eq .Values.availability.multiaz.mode "soft" }}
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - {{ .Values.application.name }}
              topologyKey: failure-domain.beta.kubernetes.io/zone
            weight: 100            
        {{ end }}

        {{ if eq .Values.availability.multiaz.mode "hard" }}
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - {{ .Values.application.name }}
            topologyKey: failure-domain.beta.kubernetes.io/zone
        {{ end }}
```

This is an example with `hard` mode, trying to run 4 replicas on a 3 AZ's Cluster. This is grant a high consistency pods in available AZ's  

```sh
❯ kubectl get pods -n chip
NAME                    READY   STATUS    RESTARTS   AGE
chip-58b85cfc98-dfm22   0/2     Pending   0          77s
chip-58b85cfc98-lpgq9   2/2     Running   0          77s
chip-58b85cfc98-n8fb7   2/2     Running   0          77s
chip-58b85cfc98-nk8wd   2/2     Running   0          77s
```

This is an example on `soft` mode, this is works like a soft suggestion to scheduler. Instead `hard` mode, `soft` mode don't lock scheduler to place a pods. 

```sh
❯ kubectl get pods -n chip
NAME                   READY   STATUS    RESTARTS   AGE
chip-7df5b89bb-8kb4x   2/2     Running   0          67s
chip-7df5b89bb-9gvcz   2/2     Running   0          67s
chip-7df5b89bb-b489k   2/2     Running   0          67s
chip-7df5b89bb-dcsqp   2/2     Running   0          67s
```

### Deployment with hostname Anti-Affinity suggestion 

This chart uses node label `kubernetes.io/hostname` to sugest to scheduler where place the pod. You can use `availability.host.mode` with `soft` or `hard` value. 

The `soft` mode suggest do scheduler do dont' place a pod in a node with one replica already running. The `hard` mode prevent this. 

This example show `hard` mode running `8` replicas running on cluster with `6` nodes. 

```sh
❯ kubectl get pods -n chip -o wide
NAME                   READY   STATUS    RESTARTS   AGE
chip-789cffddb-5zmzx   0/2     Pending   0          4m41s    <none>        <none>                       <none>           <none>
chip-789cffddb-cktcf   2/2     Running   0          4m40s   10.0.87.162   ip-10-0-80-123.ec2.internal   <none>           <none>
chip-789cffddb-dhv9c   2/2     Running   0          4m41s   10.0.81.54    ip-10-0-87-5.ec2.internal     <none>           <none>
chip-789cffddb-ds92c   2/2     Running   0          4m41s   10.0.55.32    ip-10-0-56-184.ec2.internal   <none>           <none>
chip-789cffddb-k7nlw   2/2     Running   0          4m41s   10.0.66.176   ip-10-0-68-5.ec2.internal     <none>           <none>
chip-789cffddb-lmpgr   0/2     Pending   0          4m40s    <none>        <none>                       <none>           <none>
chip-789cffddb-mf5jr   2/2     Running   0          4m41s   10.0.73.74    ip-10-0-65-77.ec2.internal    <none>           <none>
chip-789cffddb-vtf5x   2/2     Running   0          4m41s   10.0.70.96    ip-10-0-66-110.ec2.internal   <none>           <none>
```

### Istio Retries by Default

You can enable and disable connections retries on application virtualservice.

```yaml
# Retry Options - Envoy
retry:
  max_attemps: 2
  timeout: 
    enabled: true
    max_timeout: 4s
  error:
    enabled: true
    errors: gateway-error,connect-failure,refused-stream,5xx
```

### Istio Circuit Breaker 

You can improve Istio Circuit Breaker for pods and services. 

```yaml
# Circuit Breaking
circuit_breaker:
  enabled: true
  max_connections: 500
  max_pending_requests: 20
  max_requests_per_connection: 20
  max_consecutive_5xx_errors: 50
  evaluation_interval: 1s
  ejection_time: 10s
  ejection_percent: 100
```

### Capacity 

```yaml 
replicas:
  desired: 2
  minimum: 2
  maximum: 6
```

### Horizontal Pod Autoscaling

Setting `autoscaling` key

```yaml
autoscaling:
  hpa:
    cpu:
      enabled: true
      averageUtilization: 80
    memory:
      enabled: true
      averageUtilization: 80
```

### Vertial Pod Autoscaling

### Healthcheck and Grace Periods

```yaml
probe:
  startup:
    enabled: true
    path: /healthcheck
    failureThreshold: 10
    periodSeconds: 10
  liveness:
    enabled: true
    path: /liveness
    failureThreshold: 10
    periodSeconds: 10
  readiness:
    enabled: true
    path: /readiness
    failureThreshold: 10
    periodSeconds: 10

```

### QoS Distribuition 

## Author

👤 **Matheus Fidelis**

* Website: msfidelis.github.io
* Twitter: [@fidelissauro](https://twitter.com/fidelissauro)
* Github: [@msfidelis](https://github.com/msfidelis)
* LinkedIn: [@msfidelis](https://linkedin.com/in/msfidelis)

## 🤝 Contributing

Contributions, issues and feature requests are welcome!<br />Feel free to check [issues page](/issues). 

## Show your support

Give a ⭐️ if this project helped you!

## 📝 License

Copyright © 2021 [Matheus Fidelis](https://github.com/msfidelis).<br />
This project is [MIT](/LICENSE) licensed.

***
_This README was generated with ❤️ by [readme-md-generator](https://github.com/kefranabg/readme-md-generator)_