## 📌 Diagramme réseau détaillé :

```
┌──────────────────────────────────────────────────────────────────────┐
│ Réseau privé de l'Université (UM6P - Réseau Wi-Fi interne sécurisé)  │
│                  Sous-réseau : 10.44.28.0/24 (exemple)               │
│ DNS UM6P : cloud.cc.um6p.ma → 10.44.28.100 (entrée principale)       │
│ DNS wildcard (*.cloud.cc.um6p.ma) → 10.44.28.x                       │
└─────────────────────────────────┬────────────────────────────────────┘
                                  │ Interface Wi-Fi (wlan0)
                                  │ Adresse IP: 10.44.28.100 (DHCP)
                                  │
                 ┌────────────────┴───────────────────────┐
                 │          Nœud Master (Contrôleur RKE2) │
                 │ ┌───────────────────────────────────┐  │
                 │ │ MetalLB + Traefik Ingress (LB)   │  │
                 │ └──────────────────────────────────┘  │
                 │                                       │
                 │  Interfaces :                         │
                 │ ┌──────────────┐     ┌─────────────┐  │
                 │ │ wlan0        │     │ eth0        │  │
                 │ │10.44.28.100  │     │10.50.29.10  │  │
                 │ └──────────────┘     └─────────────┘  │
                 └───────────────────┬───────────────────┘
                                   │
             ┌─────────────────────┴─────────────────────────┐
             │       Réseau privé interne du Cluster         │
             │             (LAN : 10.50.29.0/24)             │
             │                                               │
  ┌──────────┴───────┐ ┌──────────┴───────┐ ┌─────────── ┐   │
  │ Nœud 2           │ │ Node 3           │ │ Node 4     |   │
  │ 10.50.29.11      │ │ 10.50.29.12      │ │ 10.50.29.13|   │
  └─────────────────┘  └──────────────────┘ └────────────┘   |
  ┌─────────────────┐  ┌──────────────────┐ ┌─────────────────┐
  │ Node 5          │  │ Node 6           │ │ Node 7          │
  │ 10.50.29.14     │  │ 10.50.29.15      │ │ 10.50.29.16     │
  └─────────────────┘  └──────────────────┘ └─────────────────┘
```

## 1. Réseau Universitaire (UM6P) :

Ce réseau est géré directement par vous.
Vous bénéficiez d'une adresse IP (example 10.44.28.100) attribuée dynamiquement par DHCP à travers l’interface Wi-Fi.

L'administration réseau de l'université fournit des entrées DNS, notamment :
cloud.cc.um6p.ma → 10.44.28.100
Entrée wildcard DNS : *.cloud.cc.um6p.ma → 10.44.28.100.
Ces entrées permettent d'accéder facilement à vos services Kubernetes via des noms DNS explicites plutôt que des adresses IP brutes.

## 2. Nœud Master Kubernetes :

C'est le nœud contrôleur principal du cluster RKE2 (aussi appelé Control-plane).
Il dispose de deux interfaces :

wlan0 (Wi-Fi) : Connectée directement au réseau UM6P, reçoit une adresse DHCP (10.44.28.100).

eth0 (LAN) : Connectée au réseau privé du cluster Kubernetes (10.50.29.10 IP statique).

Le Master node utilise MetalLB pour affecter une adresse IP LoadBalancer (celle de wlan0) à vos services Kubernetes.

Traefik fonctionne en tant que contrôleur ingress et route les requêtes entrantes aux applications déployées dans votre cluster.

## 3. LAN Privé (Cluster Kubernetes) :
Réseau interne isolé utilisé exclusivement par les nœuds Kubernetes (10.50.29.0/24).
Aucun accès direct depuis l'extérieur sauf via le Master node (point d’entrée unique).
Tous les autres nœuds (Nodes 2 à 7) utilisent des adresses IP statiques privées pour des raisons de sécurité interne.

📌 Exemple concret d’utilisation (DNS) :
Supposons que vous avez une application React exposée par Traefik :

DNS :
```
cloud.cc.um6p.ma → 10.44.28.100 (entrée principale Traefik)
```
* Une application React typique déployée dans Kubernetes accessible via Traefik :

```
reactAPP.cloud.cc.um6p.ma → 10.44.28.100
```

## Pourquoi les autres utilisateurs ne peuvent pas compromettre notre réseau facilement ?
Les adresses IP internes (10.50.29.0/24) ne sont pas routables depuis l’extérieur (UM6P Wi-Fi), rendant impossible l’accès direct.
L’utilisation du LoadBalancer via MetalLB est strictement contrôlée par votre Master node uniquement, rendant toute attaque directe sur les nœuds internes impossible depuis le réseau extérieur.
Les DNS fournis par l’université redirigent exclusivement vers une seule IP sécurisée par nos propres mécanismes (ingress, RBAC, TLS), minimisant le risque.

### 📌 Conclusion :

Cette architecture ne compromet pas la sécurité du réseau universitaire, car elle utilise :

Une zone isolée (LAN privé Kubernetes).
Une interface unique exposée avec contrôles sécurisés.
Une administration DNS par l’université vous-même, ce qui empêche les utilisateurs externes ou malveillants de détourner facilement les accès vers votre réseau ou vos services internes.