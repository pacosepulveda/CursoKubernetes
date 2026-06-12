# Integración de Kubernetes con red física usando Calico VXLAN + BGP

## 1. Objetivo del módulo

En este módulo se estudia una arquitectura avanzada de Kubernetes en la que los nodos del clúster son **servidores físicos conectados a una red leaf-spine**, Calico se utiliza como CNI, el tráfico entre pods de diferentes nodos se encapsula mediante **VXLAN**, y la red física aprende rutas hacia las redes de pods mediante **BGP**.

El objetivo no es sustituir VXLAN por BGP, sino combinar ambos mecanismos:

```text
VXLAN  → Comunicación pod-to-pod entre nodos Kubernetes
BGP    → Anuncio de rutas hacia la infraestructura de red del CPD
```

Este diseño es especialmente interesante en entornos on-premise o CPD donde existe una red física controlada, con switches leaf, spine y route reflectors BGP.

---

## 2. Escenario de referencia

Partimos de una infraestructura similar a la siguiente:

```text
                         +----------------------+
                         |      Spine / RR      |
                         |   Route Reflector    |
                         +----------+-----------+
                                    |
               +--------------------+--------------------+
               |                                         |
        +------+-------+                         +-------+------+
        |    Leaf 1    |                         |    Leaf 2    |
        +------+-------+                         +-------+------+
               |                                         |
        +------+-------+                         +-------+------+
        |  Worker 1    |                         |  Worker 2    |
        |  físico      |                         |  físico      |
        +------+-------+                         +-------+------+
               |                                         |
        Pods 10.244.1.0/26                  Pods 10.244.2.0/26
```

Cada nodo Kubernetes está conectado físicamente a uno o varios switches leaf. El spine actúa como punto central de redistribución de rutas, normalmente mediante un **Route Reflector BGP**.

Calico asigna bloques de IPs a los nodos. Por defecto, Calico IPAM suele trabajar con bloques `/26`, aunque este tamaño puede modificarse mediante el parámetro `blockSize` del IPPool.

Ejemplo:

```text
Worker 1 → bloque de pods 10.244.1.0/26
Worker 2 → bloque de pods 10.244.2.0/26
Worker 3 → bloque de pods 10.244.3.0/26
```

---

## 3. Concepto clave: VXLAN y BGP no hacen lo mismo

Es importante separar claramente las funciones de cada tecnología.

### 3.1 VXLAN

VXLAN se utiliza para encapsular tráfico entre nodos. En Calico, VXLAN permite que el tráfico de pods cruce una red física que no conoce directamente las redes internas de pods.

Flujo típico pod-to-pod entre nodos:

```text
Pod A
  |
Worker 1
  |
VXLAN sobre red física
  |
Worker 2
  |
Pod B
```

La red física solo ve tráfico entre IPs de nodos, no entre IPs de pods.

### 3.2 BGP

BGP sirve para distribuir información de routing. Calico permite configurar BGP entre nodos Calico o con infraestructura de red externa para distribuir rutas.

En este diseño, BGP se usa para que la red del CPD conozca cómo llegar a las redes de pods:

```text
Worker 1 anuncia 10.244.1.0/26
Worker 2 anuncia 10.244.2.0/26
Worker 3 anuncia 10.244.3.0/26
```

La infraestructura local aprende que, para llegar a una IP de pod concreta, debe enviar el tráfico al nodo físico correspondiente.

---

## 4. Diseño híbrido: VXLAN interno + BGP externo

El diseño que queremos estudiar es este:

```text
                Red corporativa / CPD
                         |
                   Spine / RR BGP
                         |
              +----------+----------+
              |                     |
            Leaf 1                Leaf 2
              |                     |
          Worker 1              Worker 2
              |                     |
        Pods locales          Pods locales
```

Flujos principales:

```text
Tráfico entre pods de distintos nodos:
Pod → Worker → VXLAN → Worker → Pod

Tráfico desde red corporativa hacia pods:
Cliente externo → Leaf/Spine → Worker correcto → Pod
```

La idea es que el plano de datos interno de Kubernetes siga usando VXLAN, mientras que el plano de routing del CPD conoce las rutas necesarias para llegar a los pods desde fuera del clúster.

---

## 5. Flujo de tráfico desde la red corporativa hacia un pod

Supongamos:

```text
Cliente corporativo: 172.16.10.50
Pod destino:         10.244.2.15
Nodo del pod:        Worker 2
Bloque del nodo:     10.244.2.0/26
```

Worker 2 anuncia por BGP:

```text
10.244.2.0/26 next-hop Worker2
```

El flujo sería:

```text
172.16.10.50
   |
Red corporativa
   |
Spine / RR
   |
Leaf correspondiente
   |
Worker 2
   |
Pod 10.244.2.15
```

En este caso no es necesario encapsular el tráfico entrante con VXLAN porque el tráfico llega directamente al nodo que aloja el pod.

---

## 6. Flujo de tráfico entre pods de nodos diferentes

Ahora supongamos:

```text
Pod A: 10.244.1.10 en Worker 1
Pod B: 10.244.2.15 en Worker 2
```

Con Calico en modo VXLAN, el tráfico entre pods de nodos distintos se encapsula:

```text
Pod A
  |
Worker 1
  |
VXLAN
  |
Worker 2
  |
Pod B
```

La red física ve tráfico entre las IPs de Worker 1 y Worker 2, no entre las IPs reales de los pods.

Esto permite que Kubernetes funcione aunque la red física no tenga rutas internas para todos los Pod CIDRs.

---

## 7. ¿Qué rutas conviene anunciar?

Hay varias opciones. La elección depende del nivel de integración deseado entre Kubernetes y la red corporativa.

### 7.1 Opción 1: anunciar bloques de pods por nodo

Es la opción más precisa si se quiere acceso directo desde el CPD a las IPs de los pods.

Ejemplo:

```text
Worker 1 → anuncia 10.244.1.0/26
Worker 2 → anuncia 10.244.2.0/26
Worker 3 → anuncia 10.244.3.0/26
```

Ventajas:

- El tráfico externo llega directamente al nodo correcto.
- No se depende de un nodo intermedio.
- El routing es explícito y predecible.
- Encaja bien en una arquitectura leaf-spine con RR.

Inconvenientes:

- La red física aprende más rutas.
- Requiere control de filtros BGP.
- Requiere coordinación con el equipo de red.
- Puede exponer demasiado el direccionamiento interno de Kubernetes.

Esta opción tiene sentido cuando los pods deben ser alcanzables directamente desde otras redes del CPD.

### 7.2 Opción 2: anunciar un agregado del Pod CIDR

Por ejemplo:

```text
10.244.0.0/16
```

Esto reduce el número de rutas, pero puede provocar caminos menos óptimos.

Si varios nodos anuncian el mismo agregado, la red física puede enviar el tráfico a un nodo que no aloja el pod destino. Ese nodo tendría que reenviar internamente hacia el nodo correcto, posiblemente usando VXLAN.

```text
Cliente externo
   |
Red del CPD
   |
Worker equivocado
   |
VXLAN
   |
Worker correcto
   |
Pod destino
```

Ventajas:

- Menos rutas en la red física.
- Configuración aparentemente más sencilla.

Inconvenientes:

- Tráfico menos predecible.
- Posible forwarding interno adicional.
- Mayor dificultad de troubleshooting.
- Riesgo de rutas ambiguas si no se controla bien BGP.

Para un entorno avanzado y bien diseñado, no suele ser la opción preferida salvo que exista una razón clara.

### 7.3 Opción 3: anunciar servicios en lugar de pods

En muchos casos, lo más limpio no es anunciar las redes de pods, sino anunciar las IPs de servicios Kubernetes o rangos de LoadBalancer.

Ejemplo:

```text
Service IP: 10.96.20.50

Red corporativa
   |
BGP
   |
Nodo Kubernetes
   |
Service
   |
Pod backend
```

Ventajas:

- No se expone todo el direccionamiento de pods.
- Se publica solo lo que realmente debe consumirse.
- Es más controlado desde el punto de vista de seguridad.
- Encaja mejor con aplicaciones productivas.

Inconvenientes:

- No da acceso directo a cualquier pod.
- Requiere diseñar correctamente los servicios.
- Puede requerir integración adicional con Ingress, Gateway API o LoadBalancer.

Para la mayoría de entornos productivos, esta suele ser la opción más recomendable.

---

## 8. Recomendación de diseño

Para un curso avanzado, conviene presentar varios niveles de madurez.

### Nivel 1: VXLAN puro

```text
- Calico usa VXLAN.
- La red física no conoce los Pod CIDRs.
- El acceso externo se hace mediante NodePort, Ingress, Gateway API o LoadBalancer.
```

Es el diseño más habitual cuando no se quiere tocar la red física.

### Nivel 2: VXLAN + BGP para servicios

```text
- Calico mantiene VXLAN para tráfico interno.
- BGP anuncia IPs de servicios o rangos LoadBalancer.
- La red corporativa consume aplicaciones, no pods individuales.
```

Es un diseño equilibrado para producción.

### Nivel 3: VXLAN + BGP para redes de pods

```text
- Calico mantiene VXLAN entre nodos.
- Los nodos anuncian bloques de pods al RR.
- La red corporativa puede llegar directamente a IPs de pods.
```

Es un diseño más avanzado y requiere más control operativo.

### Nivel 4: Calico sin encapsulación + BGP puro

```text
- Se elimina VXLAN.
- La red física enruta directamente los Pod CIDRs.
- Los nodos anuncian sus bloques de pods por BGP.
```

Este diseño puede ser muy limpio en CPDs donde el equipo de red controla completamente el underlay, pero exige que la red física enrute correctamente las redes de pods.

---

## 9. Componentes de Calico implicados

### 9.1 IPPool

Define los rangos de IP que Calico puede asignar a los pods.

Ejemplo conceptual:

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool-vxlan
spec:
  cidr: 10.244.0.0/16
  vxlanMode: Always
  ipipMode: Never
  natOutgoing: true
  blockSize: 26
```

Elementos importantes:

| Campo | Descripción |
|---|---|
| `cidr` | Rango global de pods |
| `vxlanMode` | Modo de encapsulación VXLAN |
| `ipipMode` | Modo de encapsulación IP-in-IP |
| `natOutgoing` | SNAT para tráfico saliente |
| `blockSize` | Tamaño de los bloques asignados por nodo |

### 9.2 BGPConfiguration

Define parámetros globales de BGP en Calico.

Ejemplo conceptual:

```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 64512
```

En una arquitectura con route reflectors externos, normalmente se desactiva el full mesh entre nodos:

```yaml
nodeToNodeMeshEnabled: false
```

Esto evita que todos los nodos tengan que establecer sesiones BGP con todos los demás.

### 9.3 BGPPeer

Define vecinos BGP.

Ejemplo de peering desde los nodos Calico hacia un RR externo:

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rr-spine-1
spec:
  peerIP: 192.168.100.10
  asNumber: 65000
```

En este ejemplo:

```text
AS de Calico: 64512
AS del RR:    65000
RR:           192.168.100.10
```

También podría configurarse peering por nodo, por selector o por grupos de nodos, dependiendo del diseño.

---

## 10. Consideraciones sobre NAT

Uno de los puntos más importantes en este diseño es decidir si los pods deben salir con su IP real o con la IP del nodo.

Con `natOutgoing: true`, cuando un pod inicia tráfico hacia una red externa a los pools de Calico, el nodo puede aplicar SNAT y sustituir la IP origen del pod por la IP del nodo.

Ejemplo con NAT:

```text
Pod 10.244.2.15 → Cliente 172.16.10.50

La red externa ve:
Origen: Worker2
Destino: 172.16.10.50
```

Ejemplo sin NAT:

```text
Pod 10.244.2.15 → Cliente 172.16.10.50

La red externa ve:
Origen: 10.244.2.15
Destino: 172.16.10.50
```

Si la red corporativa ya conoce las rutas hacia los pods mediante BGP, puede tener sentido evitar NAT para ciertas redes internas. Pero esto debe diseñarse cuidadosamente.

---

## 11. Problema habitual: rutas de ida y vuelta

El tráfico debe ser simétrico o, al menos, aceptado por firewalls y sistemas intermedios.

Flujo de ida:

```text
Cliente externo → Red CPD → Worker → Pod
```

Flujo de vuelta:

```text
Pod → Worker → Red CPD → Cliente externo
```

Si la ida entra por un nodo y la vuelta sale por otro, pueden aparecer problemas con:

- Firewalls stateful.
- `rp_filter` en Linux.
- Balanceadores intermedios.
- Inspección de tráfico.
- Políticas de seguridad.
- NAT inesperado.

Por eso, antes de anunciar redes de pods hacia el CPD, hay que verificar:

```bash
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.default.rp_filter
```

En entornos con routing asimétrico, suele ser recomendable usar valores menos restrictivos o desactivar RPF donde proceda, siempre siguiendo las políticas de seguridad de la organización.

---

## 12. Consideraciones de MTU

VXLAN añade cabeceras adicionales, por lo que reduce la MTU útil disponible para los pods.

Si la red física usa MTU 1500, la MTU efectiva dentro del overlay debe ser menor.

Ejemplo conceptual:

```text
MTU física:       1500
Overhead VXLAN:   ~50 bytes
MTU pod:          ~1450
```

Síntomas de problemas de MTU:

- Conexiones que funcionan con paquetes pequeños pero fallan con paquetes grandes.
- Timeouts intermitentes.
- Problemas con TLS.
- Errores en transferencias grandes.
- Comportamiento irregular entre pods de nodos distintos.

Comprobaciones útiles:

```bash
ip link show vxlan.calico
ip link show
ping -M do -s 1400 <IP_DESTINO>
ping -M do -s 1450 <IP_DESTINO>
```

En redes leaf-spine con jumbo frames, se puede aumentar la MTU física para evitar pérdida de eficiencia, pero debe estar configurado extremo a extremo.

---

## 13. Seguridad del diseño

Anunciar rutas de pods hacia la red corporativa aumenta la integración, pero también la exposición.

Antes de hacerlo, conviene responder a estas preguntas:

- ¿Realmente la red corporativa necesita llegar directamente a los pods?
- ¿O basta con publicar servicios concretos?
- ¿Qué redes pueden acceder a los Pod CIDRs?
- ¿Qué firewalls controlan ese tráfico?
- ¿Se aplican NetworkPolicies de Kubernetes?
- ¿Se aplican políticas en el host?
- ¿Se filtran rutas BGP?
- ¿Se evita anunciar rutas no deseadas?

Un diseño seguro debería combinar:

- Filtros BGP.
- Control de prefijos anunciados.
- ACLs en leaf/spine.
- Firewalls perimetrales.
- Kubernetes NetworkPolicy.
- Observabilidad de tráfico.
- Segmentación por namespaces.
- Rangos de pods separados por entorno si es necesario.

---

## 14. Laboratorio propuesto

### 14.1 Objetivo del laboratorio

Configurar un clúster Kubernetes con Calico VXLAN y preparar la integración BGP con un route reflector externo simulado.

El laboratorio puede hacerse con máquinas virtuales, aunque el caso real esté pensado para servidores físicos.

### 14.2 Topología del laboratorio

```text
+-------------------+
| RR Linux / FRR    |
| 192.168.56.10     |
| AS 65000          |
+---------+---------+
          |
+---------+---------+
| Red underlay      |
| 192.168.56.0/24   |
+---------+---------+
          |
   +------+------+
   |             |
+--+--+       +--+--+
| k8s1|       | k8s2|
| AS  |       | AS  |
|64512|       |64512|
+-----+       +-----+
```

Componentes:

- 1 VM para RR con FRRouting.
- 2 o 3 nodos Kubernetes.
- Calico instalado como CNI.
- IPPool Calico en modo VXLAN.
- Peering BGP entre nodos y RR.

---

## 15. Fase 1: verificar el IPPool de Calico

Comprobar los pools existentes:

```bash
calicoctl get ippool -o wide
```

O con Kubernetes:

```bash
kubectl get ippools.crd.projectcalico.org -o wide
```

Ver el detalle:

```bash
kubectl get ippool default-ipv4-ippool -o yaml
```

Se busca algo similar a:

```yaml
spec:
  cidr: 10.244.0.0/16
  vxlanMode: Always
  ipipMode: Never
  natOutgoing: true
```

Interpretación:

```text
vxlanMode: Always → tráfico entre nodos encapsulado con VXLAN.
ipipMode: Never   → no se usa IP-in-IP.
natOutgoing: true → los pods pueden salir hacia redes externas usando SNAT.
```

---

## 16. Fase 2: comprobar bloques asignados a nodos

Comando:

```bash
calicoctl get blockaffinity -o wide
```

O mediante CRDs:

```bash
kubectl get blockaffinities.crd.projectcalico.org
```

Ejemplo esperado:

```text
NODE      CIDR
k8s1      10.244.1.0/26
k8s2      10.244.2.0/26
k8s3      10.244.3.0/26
```

Esto indica qué bloque de pods pertenece a cada nodo.

---

## 17. Fase 3: preparar el Route Reflector con FRR

En la VM que actuará como RR:

```bash
sudo apt update
sudo apt install -y frr frr-pythontools
```

Activar BGP en FRR:

```bash
sudo nano /etc/frr/daemons
```

Cambiar:

```text
bgpd=yes
```

Reiniciar FRR:

```bash
sudo systemctl restart frr
```

Entrar en la consola:

```bash
sudo vtysh
```

Configuración conceptual:

```text
configure terminal
router bgp 65000
 bgp router-id 192.168.56.10
 neighbor 192.168.56.11 remote-as 64512
 neighbor 192.168.56.12 remote-as 64512
 neighbor 192.168.56.11 route-reflector-client
 neighbor 192.168.56.12 route-reflector-client
end
write memory
```

Donde:

```text
192.168.56.10 → RR
192.168.56.11 → nodo k8s1
192.168.56.12 → nodo k8s2
```

---

## 18. Fase 4: configurar Calico BGP

Ejemplo de configuración global:

```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  asNumber: 64512
  nodeToNodeMeshEnabled: false
  logSeverityScreen: Info
```

Aplicar:

```bash
calicoctl apply -f bgpconfiguration.yaml
```

O, si se usan CRDs:

```bash
kubectl apply -f bgpconfiguration.yaml
```

Configurar el peering con el RR:

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rr-spine
spec:
  peerIP: 192.168.56.10
  asNumber: 65000
```

Aplicar:

```bash
calicoctl apply -f bgppeer-rr.yaml
```

---

## 19. Fase 5: comprobar sesiones BGP

En Calico:

```bash
calicoctl node status
```

Salida esperada:

```text
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+------------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+---------------+-------------------+-------+------------+-------------+
| 192.168.56.10 | global            | up    | 10:30:12   | Established |
+---------------+-------------------+-------+------------+-------------+
```

En el RR con FRR:

```bash
sudo vtysh
show bgp summary
show bgp ipv4 unicast
```

Se deberían ver rutas hacia los bloques de pods.

Ejemplo conceptual:

```text
*> 10.244.1.0/26 via 192.168.56.11
*> 10.244.2.0/26 via 192.168.56.12
```

---

## 20. Fase 6: probar conectividad desde fuera del clúster

Crear un pod de prueba:

```bash
kubectl run webtest --image=nginx --restart=Never
```

Obtener su IP:

```bash
kubectl get pod webtest -o wide
```

Ejemplo:

```text
NAME      IP             NODE
webtest   10.244.2.15    k8s2
```

Desde una máquina externa de la red del laboratorio:

```bash
ping 10.244.2.15
```

O probar HTTP si el contenedor escucha en el puerto 80:

```bash
curl http://10.244.2.15
```

Si no responde, comprobar:

```bash
ip route
traceroute 10.244.2.15
show bgp ipv4 unicast
kubectl get networkpolicy -A
iptables -L -n -v
ip route get 10.244.2.15
```

---

## 21. Comandos de troubleshooting

### 21.1 En nodos Kubernetes

Ver rutas:

```bash
ip route
```

Ver interfaz VXLAN:

```bash
ip link show vxlan.calico
```

Ver interfaces Calico:

```bash
ip addr | grep cali
```

Ver logs de Calico:

```bash
kubectl logs -n calico-system -l k8s-app=calico-node
```

O, según instalación:

```bash
kubectl logs -n kube-system -l k8s-app=calico-node
```

Ver estado BGP:

```bash
calicoctl node status
```

### 21.2 En el RR

```bash
sudo vtysh
show bgp summary
show bgp ipv4 unicast
show ip route
```

### 21.3 En la red física

Comprobaciones equivalentes en switches leaf/spine:

```text
show bgp summary
show bgp ipv4 unicast
show ip route 10.244.2.15
show ip route 10.244.2.0/26
show interfaces counters
show access-lists
```

Los comandos exactos dependerán del fabricante: Cisco, Arista, Juniper, Dell, Huawei, etc.

---

## 22. Errores habituales

### Error 1: las sesiones BGP no levantan

Posibles causas:

- AS incorrecto.
- IP del peer incorrecta.
- Firewall bloqueando TCP/179.
- Conectividad underlay rota.
- RR no configurado como route reflector.
- Calico no tiene BGP habilitado.

Comprobaciones:

```bash
nc -vz <IP_RR> 179
calicoctl node status
show bgp summary
```

### Error 2: BGP levanta, pero no aparecen rutas de pods

Posibles causas:

- Calico no está anunciando los bloques.
- El IPPool no está asociado a los nodos esperados.
- Filtros BGP en el RR.
- Route-maps o prefix-lists bloqueando prefijos.
- Problemas con block affinities.

Comprobaciones:

```bash
calicoctl get ippool -o wide
calicoctl get blockaffinity -o wide
show bgp ipv4 unicast
```

### Error 3: la red externa llega al nodo, pero no al pod

Posibles causas:

- Ruta correcta hasta el nodo, pero fallo en forwarding local.
- NetworkPolicy bloqueando.
- Firewall local en el nodo.
- `rp_filter`.
- El pod no escucha en el puerto probado.

Comprobaciones:

```bash
kubectl get pod -o wide
kubectl describe pod <pod>
kubectl get networkpolicy -A
ip route get <IP_POD>
sysctl net.ipv4.conf.all.rp_filter
```

### Error 4: el pod responde, pero la vuelta falla

Posibles causas:

- Ruta de retorno ausente.
- NAT no deseado.
- Firewall stateful.
- Asimetría.
- RPF.

Comprobaciones:

```bash
tcpdump -i any host <IP_POD>
tcpdump -i any host <IP_CLIENTE>
ip route get <IP_CLIENTE>
```

### Error 5: problemas intermitentes o solo con tráfico grande

Posible causa:

- MTU incorrecta por overhead VXLAN.

Comprobaciones:

```bash
ping -M do -s 1400 <destino>
ping -M do -s 1450 <destino>
ip link show vxlan.calico
```

---

## 23. Buenas prácticas

Para un entorno real, aplicaría estas buenas prácticas:

1. No anunciar todo el Pod CIDR si no es necesario.
2. Preferir anuncios de servicios o rangos LoadBalancer para publicación de aplicaciones.
3. Si se anuncian pods, anunciar bloques por nodo, no agregados ambiguos.
4. Usar filtros BGP en el RR.
5. Usar prefix-lists para limitar qué rangos puede anunciar Kubernetes.
6. Separar rangos de pods por entorno: desarrollo, preproducción y producción.
7. Documentar ASNs, peers, prefijos y next-hops.
8. Monitorizar sesiones BGP.
9. Validar MTU extremo a extremo.
10. Revisar NAT y rutas de retorno.
11. Aplicar NetworkPolicies restrictivas.
12. Coordinar siempre con el equipo de red.

---

## 24. Referencias recomendadas

- [Documentación oficial de Calico: configuración de BGP](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Documentación oficial de Calico: configuración de VXLAN e IP-in-IP](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip)
- [Documentación oficial de Calico: IPPool y direccionamiento](https://docs.tigera.io/calico/latest/reference/resources/ippool)
- [Documentación oficial de Calico: configuración inicial de IPPools](https://docs.tigera.io/calico/latest/networking/ipam/initial-ippool)
- [Documentación oficial de Calico: tamaño de bloques IPAM](https://docs.tigera.io/calico/latest/networking/ipam/change-block-size)
- [Documentación oficial de FRRouting](https://docs.frrouting.org/)
- [Sitio oficial de FRRouting](https://frrouting.org/)
- [Documentación oficial de Kubernetes: Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Documentación oficial de Kubernetes: Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Documentación oficial de Kubernetes: Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Documentación oficial de Kubernetes: Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Documentación oficial del proyecto Gateway API](https://gateway-api.sigs.k8s.io/)
- [Gateway API: guía de introducción](https://gateway-api.sigs.k8s.io/guides/getting-started/introduction/)
- [Gateway API: referencia de la especificación](https://gateway-api.sigs.k8s.io/reference/spec/)
