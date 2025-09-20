# Paramétrage VirtualBox — pfSense + VMs clientes

## pfSense VM
- Adapter 1 (WAN): NAT
- Adapter 2 (LAN): Réseau interne `intnet`

## VM Cliente (ex: Client-A)
- Adapter 1: Réseau interne `intnet`
- IP manuelle: 192.168.1.10/24
- Passerelle: 192.168.1.1
- DNS: 192.168.1.1 (ou public, ex 1.1.1.1)

## VM Heimdall
- Adapter 1: Réseau interne `intnet`
- IP: 192.168.1.20/24
- Service: HTTP 8080
