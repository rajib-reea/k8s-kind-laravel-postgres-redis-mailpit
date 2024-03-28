```
Prior Considerations:
 1. The purpose of this repo is learning kubernetes.
 2. We have used Kind for making Kubernetes cluster.
 3. We have used the name of our app is auth-app. You need to change the app name.
 4. Put the .kube folder in the root dir of your laravel app.
 5. The commands assume you are running them from the laravel_root_folder
 6. chnage pv files accroding to your path of directrories.
  A. postgres:
     cd laravel_root_folder/.kube
     mkdir -p .volume/pg
     cd .volume/pg
     pwd
    set this value in auth-app-pg-pv.yml file with hostPath variable.

  B. redis:
     cd laravel_root_folder/.kube
     mkdir -p .volume/rs
     cd .volume/rs
     pwd
    set this value in auth-app-rs-pv.yml file with hostPath variable.
  B. mailpit:
     cd laravel_root_folder/.kube
     mkdir -p .volume/mp
     cd .volume/mp
     pwd
    set this value in auth-app-mp-pv.yml file with hostPath variable.
```

```
Commands for Creating and Running Cluster:
0. create docker image:
docker build -t auth-app . -f .kube/auth-app-docker.yml
docker run --rm -it --name auth-app auth-app bash
[[[
cd /usr/share/nginx/html/acc-auth-app
ls -lsa
exit
]]]

1. create cluster:
kind create cluster --name auth-app-cluster --config .kube/auth-app-cluster.yml

2. load local image:
kind load docker-image auth-app:latest -n auth-app-cluster

3. create metallb:
kubectl apply -f .kube/metallb-native.yml
kubectl get pods --namespace metallb-system -o wide
kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=300s

4. config metallb:
kubectl apply -f .kube/metallb-config.yml

5. create nginx-ingress:
kubectl apply -f .kube/ingress-nginx.yml
kubectl get pods --namespace ingress-nginx -o wide
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s

6. create local deployment with service creation:
#kubectl apply -f .kube/auth-app-configmap.yml
#kubectl apply -f .kube/auth-app-secret.yml
kubectl apply -f .kube/auth-app-deploy.yml
kubectl get configmaps -o wide
kubectl get secrets -o wide
kubectl get services -o wide
[[[
kubectl exec auth-app-deployment-full-name -- env
kubectl exec -it auth-app-deployment-full-name -- /bin/bash
php artisan --version
]]]

7. create ingress:
kubectl apply -f .kube/auth-app-ingress.yml
kubectl get pods -o wide
kubectl get services -o wide

[[[test the implementation:
# should output laravel related things
curl -i http://localhost/auth-app
]]]]

```

```
8. [postgresql pod/service]

kubectl apply -f .kube/auth-app-pg-secret.yml
kubectl apply -f .kube/auth-app-pg-pv.yml
kubectl apply -f .kube/auth-app-pg-pvc.yml
kubectl apply -f .kube/auth-app-pg-deploy.yml
kubectl get configmap
kubectl get pv -o wide
kubectl get pvc -o wide
kubectl get pods -o wide
kubectl get secrets -o wide
kubectl get services -o wide
kubectl wait --namespace default \
  --for=condition=ready pod \
  --selector=app=auth-app-pg \
  --timeout=300s
kubectl get services -o wide

[from here get postgres podname to connect in the next step]
kubectl exec -it auth-app-pg-deployment-full-name -- psql -h localhost -U auth_app_user --password -p 5432 auth_app_db
[pass:con-main!@!#]
if we login to auth-app-deployment using bash then
psql -h auth-app-pg-service -U auth_app_user --password -p 5432 auth_app_db
[pass:con-main!@!#]
SELECT version();
QUIT
```
```
9. [redis pod/service]

kubectl apply -f .kube/auth-app-rs-pv.yml
kubectl apply -f .kube/auth-app-rs-pvc.yml
kubectl apply -f .kube/auth-app-rs-deploy.yml
kubectl get pv -o wide
kubectl get pvc -o wide
kubectl get pods -o wide
kubectl get services -o wide
kubectl wait --namespace default \
  --for=condition=ready pod \
  --selector=app=auth-app-rs \
  --timeout=300s
kubectl get services -o wide
[from here get postgres podname to connect in the next step]
kubectl exec -it auth-app-rs-deployment-full-name redis-cli
SET server:name "redis server"
GET server:name
QUIT
[[for later study: https://appscode.com/blog/post/mounting-redis-credentials-for-securing-microservices/]]
```
```
10. [mailpit pod/service]

kubectl apply -f .kube/auth-app-mp-pv.yml
kubectl apply -f .kube/auth-app-mp-pvc.yml
kubectl apply -f .kube/auth-app-mp-deploy.yml
kubectl get pv -o wide
kubectl get pvc -o wide
kubectl get pods -o wide
kubectl get services -o wide
kubectl wait --namespace default \
  --for=condition=ready pod \
  --selector=app=auth-app-mp \
  --timeout=300s
kubectl get services -o wide
[from here get postgres podname to connect in the next step]
kubectl exec -it auth-app-mp-deployment-full-name sh
vi email.txt
[[[ write the following contents:
From: sender@example.com
To: recipient@example.com
Subject: Email Subject

This is the body of the email.
It can contain multiple lines of text.
]]]
sendmail -v -w 5 -t -oLogLevel=1 -S localhost:1025 < email.txt
we can access mailpit web ui at: http://localhost:8025/
```

```
Command for Removing a Cluster:
kind delete cluster --name auth-app-cluster
```

