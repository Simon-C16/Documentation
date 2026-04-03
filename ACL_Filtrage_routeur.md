# Documentation — ACL & Filtrage Routeur (Cisco IOS)

## Introduction

Ce document présente les configurations d'**Access Control Lists (ACL)** appliquées sur des routeurs Cisco dans le cadre de différents scénarios de filtrage réseau. Les ACL permettent de contrôler le trafic entrant et sortant sur les interfaces d'un routeur, en autorisant ou refusant des paquets selon des critères définis (adresse IP source/destination, protocole, port, etc.).

---

## Scénario 1 — Blue : Filtrage par réseau source

**Objectif :** Autoriser uniquement les réseaux `192.192.1.0/24` et `192.192.2.0/24` en sortie sur l'interface `gi0/0`.

### Suppression de la configuration initiale bloquante

```
Router1(conf)# access-list 1 deny any
Router1(conf)# interface s0/0/0
Router1(conf-if)# ip access-group 1 out
Router1(conf-if)# exit

Router1(conf)# no access-list 1
Router1(conf)# interface s0/0/0
Router1(conf-if)# no ip access-group 1 out
Router1(conf-if)# exit
```

> ⚠️ Une ACL `deny any` bloque tout le trafic. Elle est supprimée avant de définir les règles définitives.

### Configuration finale

```
Router1(conf)# access-list 1 permit 192.192.1.0 0.0.0.255
Router1(conf)# access-list 2 permit 192.192.2.0 0.0.0.255

Router1(conf)# interface gi0/0
Router1(conf-if)# ip access-group 1 out
```

**Résumé :**

| ACL | Action  | Réseau autorisé    | Interface | Direction |
|-----|---------|--------------------|-----------|-----------|
| 1   | permit  | 192.192.1.0/24     | gi0/0     | out       |
| 2   | permit  | 192.192.2.0/24     | —         | —         |

---

## Scénario 2 — Cobalt et Ciel : Filtrage entrant/sortant sur WAN

**Objectif :** Contrôler le trafic sur l'interface série `s0/0/0` — filtrer les flux sortants vers `192.192.1.0/24` et restreindre les flux entrants (tout IP à destination de `192.192.1.0/24`) + autoriser RIP (UDP port 520).

### Nettoyage des ACL et groupes précédents

```
Router1(conf)# no access-list 1
Router1(conf)# no access-list 2
Router1(conf)# interface gi0/0
Router1(conf-if)# no ip access-group 2 out
Router1(conf-if)# exit
Router1(conf)# interface gi0/1
Router1(conf-if)# no ip access-group 1 out
Router1(conf-if)# exit
```

### Configuration

```
Router1(conf)# access-list 2 permit 192.192.1.0 0.0.0.255

Router1(conf)# access-list 102 permit ip any 192.192.1.0 0.0.0.255

Router1(conf)# interface s0/0/0
Router1(conf-if)# ip access-group 2 out
Router1(conf-if)# ip access-group 102 in
Router1(conf-if)# exit

Router1(conf)# access-list 102 permit udp any eq 520 any eq 520
```

**Résumé :**

| ACL | Type     | Action  | Source / Destination           | Interface | Direction |
|-----|----------|---------|-------------------------------|-----------|-----------|
| 2   | Standard | permit  | src: 192.192.1.0/24           | s0/0/0    | out       |
| 102 | Étendue  | permit  | any → 192.192.1.0/24          | s0/0/0    | in        |
| 102 | Étendue  | permit  | UDP port 520 ↔ 520 (RIP)      | s0/0/0    | in        |

> 📝 **RIP v1/v2** utilise UDP port 520. Cette règle autorise les mises à jour de routage RIP.

---

## Scénario 3 — Cobalt : Blocage sélectif de destinations

**Objectif :** Permettre au réseau `192.192.1.0/24` d'accéder à Internet, sauf vers les réseaux `201.10.10.0/24` et `202.20.20.0/24`.

### Configuration

```
Router1(conf)# no access-list 2

Router1(conf)# access-list 103 deny ip 192.192.1.0 0.0.0.255 201.10.10.0 0.0.0.255
Router1(conf)# access-list 103 deny ip 192.192.1.0 0.0.0.255 202.20.20.0 0.0.0.255
Router1(conf)# access-list 103 permit ip 192.192.1.0 0.0.0.255 any

Router1(conf)# interface s0/0/0
Router1(conf-if)# ip access-group 103 out
Router1(conf-if)# exit
```

**Résumé :**

| Règle | Action | Source          | Destination      |
|-------|--------|-----------------|------------------|
| 1     | deny   | 192.192.1.0/24  | 201.10.10.0/24   |
| 2     | deny   | 192.192.1.0/24  | 202.20.20.0/24   |
| 3     | permit | 192.192.1.0/24  | any              |

> 💡 L'ordre des règles est crucial : les refus spécifiques sont placés **avant** le `permit any`.

---

## Scénario 4 — Blacklister un PC vers www.emeraude.com

**Objectif :** Bloquer l'accès du PC `192.192.1.11` vers le serveur web `203.30.30.80`, tout en autorisant le reste du trafic.

### Configuration

```
Router3(conf)# access-list 104 deny ip host 192.192.1.11 host 203.30.30.80
Router3(conf)# access-list 104 permit ip any any

Router3(conf)# interface gi0/2
Router3(conf-if)# ip access-group 104 out
Router3(conf-if)# exit
```

**Résumé :**

| Règle | Action | Source          | Destination    |
|-------|--------|-----------------|----------------|
| 1     | deny   | 192.192.1.11    | 203.30.30.80   |
| 2     | permit | any             | any            |

> 🔒 Le mot-clé `host` désigne une adresse IP unique (masque implicite `0.0.0.0`).

---

## Scénario 5 — Accès restrictif aux services Orange

**Objectif :** Limiter le trafic entrant sur `s0/0/0` aux seuls services HTTP (port 80), DNS (UDP 53 vers `200.200.4.53`), RIP (UDP 520) et connexions TCP établies.

### Configuration

```
Router4(conf)# access-list 105 permit tcp any any eq www
Router4(conf)# access-list 105 permit udp any host 200.200.4.53 eq 53
Router4(conf)# access-list 105 permit udp any eq 520 any eq 520

Router4(conf)# interface s0/0/0
Router4(conf-if)# ip access-group 105 in
Router4(conf-if)# exit

Router4(conf)# access-list 105 permit tcp any any established
```

**Résumé des services autorisés :**

| Protocole | Port      | Description                     |
|-----------|-----------|---------------------------------|
| TCP       | 80 (www)  | Navigation Web (HTTP)           |
| UDP       | 53        | DNS vers 200.200.4.53 uniquement |
| UDP       | 520       | RIP (routage dynamique)         |
| TCP       | established | Réponses aux connexions TCP sortantes |

> ℹ️ Le mot-clé `established` permet les réponses TCP aux connexions initiées depuis le réseau interne (bit ACK ou RST présent).

---

## Scénario 6 — Modification de l'ACL 103 : accès email partiel vers 202.20.20.0/24

**Objectif :** Affiner l'ACL 103 pour autoriser uniquement POP3 (port 110) et SMTP (port 25) vers `202.20.20.0/24`, tout en bloquant le reste du trafic IP vers ce réseau.

### Recréation complète de l'ACL 103

```
Router1(conf)# no access-list 103
Router1(conf)# access-list 103 deny ip 192.192.1.0 0.0.0.255 201.10.10.0 0.0.0.255
Router1(conf)# access-list 103 permit tcp 192.192.1.0 0.0.0.255 202.20.20.0 0.0.0.255 eq pop3
Router1(conf)# access-list 103 permit tcp 192.192.1.0 0.0.0.255 202.20.20.0 0.0.0.255 eq smtp
Router1(conf)# access-list 103 deny ip 192.192.1.0 0.0.0.255 202.20.20.0 0.0.0.255
Router1(conf)# access-list 103 permit ip 192.192.1.0 0.0.0.255 any
```

### Insertion de règles numérotées (sans recréer l'ACL)

```
Router1(config)# ip access-list extended 103
Router1(config-ext-nacl)# 15 permit tcp 192.192.1.0 0.0.0.255 202.20.20.0 0.0.0.255 eq pop3
Router1(config-ext-nacl)# 16 permit tcp 192.192.1.0 0.0.0.255 202.20.20.0 0.0.0.255 eq smtp
Router1(config-ext-nacl)# exit
```

**Résumé de l'ACL 103 finale :**

| N° | Action | Protocole | Source         | Destination    | Port      |
|----|--------|-----------|----------------|----------------|-----------|
| —  | deny   | ip        | 192.192.1.0/24 | 201.10.10.0/24 | —         |
| 15 | permit | tcp       | 192.192.1.0/24 | 202.20.20.0/24 | 110 (POP3)|
| 16 | permit | tcp       | 192.192.1.0/24 | 202.20.20.0/24 | 25 (SMTP) |
| —  | deny   | ip        | 192.192.1.0/24 | 202.20.20.0/24 | —         |
| —  | permit | ip        | 192.192.1.0/24 | any            | —         |

> 🔢 Les **numéros de séquence** (15, 16) permettent d'insérer des règles à une position précise sans recréer l'ACL entièrement.

---

## Scénario 7 — Autoriser le ping sortant, refuser le ping entrant

**Objectif :** Bloquer les requêtes ICMP Echo entrantes vers `192.192.1.0/24` (blocage du ping depuis l'extérieur), tout en laissant passer les pings initiés depuis l'intérieur.

### Configuration

```
Router1(conf)# ip access-list extended 102
Router1(config-ext-nacl)# 5 deny icmp any 192.192.1.0 0.0.0.255 echo
Router1(config-ext-nacl)# exit
```

> 🏓 Le type ICMP `echo` correspond aux **requêtes ping** (type 8). Les **réponses echo-reply** (type 0) ne sont pas bloquées, ce qui permet aux pings sortants de recevoir une réponse.

---

## Récapitulatif des ACL configurées

| ACL | Type     | Routeur  | Interface | Direction | Objectif principal                             |
|-----|----------|----------|-----------|-----------|------------------------------------------------|
| 1   | Standard | Router1  | gi0/0     | out       | Autoriser 192.192.1.0/24 en sortie             |
| 2   | Standard | Router1  | s0/0/0    | out       | Autoriser 192.192.1.0/24 vers WAN              |
| 102 | Étendue  | Router1  | s0/0/0    | in        | Filtrer entrants + autoriser RIP + bloquer ping|
| 103 | Étendue  | Router1  | s0/0/0    | out       | Blocage destinations + accès email sélectif    |
| 104 | Étendue  | Router3  | gi0/2     | out       | Blacklister un PC vers un serveur web          |
| 105 | Étendue  | Router4  | s0/0/0    | in        | Limiter aux services HTTP, DNS, RIP            |

---

## Bonnes pratiques

- **Règle implicite `deny any`** : toute ACL se termine implicitement par un refus total. Toujours prévoir un `permit` explicite si nécessaire.
- **Ordre des règles** : placer les règles les plus spécifiques en premier.
- **ACL standard vs étendue** : les ACL standard (1–99) filtrent uniquement sur l'IP source ; les ACL étendues (100–199) filtrent sur source, destination, protocole et port.
- **Placement des ACL** : appliquer les ACL standard près de la destination, les ACL étendues près de la source.
- **Numéros de séquence** : utiliser `ip access-list extended <numéro>` pour modifier une ACL sans la recréer entièrement.
