# Kubernetes con kind

## Introducción

En este guía mostramos como instalar un cluster de `Kubernetes` en la laptop usando la implementación
de `kind`, la cual corre cada componente en un contenedor en lugar de usar máquinas virtuales, originalmente
fue diseñado para probar kubernetes en sí, pero también puede ser usado para desarrollo local ó CI.

Este proyecto puede servir para comprender los conceptos, la arquitectura y adentrarnos más en lo que
son los contenedores, los pods y su relación con los microservicios.

Instalaremos una aplicación web simple llamada `whoami` la funcionalidad de despliegue de aplicaciones.

## Requisitos

Es necesario tener instalado y en ejecución docker para poder gestionar contenedores, este ejercicio lo
realizaremos en un equipo con MacOS, por lo que instalaremos la implementación `colima` para correr docker
en local, si tienes Linux puedes instalar docker usando tu manejador de paquetes favorito.

Iniciamos instalando colima y el cliente docker:

```shell
$ brew install colima docker
```

Ahora debemos iniciar colima:

```shell
$ colima start
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0000] preparing network ...                         context=vm
INFO[0000] creating and starting ...                     context=vm
INFO[0023] provisioning ...                              context=docker
INFO[0023] starting ...                                  context=docker
INFO[0028] done
```

**NOTA:** Por default colima levanta una máquina virtual con `2` vCPUs y `2` GB de RAM, si se desea modificar
esto para asignar más CPU o RAM, puedes agregar los parámetros `--cpu 4` y `--memory 4`.

Ahora instalamos los paquetes para kubernetes con `kind` y también instalamos el cliente `kubectl`.

```shell
$ brew install kind kubectl
```

Validamos la instalación de las herramientas, iniciamos con kind:

```shell
$ kind --version
kind version 0.14.0
```

Ahora veamos la versión de `kubectl`:

```shell
$ kubectl version --client=true
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.3"}
Kustomize Version: v4.5.4
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
    extraPortMappings:
```

Ahora creamos el cluster versión `1.23.4` con la configuración en el archivo `kind/cluster-multi.yml`:

```shell
$ kind create cluster --name k8scluster --image kindest/node:v1.23.4 --config=kind/cluster-multi.yml
Creating cluster "k8scluster" ...
 ✓ Ensuring node image (kindest/node:v1.23.4) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-k8scluster"
You can now use your cluster with:

kubectl cluster-info --context kind-k8scluster

Thanks for using kind! 😊
```

Listo!! Ya tenemos un cluster con un nodo de control plane y un worker, hagamos un listado de los clusters de kind:

```shell
$ kind get clusters
k8scluster
```

La salida del comando de arriba muestra un cluster llamado `k8scluster`.

Veamos que pasó a nivel contenedores docker:

```shell
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                       NAMES
3019b9fba1ef   kindest/node:v1.23.4   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:56200->6443/tcp   k8scluster-control-plane
ab6b690fdf56   kindest/node:v1.23.4   "/usr/local/bin/entr…"   About a minute ago   Up About a minute                               k8scluster-worker
```

Arriba se puede ver hay dos contenedores en ejecución asociados a los nodos del cluster.

## Validación del cluster

Además de que el proceso de instalación fue super rápido, kind ya agregó un contexto a la configuración de
`kubectl` local:

```shell
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER             AUTHINFO                                                NAMESPACE
*         kind-k8scluster            kind-k8scluster   kind-k8scluster                                         
```

Ahora mostramos la información de dicho cluster:

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:53551
CoreDNS is running at https://127.0.0.1:53551/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

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
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
healthz check passed
```

Listamos los nodos del cluster:

```shell
$ kubectl get nodes
NAME                       STATUS   ROLES                  AGE     VERSION
k8scluster-control-plane   Ready    control-plane,master   3m27s   v1.23.4
k8scluster-worker          Ready    <none>                 2m51s   v1.23.4
```

Como se puede ver tenemos un nodo que es el maestro, es decir, la capa de control, y tenemos otro que es el worker.

Listemos los pods de los servicios que están en ejecución:

```shell
$ kubectl get pods -A
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
kube-system          coredns-64897985d-njhpr                            1/1     Running   0          3m27s
kube-system          coredns-64897985d-wfzvh                            1/1     Running   0          3m27s
kube-system          etcd-k8scluster-control-plane                      1/1     Running   0          3m42s
kube-system          kindnet-d2gs6                                      1/1     Running   0          3m27s
kube-system          kindnet-ltnj2                                      1/1     Running   0          3m9s
kube-system          kube-apiserver-k8scluster-control-plane            1/1     Running   0          3m44s
kube-system          kube-controller-manager-k8scluster-control-plane   1/1     Running   0          3m42s
kube-system          kube-proxy-khc6t                                   1/1     Running   0          3m9s
kube-system          kube-proxy-l9vbf                                   1/1     Running   0          3m27s
kube-system          kube-scheduler-k8scluster-control-plane            1/1     Running   0          3m42s
local-path-storage   local-path-provisioner-5ddd94ff66-4952w            1/1     Running   0          3m27s
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
pod/whoami-6977d564f9-nrrg2   1/1     Running   0          92s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    16h
service/whoami       ClusterIP   10.96.144.246   <none>        8080/TCP   87s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/whoami   1/1     1            1           92s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/whoami-6977d564f9   1         1         1       92s
```

Como se puede ver se tiene los recursos `deployment`, el `replicaset`, los `pods` y el `service`.

## Limpieza

Para destruir el cluster ejecutamos:

```shell
$ kind delete cluster --name k8scluster
Deleting cluster "k8scluster" ...
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