# Cas de tests — pfSense

## Test 1 — Accès GUI pfSense
Pré-conditions: VM client sur LAN avec GW 192.168.1.1
Étapes:
  1) Ouvrir https://192.168.1.1
  2) S'authentifier (admin/pfsense si pas changé)
Résultat attendu: page d'accueil GUI accessible

## Test 2 — Blocage Internet global
Pré-conditions: Règle "Block all LAN -> Internet" en position 10 (au-dessus des passes)
Étapes:
  1) Depuis le client, ouvrir google.com
  2) ping 8.8.8.8
Résultat attendu: échec d'accès; logs firewall montrent paquets blocked

## Test 3 — Blocage Heimdall dédié
Pré-conditions: Alias HEIMDALL_HOST=192.168.1.20, HEIMDALL_PORT=8080; règle en place
Étapes:
  1) Depuis le client, http://192.168.1.20:8080
Résultat attendu: connexion refusée; log bloqué

## Test 4 — Accès services internes autorisés
Pré-conditions: règle "Allow LAN to LAN"
Étapes:
  1) Depuis client, accéder GLPI (192.168.1.30:80)
Résultat attendu: accès OK

## Test 5 — Traçabilité
Étapes:
  1) pfSense → Status → System Logs → Firewall
  2) Filtrer sur IP client
Résultat attendu: entrées "Blocked" correspondant aux tests 2 et 3
