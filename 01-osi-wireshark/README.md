# Projet 01 — Dissection de paquets / Packet dissection

🇫🇷 [Français](#-français) · 🇬🇧 [English](#-english)

---

## 🇫🇷 Français

### Objectif

Vérifier le modèle OSI par l'observation directe plutôt que par la mémorisation : ouvrir un paquet réel et y retrouver chaque couche, puis observer l'ouverture d'une connexion TCP.

### Environnement

Kali Linux sous VirtualBox, adaptateur en mode NAT (`10.0.2.15/24`), capture sur l'interface `eth0` avec Wireshark.

### Démarche

**Capture DNS** — une requête simple, volontairement minimale, pour isoler les couches sans bruit :

```bash
dig google.com
```

Filtre d'affichage : `dns`

**Capture TCP** — l'ouverture d'une connexion HTTP :

```bash
curl -4 -s http://example.com -o /dev/null
```

Filtre d'affichage : `tcp.flags.syn == 1 or tcp.flags.ack == 1`

Le drapeau `-4` force IPv4. Sans lui, la première tentative est partie en IPv6 vers `example.com`, qui a répondu par un `RST, ACK` — un rejet de connexion, pas un handshake. Le flux capturé était donc inexploitable pour l'exercice.

### Observations

#### Les couches empilées dans une requête DNS

![Couches OSI dans une requête DNS](img/couches-osi-dns.png)

| Ce que Wireshark affiche | Couche OSI | Ce qu'on y lit |
|---|---|---|
| Frame | — (métadonnée de capture) | Horodatage, taille, interface |
| Ethernet II | 2 — Liaison | Adresses MAC source et destination |
| Internet Protocol Version 4 | 3 — Réseau | Adresses IP, TTL, protocole encapsulé |
| User Datagram Protocol | 4 — Transport | Ports source et destination (53) |
| Domain Name System | 7 — Application | Le nom demandé, le type d'enregistrement |

La ligne `[Protocols in frame: eth:ethertype:ip:udp:dns]` résume l'encapsulation entière en une seule chaîne. L'adresse du résolveur DNS a été caviardée.

#### L'ouverture d'une connexion TCP

![Handshake TCP en trois temps](img/handshake-tcp.png)

Les trois premiers paquets forment le handshake : `[SYN]`, `[SYN, ACK]`, `[ACK]`. La capture montre ensuite le cycle complet — `GET / HTTP/1.1`, `200 OK`, puis la fermeture ordonnée par `[FIN, ACK]`.

Le détail du premier paquet affiche deux formes du même champ :

```
Sequence Number: 0        (relative sequence number)
Sequence Number (raw): 3384490473
Flags: 0x002 (SYN)
```

### Ce que j'en retiens

**Pourquoi DNS utilise UDP plutôt que TCP**

UDP n'établit aucune connexion préalable. Une requête DNS coûte deux paquets au total, là où TCP imposerait d'abord trois paquets de handshake, puis la requête, la réponse et la fermeture — un coût disproportionné pour un échange aussi court. L'absence d'état permet en outre à un serveur DNS de traiter un volume de requêtes bien supérieur.

DNS utilise malgré tout TCP dans deux cas, sur le même port 53 : lorsqu'une réponse dépasse 512 octets (le serveur active le drapeau de troncature `TC` et le client relance en TCP), et pour les transferts de zone entre serveurs.

**Le TTL et ce qui se passe à zéro**

Le TTL de mon paquet vaut 64, valeur par défaut sous Linux (Windows initialise à 128). Malgré son nom, ce n'est pas une durée mais un compteur de sauts, décrémenté de 1 par chaque routeur traversé.

Quand il atteint zéro, le routeur détruit le paquet et renvoie à l'expéditeur un message ICMP Time Exceeded (type 11). Ce n'est pas un ACK : l'ACK appartient à TCP en couche 4, alors que le TTL vit dans l'en-tête IP en couche 3. Le mécanisme existe pour empêcher qu'un paquet mal routé ne circule indéfiniment dans une boucle.

C'est ce comportement que `traceroute` exploite : en envoyant des paquets avec un TTL croissant (1, puis 2, puis 3…), il collecte les ICMP Time Exceeded et reconstitue le chemin routeur par routeur.

**À quoi servent les numéros de séquence initiaux**

Ils remplissent trois rôles. Le réassemblage dans l'ordre d'abord : les segments d'une même connexion peuvent arriver dans le désordre, et la numérotation permet de les remettre en séquence. La détection de perte ensuite : un trou dans la numérotation signale un segment manquant à retransmettre.

Le troisième rôle est une mesure de sécurité : le numéro de séquence initial est tiré aléatoirement. S'il était prévisible, un attaquant pourrait deviner la numérotation d'une connexion en cours et y injecter des paquets sans jamais l'intercepter. Le `SYN` du handshake signifie *Synchronize* — chaque extrémité choisit son propre ISN, et le handshake sert à les synchroniser.

La capture rend cette notion tangible : Wireshark affiche `Sequence Number: 0` par commodité de lecture, mais la valeur réellement transmise sur le réseau est `3384490473`. Le zéro est une normalisation d'affichage, pas ce qui circule.

**Ce qui disparaît au passage d'un routeur**

Toute la trame de couche 2 est détruite et reconstruite à chaque saut. Le routeur retire l'en-tête Ethernet complet, consulte l'en-tête IP pour décider de la destination, puis fabrique une nouvelle trame portant sa propre adresse MAC en source et celle du prochain équipement en destination.

C'est la distinction centrale entre les deux couches : l'adressage de couche 2 est local à un segment et réécrit à chaque saut, tandis que l'adressage de couche 3 reste identique de bout en bout — à l'exception du NAT, qui réécrit l'adresse IP source.

**Un filtre trop étroit masque le phénomène qu'on veut observer**

La première tentative utilisait `tcp.flags.syn == 1`. Ce filtre exclut mécaniquement le troisième temps du handshake, l'`ACK` final ne portant pas le drapeau SYN : on ne voit que deux paquets sur trois et le mécanisme paraît incomplet. Élargir à `tcp.flags.syn == 1 or tcp.flags.ack == 1` rétablit la séquence entière.

---

## 🇬🇧 English

### Objective

Verify the OSI model through direct observation rather than memorisation: open a real packet and locate each layer inside it, then observe a TCP connection being established.

### Environment

Kali Linux under VirtualBox, adapter in NAT mode (`10.0.2.15/24`), capture on interface `eth0` with Wireshark.

### Method

**DNS capture** — a deliberately minimal request, to isolate the layers without noise:

```bash
dig google.com
```

Display filter: `dns`

**TCP capture** — an HTTP connection being opened:

```bash
curl -4 -s http://example.com -o /dev/null
```

Display filter: `tcp.flags.syn == 1 or tcp.flags.ack == 1`

The `-4` flag forces IPv4. Without it, the first attempt went out over IPv6 to `example.com`, which answered with a `RST, ACK` — a connection refusal, not a handshake. The captured stream was therefore unusable for the exercise.

### Observations

#### Layers stacked inside a DNS query

![OSI layers in a DNS query](img/couches-osi-dns.png)

| What Wireshark shows | OSI layer | What you read there |
|---|---|---|
| Frame | — (capture metadata) | Timestamp, size, interface |
| Ethernet II | 2 — Data link | Source and destination MAC addresses |
| Internet Protocol Version 4 | 3 — Network | IP addresses, TTL, encapsulated protocol |
| User Datagram Protocol | 4 — Transport | Source and destination ports (53) |
| Domain Name System | 7 — Application | The queried name, the record type |

The line `[Protocols in frame: eth:ethertype:ip:udp:dns]` summarises the entire encapsulation in a single string. The DNS resolver's address has been redacted.

#### A TCP connection opening

![Three-way TCP handshake](img/handshake-tcp.png)

The first three packets form the handshake: `[SYN]`, `[SYN, ACK]`, `[ACK]`. The capture then shows the full cycle — `GET / HTTP/1.1`, `200 OK`, and the orderly teardown through `[FIN, ACK]`.

The first packet's detail pane displays two forms of the same field:

```
Sequence Number: 0        (relative sequence number)
Sequence Number (raw): 3384490473
Flags: 0x002 (SYN)
```

### Key takeaways

**Why DNS uses UDP rather than TCP**

UDP is connectionless. A DNS query costs two packets in total, whereas TCP would first require a three-packet handshake, then the query, the response, and the teardown — disproportionate overhead for such a short exchange. Being stateless also lets a DNS server handle a far higher request volume.

DNS does still use TCP in two cases, on the same port 53: when a response exceeds 512 bytes (the server sets the truncation flag `TC` and the client retries over TCP), and for zone transfers between servers.

**TTL and what happens at zero**

The TTL on my packet is 64, the Linux default (Windows initialises to 128). Despite its name, it is not a duration but a hop counter, decremented by 1 at every router it crosses.

When it reaches zero, the router discards the packet and returns an ICMP Time Exceeded message (type 11) to the sender. This is not an ACK: ACKs belong to TCP at layer 4, whereas TTL lives in the IP header at layer 3. The mechanism exists to stop a misrouted packet from circulating indefinitely in a loop.

This is exactly the behaviour `traceroute` exploits: by sending packets with an incrementing TTL (1, then 2, then 3…), it collects the ICMP Time Exceeded replies and reconstructs the path router by router.

**What initial sequence numbers are for**

They serve three purposes. In-order reassembly first: segments belonging to the same connection can arrive out of order, and the numbering lets the receiver put them back in sequence. Loss detection second: a gap in the numbering signals a missing segment that needs retransmitting.

The third purpose is a security measure: the initial sequence number is chosen randomly. If it were predictable, an attacker could guess the numbering of an ongoing connection and inject packets into it without ever intercepting the traffic. The `SYN` in the handshake stands for *Synchronize* — each endpoint picks its own ISN, and the handshake exists to synchronise them.

The capture makes this tangible: Wireshark displays `Sequence Number: 0` for readability, but the value actually transmitted on the wire is `3384490473`. The zero is a display normalisation, not what travels.

**What disappears when a packet crosses a router**

The entire layer 2 frame is destroyed and rebuilt at every hop. The router strips the full Ethernet header, reads the IP header to determine where the packet should go, then builds a new frame carrying its own MAC address as the source and the next device's MAC as the destination.

This is the core distinction between the two layers: layer 2 addressing is local to a segment and rewritten at every hop, while layer 3 addressing stays the same end to end — with the exception of NAT, which rewrites the source IP address.

**A filter that is too narrow hides the very thing you want to observe**

The first attempt used `tcp.flags.syn == 1`. That filter mechanically excludes the third leg of the handshake, since the final `ACK` does not carry the SYN flag: only two packets out of three appear and the mechanism looks incomplete. Widening it to `tcp.flags.syn == 1 or tcp.flags.ack == 1` restores the full sequence.
