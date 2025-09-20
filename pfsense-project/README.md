# Déploiement pfSense — Pare-feu et segmentation réseau (VirtualBox / environnement VM)

## 1. Contexte et objectifs

Dans un contexte professionnel, l’entreprise souhaitait renforcer sa **sécurité périmétrique**, segmenter les flux entre services (GLPI, Heimdall, etc.) et disposer d’une **gestion centralisée** des règles d’accès. J’ai mis en place **pfSense** en tant que pare-feu/routeur au cœur d’un environnement virtualisé, avec :
- sécurisation des flux sortants/entrants,
- **segmentation** des zones réseau,
- **contrôle d’accès granulaire** par règles,
- intégration avec des **VM métiers**,
- tests et **journalisation** pour l’audit.

## 2. Périmètre

- Une VM **pfSense** servant de passerelle entre Internet (WAN) et le réseau interne (LAN).
- Des VMs clientes (dont **Heimdall**) connectées au LAN et **forcées** à utiliser pfSense comme **passerelle par défaut**.
- Mise en place et test de règles : **blocage Internet depuis le LAN**, **blocage spécifique de Heimdall** (ex: `10.0.2.15:8080`).

## 3. Architecture logique

```
                      Internet
                         │
                 (VirtualBox NAT)
                         │  (WAN, DHCP)
                  ┌──────▼────────┐
                  │   pfSense     │
                  │  WAN: 10.0.2.x│
                  │  LAN: 192.168.1.1/24
                  └──────┬────────┘
                         │  (LAN - Internal Network)
        ┌────────────────┼─────────────────┐
        │                │                 │
   VM Client A      VM Heimdall       VM GLPI ...
  192.168.1.10      192.168.1.20      192.168.1.30
  GW 192.168.1.1    GW 192.168.1.1    GW 192.168.1.1
```

> Point clé : **tout trafic** des VMs passe par pfSense grâce au paramétrage **passerelle par défaut = 192.168.1.1**. Sans cela, les règles ne s’appliquent pas.

## 4. Pré-requis

- Hyperviseur (VirtualBox/Proxmox/VMware). Ici : **VirtualBox**.
- ISO **pfSense CE (amd64)** téléchargé depuis le site officiel.
- Hôte avec accès Internet.
- Connaissances de base TCP/IP.

## 5. Création de la VM pfSense (VirtualBox)

1. **Créer** une VM `pfSense.home.arpa`
   - Type: *BSD*
   - Version: *FreeBSD 64-bit*
   - RAM: 2 Go, CPU: 2 vCPU
   - Disque: 20 Go (dynamique)
2. **Réseau**
   - **Adapter 1 (WAN)**: *NAT* (obtiendra une IP privée 10.0.2.x via DHCP)
   - **Adapter 2 (LAN)**: *Réseau interne* (ex: `intnet`)
3. **Monter** l’ISO pfSense et **démarrer** l’installation (partitionnement auto).
4. Au premier boot, **assigner les interfaces** (console) :
   - `em0` → **WAN**
   - `em1` → **LAN**
5. Par défaut, **LAN = 192.168.1.1/24** et l’accès Web est ouvert.

## 6. Configuration initiale via l’interface Web

1. Depuis une VM du **LAN** (interface sur `intnet`), configurer l’IP (ex: `192.168.1.10/24`, GW `192.168.1.1`).
2. Accéder à **https://192.168.1.1** (certificat auto-signé).
3. Connexion par défaut : **admin / pfsense** (à changer immédiatement).
4. Assistant initial :
   - **Timezone**: Europe/Paris
   - **WAN**: DHCP (NAT VirtualBox)
   - **LAN**: 192.168.1.1/24 (DHCP Server activé si souhaité)
   - **DNS Resolver** actif par défaut
   - **Changer le mot de passe admin**

## 7. Plan d’adressage & alias

- **LAN**: 192.168.1.0/24
- **pfSense LAN**: 192.168.1.1
- **Clients**: 192.168.1.10–.200
- **Heimdall** (ex): 192.168.1.20 (alias `HEIMDALL_HOST`), port 8080 (alias `HEIMDALL_PORT`)

Dans pfSense: **Firewall → Aliases** pour déclarer `HEIMDALL_HOST = 192.168.1.20` et `HEIMDALL_PORT = 8080`.

## 8. Règles Firewall (LAN)

pfSense applique les règles **de haut en bas**. La **première** qui match est utilisée.
- Conserver en tête la **règle anti-lockout** sur LAN.
- Ensuite, créer des règles spécifiques (blocages) **au-dessus** des règles permissives.

### 8.1. Bloquer Internet pour tout le LAN
- Interface: **LAN**
- Action: **Block**
- Source: **LAN net**
- Destination: **any**
- Protocol: **any**
- Description: `Block all LAN → Internet`
- Position: **juste après** l’anti-lockout

### 8.2. Bloquer l’accès à Heimdall (ex: 10.0.2.15:8080) depuis le LAN
- Interface: **LAN**
- Action: **Block**
- Source: **LAN net**
- Destination: **HEIMDALL_HOST**
- Protocol: **TCP**
- Destination port range: **HEIMDALL_PORT**
- Description: `Block LAN → Heimdall:8080`
- Position: **au-dessus** d’éventuelles autorisations

> Remarque : selon l’IP effective de Heimdall dans votre lab (LAN vs autre segment), **adaptez** l’adresse de destination.

## 9. NAT et routage

- Mode par défaut : **NAT IPv4 automatique** sur WAN.
- Pour que les **blocages** fonctionnent, s’assurer que **toutes les VMs** ont bien **192.168.1.1** en passerelle.
- En cas de multi-NIC ou d’un autre chemin vers Internet, le trafic peut **bypasser** pfSense.

## 10. Tests fonctionnels

### 10.1. Vérification côté client
- `ip route` (Linux) / `route print` (Windows) → la **gateway** doit être `192.168.1.1`.
- Accès Internet (Google/Amazon) → **bloqué** si la règle 8.1 est en place.
- Accès à `http://HEIMDALL_HOST:8080` → **bloqué** par la règle 8.2
- Accès aux **services internes** (ex: GLPI sur le LAN) → **OK** si autorisés.

### 10.2. Journalisation
- `Status → System Logs → Firewall`: vérifier les entrées **Blocked**.
- Filtrer par **source** (client), **destination** (externe/Heimdall), **règle**.
- Optionnel : `Diagnostics → Packet Capture` pour capturer un flux.

## 11. Difficultés rencontrées & résolution

- **Règle inopérante** : le client ne passait pas par pfSense (gateway incorrecte).  
  → Forcer la passerelle `192.168.1.1`, retirer routes alternatives, vérifier l’interface attachée.
- **Ordre des règles** : une règle permissive située au-dessus **bypassait** le blocage.  
  → Remonter les règles de blocage **au-dessus**.
- **Multi-NIC VM** : trafic sortant via une autre NIC (contournement).  
  → Débrancher NIC non nécessaires ou ajuster métriques/route.

## 12. Sécurité & exploitation

- **Changer** le mot de passe admin, restreindre l’accès GUI au LAN uniquement.
- **Mises à jour** pfSense : `System → Update`.
- **Sauvegardes** : `Diagnostics → Backup/Restore` → **exporter** `config.xml` (ne pas publier).
- **NTP**, **Syslog** distant, **alarmes** possibles via *pfBlockerNG* (optionnel).

## 13. Livrables du dépôt

- `files/network-plan.yml` : plan d’adressage et segments.
- `files/pfsense-rules.csv` : règles prévues (interface, action, sources, destinations, ports, description, ordre).
- `files/virtualbox-setup.md` : paramétrage des NIC et rattachement des VMs.
- `files/test-cases.md` : cas de tests détaillés (pré-conditions, étapes, résultat attendu).
- `files/ops-runbook.md` : procédures jour 2 (sauvegarde, restauration, ajout/modification de règle, vérification logs).
- `files/diagram.txt` : schéma ASCII de l’architecture.

Ces fichiers sont inclus dans ce projet pour pouvoir **rejouer** et **prouver** la mise en place.

## 14. Résultats

- **Blocage Internet** du LAN prouvé par tests + logs.
- **Blocage spécifique** d’un service interne (Heimdall:8080) prouvé par tests + logs.
- Topologie et routage **maîtrisés** ; documentation complète livrée.

## 15. Pistes d’évolution

- VLAN de segmentation (LAN-Users, LAN-Admins, DMZ), règles inter-VLAN.
- VPN site-à-site / accès nomade.
- Portail captif pour postes invités.
- Intégration supervision (Zabbix/Prometheus) pour l’état du pare-feu et l’usage de bande passante.
