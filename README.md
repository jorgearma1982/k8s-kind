# Kubernetes con kind

## Introducción

En este guía mostramos como instalar un cluster de `Kubernetes` en la laptop usando la implementación de `kind`,
la cual corre cada componente en un contenedor en lugar de usar máquinas virtuales, originalmente fue diseñado
para probar kubernetes en sí, pero también puede ser usado para desarrollo local o CI.

Este proyecto puede servir para comprender los conceptos, la arquitectura y adentrarnos más en lo que son los
contenedores, los pods y su relación con los micro servicios.

Instalaremos una aplicación web simple llamada `whoami` la funcionalidad de despliegue de aplicaciones.

## Requisitos

Es necesario tener instalado y en ejecución docker para poder gestionar contenedores, este ejercicio lo realizaremos
en un equipo con MacOS, por lo que instalaremos la implementación `colima` para correr docker en local, si tienes
Linux puedes instalar docker usando tu manejador de paquetes favorito.

Iniciamos instalando colima y el cliente docker:

```shell
brew install colima docker
```

Ahora debemos iniciar colima:

```shell
$ colima start
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0001] starting ...                                  context=vm
INFO[0012] provisioning ...                              context=docker
INFO[0012] starting ...                                  context=docker
INFO[0013] done
```

**NOTA:** Por default colima levanta una máquina virtual con `2` vCPUs y `2` GB de RAM, si se desea modificar esto
para asignar más CPU o RAM, puedes agregar los parámetros `--cpu 4` y `--memory 4`.

Ahora instalamos los paquetes para kubernetes con `kind` y también instalamos el cliente `kubectl`.

```shell
brew install kind kubectl
```

Validamos la instalación de las herramientas, iniciamos con kind:

```shell
$ kind --version
kind version 0.22.0
```

Ahora veamos la versión de `kubectl`:

```shell
$ kubectl version --client=true
Client Version: v1.30.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

## Instalación de cluster

Definimos la configuración del cluster con dos nodos, uno con rol de `control-plane` y otro de `worker`.

La configuración está almacenada en el archivo `kind/cluster-multi.yml`

```shell
$ cat kind/cluster-multi.yml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
```

Ahora creamos el cluster versión `1.27.10` con la configuración en el archivo `kind/cluster-multi.yml`:

```shell
$ kind create cluster --name develop --image kindest/node:v1.27.10 --config=kind/cluster-multi.yml
Creating cluster "develop" ...
 ✓ Ensuring node image (kindest/node:v1.27.10) 🖼
 ✓ Preparing nodes 📦 📦 
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-develop"
You can now use your cluster with:

kubectl cluster-info --context kind-develop

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

Listo!! Ya tenemos un cluster con un nodo de control plane y un worker, hagamos un listado de los clusters de kind:

```shell
$ kind get clusters
develop
```

La salida del comando de arriba muestra un cluster llamado `develop`.

Veamos que pasó a nivel contenedores docker:

```shell
$ docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED              STATUS              PORTS                       NAMES
0055623ca4cf   kindest/node:v1.27.10   "/usr/local/bin/entr…"   About a minute ago   Up About a minute                               develop-worker
66e29a2aa47d   kindest/node:v1.27.10   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:63455->6443/tcp   develop-control-plane
```

Arriba se puede ver hay dos contenedores en ejecución asociados a los nodos del cluster.

## Validación del cluster

Además de que el proceso de instalación fue super rápido, kind ya agregó un contexto a la configuración de
`kubectl` local:

```shell
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER             AUTHINFO                                                NAMESPACE
*         kind-develop               kind-develop        kind-develop
```

Ahora mostramos la información de dicho cluster:

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:63455
CoreDNS is running at https://127.0.0.1:63455/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Como se pude ver, el cluster está corriendo en `localhost`.

Mostremos la salud del cluster:

```shell
$ kubectl get --raw '/healthz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/storage-object-count-tracker-hook ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/start-system-namespaces-controller ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/start-kube-apiserver-identity-lease-controller ok
[+]poststarthook/start-deprecated-kube-apiserver-identity-lease-garbage-collector ok
[+]poststarthook/start-kube-apiserver-identity-lease-garbage-collector ok
[+]poststarthook/start-legacy-token-tracking-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
[+]poststarthook/apiservice-openapiv3-controller ok
[+]poststarthook/apiservice-discovery-controller ok
healthz check passed
```

Listamos los nodos del cluster:

```shell
$ kubectl get nodes
NAME                    STATUS   ROLES           AGE     VERSION
develop-control-plane   Ready    control-plane   4m9s    v1.27.10
develop-worker          Ready    <none>          3m46s   v1.27.10
```

Como se puede ver tenemos un nodo que es el maestro, es decir, la capa de control, y tenemos otro que es el worker.

Listemos los pods de los servicios que están en ejecución:

```shell
$ kubectl get pods -A
NAMESPACE            NAME                                            READY   STATUS    RESTARTS   AGE
kube-system          coredns-5d78c9869d-55jm5                        1/1     Running   0          4m28s
kube-system          coredns-5d78c9869d-rdn6r                        1/1     Running   0          4m28s
kube-system          etcd-develop-control-plane                      1/1     Running   0          4m42s
kube-system          kindnet-np2rm                                   1/1     Running   0          4m22s
kube-system          kindnet-qfgfl                                   1/1     Running   0          4m28s
kube-system          kube-apiserver-develop-control-plane            1/1     Running   0          4m42s
kube-system          kube-controller-manager-develop-control-plane   1/1     Running   0          4m42s
kube-system          kube-proxy-8kq5z                                1/1     Running   0          4m28s
kube-system          kube-proxy-j9bbl                                1/1     Running   0          4m22s
kube-system          kube-scheduler-develop-control-plane            1/1     Running   0          4m42s
local-path-storage   local-path-provisioner-5b77c697fd-wrmhp         1/1     Running   0          4m28s
```

Esto se ve bien, todos los pods están `Running` :), en su mayoría son los servicios del cluster:

* kube-apiserver
* kube-scheduler
* kube-proxy
* kube-controller-manager
* etcd
* kindnet
* coredns
* local-path-provisioner

Todo indica a que el cluster tiene todo listo para desplegar nuestras aplicaciones.

## Despliegue aplicación

Realizamos el despliegue de una aplicación:

```shell
$ kubectl apply -f whoami/1_deployment.yml
deployment.apps/whoami created
```

Creamos el service de una aplicación:

```shell
$ kubectl apply -f whoami/2_service.yml
service/whoami created
```

Esperamos unos segundos a que levanten los servicios y continuamos con la validaciones.

## Validación aplicación

Ahora validamos listando todos los recursos del namespace `default`:

```shell
$ kubectl -n default get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/whoami-77d96f75c8-hfck6   1/1     Running   0          27s

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP    6m10s
service/whoami       ClusterIP   10.96.92.62   <none>        8080/TCP   20s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/whoami   1/1     1            1           27s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/whoami-77d96f75c8   1         1         1       27s
```

Como se puede ver se tiene los recursos `deployment`, el `replicaset`, los `pods` y el `service`.

## Limpieza

Para destruir el cluster ejecutamos:

```shell
$ kind delete cluster --name develop
Deleting cluster "develop" ...
```

También podemos limpiar colima:

```shell
$ colima delete
are you sure you want to delete colima and all settings? [y/N] y
INFO[0001] deleting colima
INFO[0001] deleting ...                                  context=docker
INFO[0001] done
```

Y listo todo se ha terminado.

## Problemas conocidos

Si usas una mac m1 es probable que tengas errores al descargar las imágenes de los contenedores, si es un error
relacionado a resolución de nombres DNS, puedes probar agregando la configuración de `lima` para que no use
el dns del host y en su lugar use el de google, por ejemplo:

Creamos configuración para dns de lima:

```shell
$ vim ~/.lima/_config/override.yaml
```

Con el siguiente contenido:

```shell
useHostResolver: false
dns:
  - 8.8.8.8
```

Se recomienda que hagas un reset de colima, haciendo delete, y nuevamente start.

También puedes iniciar colima con la opción `--dns`, por ejemplo:

```shell
$ colima start --dns 8.8.8.8
```

## Comandos útiles

Listado versiones:

* kubectl version

Listado contextos:

* kubectl config get-contexts

Detalles de cluster:

* kubectl cluster-info

Gestión de nodos:

* kubectl get nodes
* kubectl describe node NODENAME

Gestión de pods:

* kubectl get pods
* kubectl describe pod PODNAME
* kubectl logs PODNAME
* kubectl delete pod PODNAME

Gestión de services:

* kubectl get services
* kubectl describe service SVCNAME
* kubectl delete service SVCNAME

Gestión de namespaces:

* kubectl get namespaces
* kubectl describe namespace NSNAME
* kubectl delete namespace NSNAME

Gestión de recursos en modo declarativo:

* kubectl apply -f YAMLFILE
* kubectl delete -f YAMLFILE

Gestión de deployments:

* kubectl get deployment
* kubectl describe deployment PODNAME
* kubectl delete deployment PODNAME

## Referencias

La siguiente es una lista de referencias externas que pueden serle de utilidad:

* [Colima - container runtimes on macOS (and Linux) with minimal setup](https://github.com/abiosoft/colima)
* [Kubernetes - Orquestación de contenedores para producción](https://kubernetes.io/es/)
* [kind - home](https://kind.sigs.k8s.io/)
* [kind - quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)
