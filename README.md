# singlenode-k8s-metallb-traefik-letsencrypt-crowdsec
Howto SingleNode vserver k8s Setup with metallb, traefik ingress, letsencrypt and crowdsec traefik bouncer


## This is a Setup of how to implement a SingleNode Kubernetes with a Traefik Ingress bound to your External IP address and the crowdsec traefik bouncer.
   This Setup also preserves the client source IP addresses in your Traefik Ingress access Log.

Requirements:
 > internet attached vserver with >= 4GB memory
 > helm to install helm charts

- Install Singlenode Kubernetes using k0s
- Install a "default" local-hostpath storage class so crowdsec can claim storage
- Install Metallb
- Install Traefik Ingress
- Create a nginx backend for your ingress
- Install crowdsec and crowdsec traefik bouncer
- Check install results



### Install k0s

```
curl -sSLf https://get.k0s.sh | sudo sh

sudo k0s install controller --single

k0s start
```

check with
```
k0s status
Version: v1.23.8+k0s.0
Process ID: 1713
Role: controller
Workloads: true
SingleNode: true
```

place an alias in your .bashrc
```
alias k='k0s kubectl'
```


### Install a "default" local-hostpath storage class

```
k apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml

k patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}
```

### Install Metallb where Traefik Ingress external loadBalancer IP will be bound

patch the kube-proxy
```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

create namespace and install Metallb
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
```

place this in your metallb-cm.yaml
```
apiVersion: v1
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - YOUR-IPV4-ADDRESS/32
kind: ConfigMap
metadata:
  annotations:
  name: config
  namespace: metallb-system
```

and apply the config map:
```
k apply -f metallb-cm.yaml
```

### Install Traefik Ingress

place this in your ingress-traefik-kind-values.yaml

```
image:
  name: traefik
  pullPolicy: IfNotPresent
logs:
  general:
    level: ERROR
  access:
    enabled: true
    format: json
service:
  type: LoadBalancer
  spec:
    externalTrafficPolicy: Local
ports:
  traefik:
    expose: false
providers:
  kubernetesCRD:
    allowCrossNamespace: true
# securityContext:
#   readOnlyRootFilesystem: False
#   runAsGroup: 0
#   runAsUser: 0
#   runAsNonRoot: false      
initContainers:
  # The "volume-permissions" init container is required if you run into permission issues.
  # Related issue: https://github.com/containous/traefik/issues/6972
  - name: volume-permissions
    image: busybox:1.31.1
    command: ["sh", "-c", "chmod -Rv 600 /data/*"]
    volumeMounts:
      - name: traefik-pv
        mountPath: "/data"      
spec:
  volumes:
    - name: traefik-pv
      persistentVolumeClaim:
        claimName: traefik-pvc          
containers:
  - name: ingress-traefik
    volumeMounts:
      - mountPath: "/"
        name: traefik-pv
certResolvers:
   letsencrypt:
#     # for challenge options cf. https://doc.traefik.io/traefik/https/acme/
     email: your-email-address.com
       #     tlsChallenge: true
     httpChallenge:
       entryPoint: "web"
#     # match the path to persistence
additionalArguments:
  - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
  - "--entrypoints.web.http.middlewares=ingress-traefik-traefik-bouncer@kubernetescrd"
  - "--entrypoints.websecure.http.middlewares=ingress-traefik-traefik-bouncer@kubernetescrd" 
```

install traefik with helm charts and our values
```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install -n ingress-traefik ingress-traefik traefik/traefik -f ./ingress-traefik-kind-values.yaml --create-namespace
```


### Create a Traefik Ingress Nginx backend

```
k create deployment nginx --image=nginx
k create service clusterip nginx --tcp=80:80
```

### Create the Ingress Router for your domain, with an automatic generated letsencrypt certificate

Place this in ingress-your-domain.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt
  labels:
  name: your-internet-domain.com
  namespace: default
spec:
  rules:
  - host: your-internet-domain.com
    http:
      paths:
      - backend:
          service:
            name: nginx
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - your-internet-domain.com 
 ```
 
 apply it
 
 ```
 k apply -f ingress-your-domain.yaml
 ```
 
 ### Now everything is prepared and we can install crowdsec and crowdsec traffic bouncer according this howto:
     
 https://www.crowdsec.net/blog/how-to-mitigate-security-threats-with-crowdsec-and-traefik
 
 just follow the instructions beginning at "Installing CrowdSec" and "Install CrowdSec Traefik bouncer"
 
 ### Check the install results:

```
k get all -n ingress-traefik
NAME                                   READY   STATUS    RESTARTS   AGE
pod/traefik-bouncer-7667cdfbc-zlwxd    1/1     Running   0          3h20m
pod/ingress-traefik-74c66fcc58-kvktg   1/1     Running   0          3h7m

NAME                              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
service/traefik-bouncer-service   ClusterIP      10.102.208.19   <none>          80/TCP                       3h20m
service/ingress-traefik           LoadBalancer   10.101.131.37   MySecretIPv4.42   80:32333/TCP,443:30153/TCP   23h

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik-bouncer   1/1     1            1           3h20m
deployment.apps/ingress-traefik   1/1     1            1           23h
```


```
k get all -n crowdsec
NAME                                 READY   STATUS    RESTARTS   AGE
pod/crowdsec-agent-d7jg9             1/1     Running   0          4h1m
pod/crowdsec-lapi-66446775fb-28bpn   1/1     Running   0          3h58m

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/crowdsec-service   ClusterIP   10.98.216.76   <none>        8080/TCP   4h1m

NAME                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/crowdsec-agent   1         1         1       1            1           <none>          4h1m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/crowdsec-lapi   1/1     1            1           4h1m
```


curl before crowdsec ban
```
curl -iL my-domain-example.com
HTTP/1.1 301 Moved Permanently
Location: https://my-domain-example.com/
Date: Tue, 05 Jul 2022 12:40:52 GMT
Content-Length: 17
Content-Type: text/plain; charset=utf-8

HTTP/2 200 
accept-ranges: bytes
content-type: text/html
date: Tue, 05 Jul 2022 12:40:52 GMT
etag: "62beb80b-ba"
last-modified: Fri, 01 Jul 2022 09:02:03 GMT
server: nginx/1.23.0
content-length: 186
```

use nikto to test crowdsec
```
./nikto.pl -host my-domain-example.com
```

curl after nikto and crowdes traefik bouncer works as expected
```
curl -iL my-domain-example.com
HTTP/1.1 403 Forbidden
Content-Length: 9
Content-Type: text/plain; charset=utf-8
Date: Tue, 05 Jul 2022 12:44:17 GMT
```
