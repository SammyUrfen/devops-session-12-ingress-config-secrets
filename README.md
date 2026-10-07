# Kubernetes Ingress, ConfigMaps & Secrets Homework — Session 12

Bibek Jyoti Charah — 24bcs10112 (GitHub: SammyUrfen)

I ran every command in this file on my own machine and pasted the output as it was printed. The
cluster is minikube v1.38.1 with the Docker driver, Kubernetes v1.35.1, on Fedora. The `kubectl`
client is v1.37.0. All commands ran against the minikube profile `s12`. Long outputs are cut, and
each cut is marked with `...`. Nothing else is edited. I deleted the cluster at the end.

| Task | Files |
| :--- | :--- |
| 1. ConfigMap | [configmap/configmap.yaml](configmap/configmap.yaml) |
| 2. Secret | [secret/secret.yaml](secret/secret.yaml) (dummy values only) |
| 3. Ingress | [ingress/apps.yaml](ingress/apps.yaml), [ingress/ingress.yaml](ingress/ingress.yaml) |
| 4. Ingress vs Ingress Controller | [this README](#task-4--ingress-vs-ingress-controller) |
| 5. Troubleshooting | [troubleshooting/](troubleshooting/) and [Task 5](#task-5--troubleshooting-the-trailing-newline-secret) |

The task list asks for screenshots. I pasted the terminal output instead, as text.

---

## Task 1 — ConfigMap

A ConfigMap holds non-secret settings outside the image. The same image can then run in dev,
staging and prod with different values.

My ConfigMap has three single keys and one whole file (`app.properties`). The Pod reads it in two
ways:

| Way | YAML | Result in the container |
| :--- | :--- | :--- |
| Environment variables | `envFrom.configMapRef` | Every key becomes an env var |
| Mounted volume | `volumes.configMap` + `volumeMounts` | The key `app.properties` becomes the file `/etc/app/app.properties` |

```
$ kubectl apply -f configmap/configmap.yaml
configmap/app-config created
pod/configmap-demo created

$ kubectl get configmap app-config
NAME         DATA   AGE
app-config   4      9s

$ kubectl describe configmap app-config
Name:         app-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
ENVIRONMENT:
----
staging

LOG_LEVEL:
----
DEBUG

MAX_BOOKING_DAYS:
----
30

app.properties:
----
cache.ttl.seconds=120
feature.dark_mode=true
...

$ kubectl get pod configmap-demo
NAME             READY   STATUS    RESTARTS   AGE
configmap-demo   1/1     Running   0          9s
```

Verify inside the container, first the env vars, then the mounted file:

```
$ kubectl exec configmap-demo -- sh -c 'env | grep -E "ENVIRONMENT|LOG_LEVEL|MAX_BOOKING" | sort'
ENVIRONMENT=staging
LOG_LEVEL=DEBUG
MAX_BOOKING_DAYS=30

$ kubectl exec configmap-demo -- ls /etc/app
app.properties

$ kubectl exec configmap-demo -- cat /etc/app/app.properties
cache.ttl.seconds=120
feature.dark_mode=true
```

`envFrom` takes every key, so the file key also became an env var with a dot in its name:

```
$ kubectl exec configmap-demo -- env | grep -i -A1 propert
app.properties=cache.ttl.seconds=120
feature.dark_mode=true
```

That is harmless here, but in a real project I would keep file keys and env keys in separate
ConfigMaps. Env vars are read once, when the container starts. A mounted file is updated by the
kubelet after a short delay when the ConfigMap changes.

---

## Task 2 — Secret

A Secret holds sensitive values. My Secret uses dummy values only: `demo-user` and
`demo-password-not-real`. The values in `data:` are base64, made with `echo -n '<value>' | base64`.
The Pod reads one key per env var (`secretKeyRef`) and also mounts the whole Secret as files.

```
$ kubectl apply -f secret/secret.yaml
secret/db-secret created
pod/secret-demo created

$ kubectl get secret db-secret
NAME        TYPE     DATA   AGE
db-secret   Opaque   2      3s

$ kubectl describe secret db-secret
Name:         db-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
DB_PASSWORD:  22 bytes
DB_USER:      9 bytes
```

`describe` shows only the size. Inside the container the value is plain text:

```
$ kubectl exec secret-demo -- sh -c 'echo DB_USER=$DB_USER; echo DB_PASSWORD=$DB_PASSWORD'
DB_USER=demo-user
DB_PASSWORD=demo-password-not-real

$ kubectl exec secret-demo -- ls /etc/creds
DB_PASSWORD
DB_USER

$ kubectl exec secret-demo -- cat /etc/creds/DB_PASSWORD
demo-password-not-real
```

### Base64 is not encryption

Anybody who can read the Secret object gets the value back with one command. No key is needed.

```
$ kubectl get secret db-secret -o yaml
apiVersion: v1
data:
  DB_PASSWORD: ZGVtby1wYXNzd29yZC1ub3QtcmVhbA==
  DB_USER: ZGVtby11c2Vy
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"DB_PASSWORD":"ZGVtby1wYXNzd29yZC1ub3QtcmVhbA==","DB_USER":"ZGVtby11c2Vy"},"kind":"Secret","metadata":{"annotations":{},"name":"db-secret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2026-10-07T14:52:26Z"
  name: db-secret
  namespace: default
  resourceVersion: "878"
  uid: 4a37e33b-f492-4c9b-8479-48dc3e0fcb09
type: Opaque

$ kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode; echo
demo-password-not-real

$ kubectl get secret db-secret -o jsonpath='{.data.DB_USER}' | base64 -d; echo
demo-user
```

Also note the `last-applied-configuration` annotation. `kubectl apply` copies the whole Secret into
it, so the value is stored a second time in the object.

| | Encoding (base64) | Encryption |
| :--- | :--- | :--- |
| Needs a key to reverse | No | Yes |
| Purpose | Store any bytes as safe text | Keep data secret |
| Who can read the value | Anyone with the text | Only the key holder |

What protects a Secret in the cluster is RBAC (who can `get` Secrets) and, if it is turned on,
encryption at rest in etcd. Base64 adds no protection.

### Why a Secret must not go to Git

- A Secret YAML in Git is the password in plain text. Base64 is decoded in one command.
- Git keeps history. Deleting the file in a later commit does not remove it from old commits.
  Every clone and fork keeps a copy.
- Bots scan public repos for keys all the time. A leaked key must be rotated, and that takes work.
- Everyone with read access to the repo gets the password, even people who have no access to the
  cluster.

Safer ways to keep Secrets with GitOps:

| Tool | What goes into Git | How the real Secret is made |
| :--- | :--- | :--- |
| Sealed Secrets (Bitnami) | A `SealedSecret`, encrypted with the controller's public key | The controller in the cluster decrypts it into a normal Secret |
| External Secrets Operator | An `ExternalSecret` that only names the key | The operator reads the value from Vault, AWS Secrets Manager, GCP Secret Manager and so on |
| SOPS (with age, PGP or a cloud KMS) | The YAML file with the values encrypted | Decrypted at deploy time (for example by Flux, or the helm-secrets plugin) |

In this repo the Secret has fake values on purpose, so it is safe to commit. A `.gitignore` is
also in place.

---

## Task 3 — Ingress

Two apps run behind one Ingress. Each app prints a fixed line, so the curl output shows which app
answered. Both Services are `ClusterIP`, so the Ingress is the only way in from outside.

```
$ minikube addons enable ingress -p s12
...
* Verifying ingress addon...
* The 'ingress' addon is enabled
```

The rules in [ingress/ingress.yaml](ingress/ingress.yaml):

| Host | Path | Service |
| :--- | :--- | :--- |
| `shop.local` | `/api` (Prefix) | `api-svc` |
| `shop.local` | `/` (Prefix) | `web-svc` |
| `api.shop.local` | `/` (Prefix) | `api-svc` |

Deploy the apps, the Services and the Ingress:

```
$ kubectl apply -f ingress/apps.yaml
deployment.apps/web created
deployment.apps/api created
service/web-svc created
service/api-svc created

$ kubectl apply -f ingress/ingress.yaml
ingress.networking.k8s.io/demo-ingress created

$ kubectl get pods,svc -l 'app in (web,api)'
NAME                       READY   STATUS    RESTARTS   AGE
pod/api-dc457ff68-rj6pm    1/1     Running   0          16s
pod/web-6d8db86c96-zhzzj   1/1     Running   0          16s

$ kubectl get svc web-svc api-svc
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.103.168.179   <none>        80/TCP    62s
api-svc   ClusterIP   10.98.101.166    <none>        80/TCP    62s

$ kubectl get endpointslices -l 'kubernetes.io/service-name in (web-svc,api-svc)'
NAME            ADDRESSTYPE   PORTS   ENDPOINTS    AGE
api-svc-pstrv   IPv4          5678    10.244.0.9   17s
web-svc-pf484   IPv4          5678    10.244.0.8   17s

$ kubectl get ingress demo-ingress
NAME           CLASS   HOSTS                       ADDRESS        PORTS   AGE
demo-ingress   nginx   shop.local,api.shop.local   192.168.58.2   80      41s

$ kubectl describe ingress demo-ingress
Name:             demo-ingress
Labels:           <none>
Namespace:        default
Address:          192.168.58.2
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host            Path  Backends
  ----            ----  --------
  shop.local      
                  /api   api-svc:80 (10.244.0.9:5678)
                  /      web-svc:80 (10.244.0.8:5678)
  api.shop.local  
                  /   api-svc:80 (10.244.0.9:5678)
Annotations:      <none>
Events:
  Type    Reason  Age               From                      Message
  ----    ------  ----              ----                      -------
  Normal  Sync    0s (x2 over 41s)  nginx-ingress-controller  Scheduled for sync
```

### Access through the Ingress and verify routing

I call the minikube node IP directly from the host. The `Host:` header picks the host rule, so no
`/etc/hosts` edit and no sudo are needed. Port 80 works because the minikube ingress controller
listens on `hostPort: 80` of the node.

```
$ minikube ip -p s12
192.168.58.2

$ curl -s -H 'Host: shop.local' http://192.168.58.2/
hello from WEB

$ curl -s -H 'Host: shop.local' http://192.168.58.2/api
hello from API

$ curl -s -H 'Host: shop.local' http://192.168.58.2/api/orders
hello from API

$ curl -s -H 'Host: shop.local' http://192.168.58.2/about
hello from WEB

$ curl -s -H 'Host: api.shop.local' http://192.168.58.2/
hello from API

$ curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: unknown.local' http://192.168.58.2/
404

$ curl -s -o /dev/null -w '%{http_code}\n' http://192.168.58.2/
404
```

| Request | Matched rule | Answer |
| :--- | :--- | :--- |
| `shop.local/` | `shop.local` + `/` | web |
| `shop.local/api`, `/api/orders` | `shop.local` + `/api` (longest prefix wins) | api |
| `shop.local/about` | `shop.local` + `/` | web |
| `api.shop.local/` | `api.shop.local` + `/` | api |
| unknown host, or no host | no rule | 404 from the controller's default backend |

The controller access log confirms which Service got each request (`[default-web-svc-80]` or
`[default-api-svc-80]`) and which Pod IP answered:

```
$ kubectl -n ingress-nginx logs pod/ingress-nginx-controller-596f8778bc-tmtff --tail=5
192.168.58.1 - - [07/Oct/2026:14:53:34 +0000] "GET / HTTP/1.1" 200 15 "-" "curl/8.18.0" 74 0.002 [default-web-svc-80] [] 10.244.0.8:5678 15 0.002 200 0d756ea5fd80c1e945e8df73404c438c
192.168.58.1 - - [07/Oct/2026:14:53:34 +0000] "GET /api HTTP/1.1" 200 15 "-" "curl/8.18.0" 77 0.001 [default-api-svc-80] [] 10.244.0.9:5678 15 0.001 200 257ab81d96bb9f1bc53b14c3d3b0e131
192.168.58.1 - - [07/Oct/2026:14:53:34 +0000] "GET /api/orders HTTP/1.1" 200 15 "-" "curl/8.18.0" 84 0.002 [default-api-svc-80] [] 10.244.0.9:5678 15 0.001 200 dedd68a7eeb464977a046adb2b56a947
192.168.58.1 - - [07/Oct/2026:14:53:34 +0000] "GET /about HTTP/1.1" 200 15 "-" "curl/8.18.0" 79 0.002 [default-web-svc-80] [] 10.244.0.8:5678 15 0.002 200 6842646a566e9fbabd9d980e6d8ea39b
192.168.58.1 - - [07/Oct/2026:14:53:34 +0000] "GET / HTTP/1.1" 200 15 "-" "curl/8.18.0" 78 0.001 [default-api-svc-80] [] 10.244.0.9:5678 15 0.001 200 3cd6af2ee45e22c57d01f3b45a16974b
```

---

## Task 4 — Ingress vs Ingress Controller

**Short version: the Ingress is the rule sheet. The Ingress Controller is the proxy that reads the
sheet and moves the traffic.**

### What is an Ingress?

An Ingress is a Kubernetes API object (`networking.k8s.io/v1`, kind `Ingress`). It describes HTTP
routing: which host and which path go to which Service and port. It can also name a TLS Secret for
HTTPS. It is only data, stored in etcd. It does not open a port and it does not run any process.
My `demo-ingress` above is an example.

### What is an Ingress Controller?

An Ingress Controller is a program that runs in the cluster, usually as a Deployment. It watches
Ingress objects through the API server and turns them into configuration for a real reverse proxy
(NGINX, Envoy, HAProxy, Traefik). Then that proxy receives the traffic and forwards it to the Pods.
Kubernetes does not ship with one. You install it, for example with `minikube addons enable ingress`.

The controller in my cluster:

```
$ kubectl -n ingress-nginx get pods,svc
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-bxjw5        0/1     Completed   3          6m49s
pod/ingress-nginx-admission-patch-67pwc         0/1     Completed   4          6m49s
pod/ingress-nginx-controller-596f8778bc-tmtff   1/1     Running     0          6m49s

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.96.245.149   <none>        80:30619/TCP,443:30932/TCP   6m50s
service/ingress-nginx-controller-admission   ClusterIP   10.106.27.99    <none>        443/TCP                      6m49s

$ kubectl -n ingress-nginx get deploy ingress-nginx-controller -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
registry.k8s.io/ingress-nginx/controller:v1.14.3@sha256:82917be97c0939f6ada1717bb39aa7e66c229d6cfb10dcfc8f1bd42f9efe0f81
```

The IngressClass links an Ingress to a controller. My Ingress says `ingressClassName: nginx`, and
the class `nginx` belongs to the controller `k8s.io/ingress-nginx`:

```
$ kubectl get ingressclass
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       6m48s
```

The controller wrote my two host rules into its own `nginx.conf`:

```
$ kubectl -n ingress-nginx exec pod/ingress-nginx-controller-596f8778bc-tmtff -- grep -n 'server_name .*shop' /etc/nginx/nginx.conf
337:		server_name "api.shop.local" ;
452:		server_name "shop.local" ;
```

### Difference between them

| | Ingress | Ingress Controller |
| :--- | :--- | :--- |
| What it is | An API object (YAML) | A running program (Pods) |
| Where it lives | etcd, through the API server | A Deployment, here in namespace `ingress-nginx` |
| Who writes it | The app team, one per app or per host | The platform team installs it once per cluster |
| What it does | Declares host/path → Service rules | Watches the rules, configures a proxy, serves the traffic |
| Built into Kubernetes | Yes, the API type is built in | No, you must install one |
| Handles packets | No | Yes |
| Examples | `demo-ingress` above, `yatri-ingress` in the course | ingress-nginx, Traefik, HAProxy Ingress, Contour, AWS Load Balancer Controller, GKE Ingress |

### Why both are required

Each one is useless alone:

- **Ingress without a controller:** the object is saved, but nothing reads it. It gets no ADDRESS,
  and no traffic goes to the app. I tested this with a class that has no controller:

  ```
  $ kubectl create ingress orphan --class=no-such-controller --rule='orphan.local/*=web-svc:80'
  ingress.networking.k8s.io/orphan created

  $ kubectl get ingress orphan
  NAME     CLASS                HOSTS          ADDRESS   PORTS   AGE
  orphan   no-such-controller   orphan.local             80      30s

  $ curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: orphan.local' http://192.168.58.2/
  404

  $ kubectl delete ingress orphan
  ingress.networking.k8s.io "orphan" deleted from default namespace
  ```

  The ADDRESS stays empty after 30 seconds, and the request gets a 404, not `hello from WEB`.
  Compare `demo-ingress`, which got the address `192.168.58.2` from the nginx controller.
- **Controller without an Ingress:** the proxy runs, but it has no rules. Every request goes to the
  default backend and gets a 404. That is what the `unknown.local` request in Task 3 shows.

This split is on purpose. It is the same as a Deployment (the wish) and the Deployment controller
(the worker). Teams write portable rules, and each cluster picks the proxy that fits it.

### Examples

| Situation | Ingress (the rules) | Controller (the engine) |
| :--- | :--- | :--- |
| This homework, minikube | `demo-ingress`: `shop.local/api` → `api-svc`, `/` → `web-svc` | ingress-nginx addon, `hostPort` 80 on the node |
| A shop on AWS EKS | `shop.com` → web, `api.shop.com` → API, TLS Secret for HTTPS | AWS Load Balancer Controller, which makes one ALB |
| A shop on bare metal | The same Ingress YAML | ingress-nginx or Traefik behind MetalLB |

The same Ingress YAML can move between those clusters. Only `ingressClassName` and the
controller-specific annotations change.

---

## Task 5 — Troubleshooting: the trailing newline Secret

The course `troubleshooting/` folder has one scenario: `secret-base64-gotcha.md`. A database
rejects the app's password, but the developer says the password is correct. The course file only
describes the bug, so I built a real reproduction in [troubleshooting/](troubleshooting/):

| File | Role |
| :--- | :--- |
| [postgres.yaml](troubleshooting/postgres.yaml) | PostgreSQL 16 Pod + Service. The server password is set with `stringData`, so it is correct. |
| [client-broken.yaml](troubleshooting/client-broken.yaml) | App Secret encoded with `echo` (no `-n`), and a `psql` client Pod that uses it |
| [client-fixed.yaml](troubleshooting/client-fixed.yaml) | The same, encoded with `echo -n` |

### 1. Apply the broken scenario

```
$ kubectl apply -f troubleshooting/postgres.yaml
secret/pg-server-secret created
pod/pg created
service/pg created

$ kubectl logs pg | tail -2
2026-10-07 14:55:00.508 UTC [57] LOG:  database system was shut down at 2026-10-07 14:55:00 UTC
2026-10-07 14:55:00.520 UTC [1] LOG:  database system is ready to accept connections

$ kubectl apply -f troubleshooting/client-broken.yaml
secret/app-db-secret created
pod/db-client created
```

### 2. Identify the problem

```
$ kubectl get pod db-client
NAME        READY   STATUS   RESTARTS   AGE
db-client   0/1     Error    0          7s

$ kubectl logs db-client
psql: error: connection to server at "pg" (10.104.91.104), port 5432 failed: FATAL:  password authentication failed for user "demo-user"

$ kubectl logs pg | grep -i fatal | tail -2
2026-10-07 14:55:04.424 UTC [61] FATAL:  password authentication failed for user "demo-user"
```

The network is fine: the client reached `pg` by Service name, and the server answered. The server
rejects the password. So the problem is the value, not DNS, the Service or the Pod.

### 3. Troubleshooting commands and root cause

`describe` gives the first clue. The password `demo-password-not-real` has 22 characters, but the
Secret holds 23 bytes:

```
$ kubectl describe secret app-db-secret | tail -3
Data
====
DB_PASSWORD:  23 bytes
```

Decode both Secrets and look at every byte with `od -c`:

```
$ kubectl get secret app-db-secret -o jsonpath='{.data.DB_PASSWORD}'; echo
ZGVtby1wYXNzd29yZC1ub3QtcmVhbAo=

$ kubectl get secret app-db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d | od -c
0000000   d   e   m   o   -   p   a   s   s   w   o   r   d   -   n   o
0000020   t   -   r   e   a   l  \n
0000027

$ kubectl get secret pg-server-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d | od -c
0000000   d   e   m   o   -   p   a   s   s   w   o   r   d   -   n   o
0000020   t   -   r   e   a   l
0000026

$ kubectl get secret app-db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d | wc -c
23

$ kubectl get secret pg-server-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d | wc -c
22
```

**Root cause:** the app's password ends with `\n`. The value was encoded with `echo`, and `echo`
adds a newline. `base64` encodes that newline too. The client sends `demo-password-not-real\n`, so
PostgreSQL rejects it. The two encodings show the difference:

```
$ echo "demo-password-not-real" | base64
ZGVtby1wYXNzd29yZC1ub3QtcmVhbAo=

$ echo -n "demo-password-not-real" | base64
ZGVtby1wYXNzd29yZC1ub3QtcmVhbA==
```

### 4. Fix

Encode with `echo -n`, or skip base64 and use `stringData`. Then re-create the client Pod, because a
Pod reads env vars from a Secret only when it starts.

```
$ kubectl delete pod db-client
pod "db-client" deleted from default namespace

$ kubectl apply -f troubleshooting/client-fixed.yaml
secret/app-db-secret configured
pod/db-client created
```

### 5. After

```
$ kubectl get pod db-client
NAME        READY   STATUS      RESTARTS   AGE
db-client   0/1     Completed   0          6s

$ kubectl logs db-client
 connected 
-----------
         1
(1 row)

$ kubectl describe secret app-db-secret | tail -3
Data
====
DB_PASSWORD:  22 bytes

$ kubectl get secret app-db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d | od -c
0000000   d   e   m   o   -   p   a   s   s   w   o   r   d   -   n   o
0000020   t   -   r   e   a   l
0000026
```

| | Before | After |
| :--- | :--- | :--- |
| Secret bytes | 23 (ends with `\n`) | 22 |
| base64 | `...cmVhbAo=` | `...cmVhbA==` |
| Client Pod | `Error`, `password authentication failed` | `Completed`, `connected = 1` |

How to avoid it: use `echo -n` or `printf '%s'`, or `stringData:`, or
`kubectl create secret generic --from-literal=...`. A quick check is to compare the byte count in
`kubectl describe secret` with the real length of the value.

---

## Findings

**1. `lab.md` Part 8 gives the wrong base64 ending.** It says `echo "secretpassword" | base64` ends
in `Ao=`. It does not:

```
$ echo "secretpassword" | base64
c2VjcmV0cGFzc3dvcmQK

$ echo -n "secretpassword" | base64
c2VjcmV0cGFzc3dvcmQ=

$ echo "mypassword" | base64
bXlwYXNzd29yZAo=
```

The base64 text of a trailing newline depends on where the newline falls in a 3-byte group. For
`mypassword` (10 bytes) it ends in `Ao=`, as `secret-base64-gotcha.md` says. For `secretpassword`
(14 bytes) it ends in `K`, with no padding. So "ends in `Ao=`" is not a reliable test for the bug.
Decode the value and check the last byte, or compare the byte count, as in Task 5.

**2. kube-proxy restarted 4 times at cluster start** with "too many open files". The cause was the
host inotify limit (`fs.inotify.max_user_instances=128`) shared by several minikube clusters on the
same machine, not the homework. It became Running on its own before I tested any Service, and every
test above ran with it Running.

---

## Cleanup

```
$ kubectl delete -f troubleshooting/ -f ingress/ -f secret/ -f configmap/
secret "app-db-secret" deleted from default namespace
pod "db-client" deleted from default namespace
...
configmap "app-config" deleted from default namespace
pod "configmap-demo" deleted from default namespace
Error from server (NotFound): error when deleting "troubleshooting/client-fixed.yaml": secrets "app-db-secret" not found
Error from server (NotFound): error when deleting "troubleshooting/client-fixed.yaml": pods "db-client" not found

$ minikube delete -p s12
* Deleting "s12" in docker ...
* Deleting container "s12" ...
* Removing /home/SammyUrfen/.minikube/machines/s12 ...
* Removed all traces of the "s12" cluster.
```

The two `NotFound` errors are expected. `client-broken.yaml` and `client-fixed.yaml` name the same
Secret and Pod, so the second file finds them already deleted.
