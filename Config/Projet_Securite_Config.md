# Projet Sécurité Réseau — Configuration Finale Complète

---

## Informations clés

| Variable | Valeur |
|---|---|
| OSPF key | `CmMO2026` |
| WLC admin | `Admin / CmM2026` |
| PAA SSID | `Visiteurs` / clé `CmM2026` |
| RADIUS key | `AAAkey202#` port `1645` |
| WPA2-PSK Direction | `Direction202!` |
| VPN key | `VPNkey2026` |
| SSH user | `admin / CmM2026` |

---

## SECTION 1 — Adressage et Routage

### Routeur Siege : Inter-VLAN + DHCP

```cisco
! ── Interfaces VLAN (trunk vers F1) ──────────────────────────
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.11
 encapsulation dot1Q 11
 ip address 10.198.11.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0.22
 encapsulation dot1Q 22
 ip address 10.198.22.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0.33
 encapsulation dot1Q 33 native
 ip address 10.198.33.1 255.255.255.0
 ip nat inside

! ── Interface WAN vers Ouest ──────────────────────────────────
interface Serial0/0/0
 ip address 1.198.0.1 255.255.255.252
 ip nat outside
 no shutdown

! ── Interface DMZ ─────────────────────────────────────────────
interface GigabitEthernet0/1
 ip address 60.198.0.1 255.255.255.240
 ip nat inside
 no shutdown

! ── DHCP ──────────────────────────────────────────────────────
ip dhcp excluded-address 10.198.11.1 10.198.11.10
ip dhcp excluded-address 10.198.22.1 10.198.22.10
ip dhcp excluded-address 10.198.33.1 10.198.33.10

ip dhcp pool VLAN11_EMPLOYEES
 network 10.198.11.0 255.255.255.0
 default-router 10.198.11.1
 dns-server 60.198.0.2

ip dhcp pool VLAN22_DIRECTION
 network 10.198.22.0 255.255.255.0
 default-router 10.198.22.1
 dns-server 60.198.0.2

ip dhcp pool VLAN33_IT
 network 10.198.33.0 255.255.255.0
 default-router 10.198.33.1
 dns-server 60.198.0.2
```

**Switch F1 — Trunk vers Siege :**
```cisco
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 11,22,33
 switchport trunk native vlan 33
 no shutdown
```

**Vérification :**
```cisco
show ip interface brief
show ip dhcp binding
show ip dhcp pool
show interfaces trunk
```

---

### OSPF + Authentification MD5 (CmMO2026)

```cisco
! ══════════════ ROUTEUR OUEST ══════════════
interface GigabitEthernet0/0
 ip address 8.0.0.1 255.0.0.0
 ip ospf message-digest-key 1 md5 CmMO2026
 ip ospf authentication message-digest
 no shutdown

interface GigabitEthernet0/1
 ip address 2.198.0.1 255.255.0.0
 no shutdown

interface Serial0/0/0
 ip address 1.198.0.2 255.255.255.252
 no shutdown

router ospf 1
 router-id 1.1.1.1
 network 8.0.0.0 0.255.255.255 area 0
 network 2.198.0.0 0.0.255.255 area 0
 network 1.198.0.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/1
 passive-interface Serial0/0/0
 area 0 authentication message-digest

! ══════════════ ROUTEUR EST ══════════════
interface GigabitEthernet0/1
 ip address 8.0.0.3 255.0.0.0
 ip ospf message-digest-key 1 md5 CmMO2026
 ip ospf authentication message-digest
 no shutdown

interface GigabitEthernet0/0
 ip address 4.198.0.1 255.255.255.252
 no shutdown

router ospf 1
 router-id 3.3.3.3
 network 8.0.0.0 0.255.255.255 area 0
 network 4.198.0.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
 area 0 authentication message-digest

! ══════════════ ROUTEUR NORD ══════════════
interface GigabitEthernet0/1
 ip address 8.0.0.2 255.0.0.0
 ip ospf message-digest-key 1 md5 CmMO2026
 ip ospf authentication message-digest
 no shutdown

interface GigabitEthernet0/0
 ip address 3.198.0.1 255.255.0.0
 no shutdown

router ospf 1
 router-id 2.2.2.2
 network 8.0.0.0 0.255.255.255 area 0
 network 3.198.0.0 0.0.255.255 area 0
 passive-interface GigabitEthernet0/0
 area 0 authentication message-digest

! ══════════════ ROUTEUR SUD ══════════════
interface GigabitEthernet0/0
 ip address 8.0.0.4 255.0.0.0
 ip ospf message-digest-key 1 md5 CmMO2026
 ip ospf authentication message-digest
 no shutdown

interface GigabitEthernet0/1
 ip address 5.198.0.1 255.255.0.0
 no shutdown

router ospf 1
 router-id 4.4.4.4
 network 8.0.0.0 0.255.255.255 area 0
 network 5.198.0.0 0.0.255.255 area 0
 passive-interface GigabitEthernet0/1
 area 0 authentication message-digest
```

**Vérification :**
```cisco
show ip ospf neighbor
show ip route ospf
show ip ospf interface
```
> Attendu : 3 voisins OSPF sur chaque routeur backbone (état FULL)

---

### — Route statique + NAT sur Siege (IT + Direction)

```cisco
! ── Route par défaut ──────────────────────────────────────────
ip route 0.0.0.0 0.0.0.0 1.198.0.2

! ── ACL NAT : exclure trafic VPN, autoriser IT et Direction ──
ip access-list extended NAT_AUTORISE
 deny ip 10.198.0.0 0.0.255.255 192.168.198.64 0.0.0.63
 permit ip 10.198.22.0 0.0.0.255 any
 permit ip 10.198.33.0 0.0.0.255 any

! ── NAT overload ──────────────────────────────────────────────
ip nat inside source list NAT_AUTORISE interface Serial0/0/0 overload
```

**Vérification :**
```cisco
show ip nat translations
show ip nat statistics
show ip route
```

---

### Route statique + NAT sur Filiale

```cisco
! ── Interfaces Filiale ────────────────────────────────────────
interface GigabitEthernet0/0
 ip address 4.198.0.2 255.255.255.252
 ip nat outside
 no shutdown

interface GigabitEthernet0/1
 ip address 192.168.198.65 255.255.255.192
 ip nat inside
 no shutdown

! ── Route par défaut ──────────────────────────────────────────
ip route 0.0.0.0 0.0.0.0 4.198.0.1

! ── DHCP pour le LAN Filiale ──────────────────────────────────
ip dhcp excluded-address 192.168.198.65 192.168.198.70
ip dhcp pool FILIALE_LAN
 network 192.168.198.64 255.255.255.192
 default-router 192.168.198.65
 dns-server 60.198.0.2

! ── ACL NAT : exclure trafic VPN ──────────────────────────────
ip access-list extended NAT_FILIALE
 deny ip 192.168.198.64 0.0.0.63 10.198.0.0 0.0.255.255
 permit ip 192.168.198.64 0.0.0.63 any

! ── NAT overload ──────────────────────────────────────────────
ip nat inside source list NAT_FILIALE interface GigabitEthernet0/0 overload
```

**Vérification :**
```cisco
show ip route
show ip nat translations
ping 8.0.0.3
```

---

### DNS + WEB dans la DMZ (interface graphique PT)

```
Serveur DNS (60.198.0.2) :
  Services > DNS > ON
  www.anas.ca      → 60.198.0.3
  anas.ca          → 60.198.0.3
  www.google.com   → 8.8.8.8
  google.com       → 8.8.8.8
  wlc              → 10.198.33.5
  IP Config : 60.198.0.2 / 255.255.255.240 / GW: 60.198.0.1

Serveur WEB (60.198.0.3) :
  Services > HTTP > ON
  Services > HTTPS > ON
  IP Config : 60.198.0.3 / 255.255.255.240 / GW: 60.198.0.1

Serveur 8.8.8.8 :
  Services > HTTP > ON
  Services > DNS > ON
  Ajouter : www.google.com → 8.8.8.8
```

---

## SECTION 2 — Réseaux Sans-Fil

### PAA Autonome

```
Config > Interface > Wireless :
  SSID              : Visiteurs
  Authentication    : WPA2-PSK
  PSK               : CmM2026
  Encryption        : AES
  Canal             : 6

Config > Interface > FastEthernet :
  IP                : DHCP (ou 192.198.198.71/26)
  Default Gateway   : 192.168.198.65
```

### WLC Admin

```
Depuis PC Siege : http://<IP-WLC>
  Login : admin / admin
  Management > User Management > admin
  New Password : CmM2026
```

### WLAN Direction (WPA2-PSK)

```
WLANs > Create New :
  SSID         : Direction
  Security     : WPA2 / PSK / AES
  PSK          : Direction202!
  Advanced VLAN: 22
```

### WLAN Employees (WPA2-Enterprise)

```
Security > AAA > RADIUS > Authentication > New :
  Server IP    : 10.198.33.10
  Secret       : AAAkey202#
  Port         : 1645

WLANs > Create New :
  SSID         : Employees
  Security     : WPA2 / 802.1X / AES
  AAA Server   : 10.198.33.10
  Advanced VLAN: 11

Sur Serveur Radius (10.198.33.10) > Services > AAA :
  Client IP    : <IP WLC>
  Secret       : AAAkey202#
  Users        : anas / 123456
```

**Connexion client Employees :**
```
Config > Wireless0 :
  SSID           : Employees
  Authentication : 802.1X
  Method         : MD5
  Username       : anas
  Password       : 123456
  Encryption     : AES (pas WEP !)
```

---

## SECTION 3 — Filtrage ACL

### SSH VLAN IT uniquement

```cisco
hostname Siege
ip domain-name anas.ca
crypto key generate rsa modulus 1024
ip ssh version 2
username admin privilege 15 secret CmM2026
enable secret CmM2026

ip access-list standard ACL_SSH_IT
 permit 10.198.33.0 0.0.0.255
 deny   any

line vty 0 4
 access-class ACL_SSH_IT in
 transport input ssh
 login local
 exec-timeout 5 0
```

### Anti-brute-force SSH

```cisco
login block-for 60 attempts 3 within 30

ip access-list standard WHITELIST
 permit 10.198.33.0 0.0.0.255

login quiet-mode access-class WHITELIST
login delay 3
```

### ACL VLAN IT (accès complet)

```cisco
ip access-list extended ACL_IT_FULL
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit ip 10.198.33.0 0.0.0.255 any
 deny   ip any any

interface GigabitEthernet0/0.33
 ip access-group ACL_IT_FULL in
```

### ACL VLAN Direction (WEB + DNS)

```cisco
ip access-list extended ACL_DIRECTION
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit udp 10.198.22.0 0.0.0.255 host 60.198.0.2 eq domain
 permit tcp 10.198.22.0 0.0.0.255 host 60.198.0.2 eq domain
 permit tcp 10.198.22.0 0.0.0.255 host 60.198.0.3 eq www
 permit tcp 10.198.22.0 0.0.0.255 host 60.198.0.3 eq 443
 permit udp 10.198.22.0 0.0.0.255 any eq domain
 permit tcp 10.198.22.0 0.0.0.255 any eq www
 permit tcp 10.198.22.0 0.0.0.255 any eq 443
 permit icmp 10.198.22.0 0.0.0.255 any echo
 permit icmp any 10.198.22.0 0.0.0.255 echo-reply
 deny   ip any any

interface GigabitEthernet0/0.22
 ip access-group ACL_DIRECTION in
```

### ACL Internet entrant → DMZ

```cisco
ip access-list extended ACL_INTERNET_IN
 permit tcp any host 60.198.0.3 eq www
 permit tcp any host 60.198.0.3 eq 443
 permit udp any host 60.198.0.2 eq domain
 permit tcp any host 60.198.0.2 eq domain
 permit udp any eq domain 10.198.22.0 0.0.0.255
 permit udp any eq domain 10.198.33.0 0.0.0.255
 permit tcp any eq www 10.198.22.0 0.0.0.255
 permit tcp any eq www 10.198.33.0 0.0.0.255
 permit tcp any eq 443 10.198.22.0 0.0.0.255
 permit tcp any eq 443 10.198.33.0 0.0.0.255
 permit icmp any any echo-reply
 permit icmp any any echo
 permit ip 192.168.198.64 0.0.0.63 10.198.33.0 0.0.0.255
 permit ip 192.168.198.64 0.0.0.63 10.198.22.0 0.0.0.255
 deny   ip any any

interface Serial0/0/0
 ip access-group ACL_INTERNET_IN in
 no ip access-group sl_def_acl in
 no ip access-group sl_def_acl out
```

### Résolution conflits DHCP/ACL

```cisco
! ACL_IT_FULL déjà correcte (bootps/bootpc en tête)
! Vérification :
show ip dhcp binding
show access-lists ACL_IT_FULL
show ip interface GigabitEthernet0/0.33 | include access list
```

---

## SECTION 4 — Tunnel VPN

### VPN IPsec Siege ↔ Filiale

**Prérequis sur les DEUX routeurs :**
```cisco
license boot module c1900 technology-package securityk9
! Tapez : yes
write memory
reload
```

**Sur SIEGE :**
```cisco
! ── Phase 1 ISAKMP ────────────────────────────────────────────
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key VPNkey2026 address 4.198.0.2

! ── Phase 2 IPsec ─────────────────────────────────────────────
crypto ipsec transform-set TS_VPN esp-aes esp-sha-hmac

! ── ACL trafic intéressant ────────────────────────────────────
ip access-list extended ACL_VPN_SIEGE
 permit ip 10.198.0.0 0.0.255.255 192.168.198.64 0.0.0.63

! ── Crypto Map ────────────────────────────────────────────────
crypto map MAP_VPN 10 ipsec-isakmp
 set peer 4.198.0.2
 set transform-set TS_VPN
 match address ACL_VPN_SIEGE

! ── Appliquer sur WAN ─────────────────────────────────────────
interface Serial0/0/0
 crypto map MAP_VPN

! ── Route vers Filiale ────────────────────────────────────────
ip route 192.168.198.64 255.255.255.192 1.198.0.2

! ── NAT : exclure trafic VPN (CRITIQUE) ──────────────────────
no ip access-list extended NAT_AUTORISE
ip access-list extended NAT_AUTORISE
 deny ip 10.198.0.0 0.0.255.255 192.168.198.64 0.0.0.63
 permit ip 10.198.22.0 0.0.0.255 any
 permit ip 10.198.33.0 0.0.0.255 any
```

**Sur FILIALE :**
```cisco
! ── Phase 1 ISAKMP ────────────────────────────────────────────
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key VPNkey2026 address 1.198.0.1

! ── Phase 2 IPsec ─────────────────────────────────────────────
crypto ipsec transform-set TS_VPN esp-aes esp-sha-hmac

! ── ACL trafic intéressant ────────────────────────────────────
ip access-list extended ACL_VPN_FILIALE
 permit ip 192.168.198.64 0.0.0.63 10.198.0.0 0.0.255.255

! ── Crypto Map ────────────────────────────────────────────────
crypto map MAP_VPN 10 ipsec-isakmp
 set peer 1.198.0.1
 set transform-set TS_VPN
 match address ACL_VPN_FILIALE

! ── Appliquer sur WAN ─────────────────────────────────────────
interface GigabitEthernet0/0
 crypto map MAP_VPN

! ── NAT : exclure trafic VPN (CRITIQUE) ──────────────────────
no ip access-list extended NAT_FILIALE
ip access-list extended NAT_FILIALE
 deny ip 192.168.198.64 0.0.0.63 10.198.0.0 0.0.255.255
 permit ip 192.168.198.64 0.0.0.63 any

! ── Route statique par défaut ─────────────────────────────────
ip route 0.0.0.0 0.0.0.0 4.198.0.1
```

**Sur OUEST (route vers Filiale) :**
```cisco
ip route 192.168.198.64 255.255.255.192 8.0.0.3
```

**Sur EST (route vers Siege) :**
```cisco
ip route 10.198.0.0 255.255.0.0 8.0.0.1
ip route 1.198.0.0 255.255.255.252 8.0.0.1
```

### Test Radius ↔ Smartphone

```
Depuis Serveur Radius (10.198.33.10) :
  ping <IP-smartphone>   (IP dans 192.168.198.x)

Depuis Smartphone (via PAA) :
  ping 10.198.33.10
```

**Vérification VPN :**
```cisco
show crypto isakmp sa
! Attendu : QM_IDLE ✅

show crypto ipsec sa
! Attendu : pkts encaps > 0, pkts decaps > 0
```

---

> **`write memory`** sur tous les routeurs avant de sauvegarder le `.pkt` !
