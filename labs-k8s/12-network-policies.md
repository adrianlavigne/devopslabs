# Network Policies

Por defecto los pods no se encuentran aislados, eso significa que todo el tráfico **ingress** (de entrada) y **egress** (de salida) se encuentra permitido.

El tráfico ingress esta permitido pero únicamente es tráfico interno, para acceder desde el exterior es necesario crear un ingress, exponer un puerto mediante **NodePort**, ...

El tráfico egress es el tráfico saliente. Por defecto está todo el tráfico saliente permitido. Tanto a otros pods/namespaces como hacía fuera del clúster de kubernetes.

## Despliegue de pods

Vamos a desplegar un pod con utilidades de red para realizar una serie de pruebas:

```console
[kubeadmin@kubemaster network-policies]$ kubectl apply -f utils.yaml 
namespace/utils created
deployment.apps/utils created
[kubeadmin@kubemaster network-policies]$ 
```

## Comprobación del tráfico egress

Por defecto todo el tráfico egress está permitido. Realizamos una resolución DNS utilizando el pod que hemos despleado con utilidades de networking:

```console
[kubeadmin@kubemaster network-policies]$ kubectl get pods --namespace utils
NAME                     READY   STATUS    RESTARTS   AGE
utils-646c66795d-qdtg9   1/1     Running   0          30s
[kubeadmin@kubemaster network-policies]$kubectl exec -i -t utils-646c66795d-qdtg9 --namespace utils -- nslookup www.google.com
Server:		10.96.0.10
Address:	10.96.0.10#53

Non-authoritative answer:
Name:	www.google.com
Address: 142.250.201.68

[kubeadmin@kubemaster network-policies]$ kubectl get svc --namespace kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   12d
[kubeadmin@kubemaster network-policies]$  
```

Vemos que tenemos resolución de DNS y que el servidor que estamos utilizado para las resoluciones DNS es **10.96.0.10**. Este servicio es el proporcionado por **kube-dns**.

Lanzamos un ping a un servidor fuera de kubernetes:

```console
[kubeadmin@kubemaster network-policies]$ kubectl exec -i -t utils-646c66795d-qdtg9 --namespace utils -- ping -c 4 www.google.com
PING www.google.com (142.250.201.68): 56 data bytes
64 bytes from 142.250.201.68: icmp_seq=0 ttl=115 time=10.300 ms
64 bytes from 142.250.201.68: icmp_seq=1 ttl=115 time=13.961 ms
64 bytes from 142.250.201.68: icmp_seq=2 ttl=115 time=12.761 ms
64 bytes from 142.250.201.68: icmp_seq=3 ttl=115 time=10.588 ms
--- www.google.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 10.300/11.903/13.961/1.522 ms
[kubeadmin@kubemaster network-policies]$
```

Desplegamos la aplicación balaceada, obtenemos la ip interna de uno de los contenedores y hacemos ping:

```console
[kubeadmin@kubemaster network-policies]$ kubectl get pod --namespace webapp-balanced -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE                  NOMINATED NODE   READINESS GATES
webapp-balanced-6f4f8dcd99-szwdr   1/1     Running   0          14m   192.169.62.30   kubenode1.jadbp.lab   <none>           <none>
[kubeadmin@kubemaster network-policies]$ kubectl exec -i -t utils-646c66795d-qdtg9 --namespace utils -- ping -c 4 192.169.62.30
PING 192.169.62.30 (192.169.62.30): 56 data bytes
64 bytes from 192.169.62.30: icmp_seq=0 ttl=63 time=0.223 ms
64 bytes from 192.169.62.30: icmp_seq=1 ttl=63 time=0.156 ms
64 bytes from 192.169.62.30: icmp_seq=2 ttl=63 time=0.144 ms
64 bytes from 192.169.62.30: icmp_seq=3 ttl=63 time=0.183 ms
--- 192.169.62.30 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.144/0.176/0.223/0.030 ms
[kubeadmin@kubemaster network-policies]$ 
```

## Definiendo ingress Network-Policies

Por defecto todo el tráfico de entrada a un namespace se encuentra permitido. En el momento que definamos una [Network-Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) para ese namespace del tipo ingress todo el tráfico de entrada se bloquerá excepto el explicitamente indicado en la política.

Aplicamos la network policy:

```console
[kubeadmin@kubemaster network-policies]$ kubectl apply -f network-policy-ingress-deny-all.yaml 
networkpolicy.networking.k8s.io/default-deny-ingress created
[kubeadmin@kubemaster network-policies]$ kubectl get networkpolicy --namespace webapp-balanced -o yaml
apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"networking.k8s.io/v1","kind":"NetworkPolicy","metadata":{"annotations":{},"name":"default-deny-ingress","namespace":"webapp-balanced"},"spec":{"podSelector":{},"policyTypes":["Ingress"]}}
    creationTimestamp: "2021-06-20T20:13:09Z"
    generation: 1
    name: default-deny-ingress
    namespace: webapp-balanced
    resourceVersion: "96244"
    uid: abe61532-7d87-4cb8-b1bc-bd5617d7c4fb
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
[kubeadmin@kubemaster network-policies]$ 
```

Y ahora hacemos un ping a cualquiera de los contenedores del namespace donde hemos aplicado la network policy:

```console
[kubeadmin@kubemaster network-policies]$ kubectl get pod --namespace webapp-balanced -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE                  NOMINATED NODE   READINESS GATES
webapp-balanced-6f4f8dcd99-szwdr   1/1     Running   0          26m   192.169.62.30   kubenode1.jadbp.lab   <none>           <none>
[kubeadmin@kubemaster network-policies]$ kubectl exec -i -t utils-646c66795d-qdtg9 --namespace utils -- ping -c 4 192.169.62.30
PING 192.169.62.30 (192.169.62.30): 56 data bytes
--- 192.169.62.30 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss
command terminated with exit code 1
[kubeadmin@kubemaster network-policies]$ 
```