# Projet 03 — Lab à deux VM et pannes provoquées / Two-VM lab and injected faults

🇫🇷 [Français](#-français) · 🇬🇧 [English](#-english)

---

## 🇫🇷 Français

### Objectif

Construire un segment réseau isolé entre deux machines virtuelles, puis y provoquer délibérément des pannes d'adressage pour observer et diagnostiquer ce qui se produit.

### Environnement

| Machine | Interface `labnet` | Adresse |
|---|---|---|
| Kali Linux | `eth1` | `10.10.10.10/24` |
| Ubuntu Server 24.04 | `enp0s8` | `10.10.10.20/24` |

Chaque VM dispose d'un second adaptateur en NAT pour l'accès Internet. Le segment `labnet` est un **réseau interne** VirtualBox : aucun serveur DHCP n'y est présent, les adresses sont donc attribuées manuellement.

### Démarche

**Attribution des adresses statiques**

```bash
# Kali
sudo ip addr add 10.10.10.10/24 dev eth1
sudo ip link set eth1 up

# Ubuntu
sudo ip addr add 10.10.10.20/24 dev enp0s8
sudo ip link set enp0s8 up
```

**Vérification obligatoire avant toute suite** — l'interface doit être en `state UP` et porter l'adresse attendue :

```bash
ip a show eth1
```

### Observations

#### État de référence

![Ping de référence réussi](img/ping-reference-reussi.png)

```
3 packets transmitted, 3 received, 0% packet loss
```

Les deux machines communiquent. C'est l'état auquel comparer tout ce qui suit.

#### Panne n° 1 — masque de sous-réseau incohérent

Le préfixe d'Ubuntu passe de `/24` à `/28`, celui de Kali reste inchangé :

```bash
sudo ip addr del 10.10.10.20/24 dev enp0s8
sudo ip addr add 10.10.10.20/28 dev enp0s8
```

![Capture tcpdump pendant la panne de masque](img/casse-masque-28-tcpdump.png)

Le ping échoue avec `100% packet loss`. La capture `tcpdump` lancée pendant l'échec montre précisément ce qui se passe :

```
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, seq 1
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, seq 2
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, seq 3
ARP, Request who-has 10.10.10.20 tell 10.10.10.10
ARP, Reply 10.10.10.20 is-at 08:00:27:1b:fa:53
```

**La résolution ARP réussit.** C'est le point non trivial de cette panne, et il contredit ce à quoi on s'attend intuitivement. Linux répond aux requêtes ARP même lorsque la source se situe hors de son sous-réseau : Kali obtient donc bien l'adresse MAC d'Ubuntu et lui envoie ses paquets ICMP.

C'est **la réponse** qui ne peut pas revenir. Avec un préfixe `/28`, Ubuntu considère sa plage locale comme `10.10.10.16` → `10.10.10.31`. L'adresse `10.10.10.10` en est exclue : Ubuntu la traite comme distante, cherche une passerelle pour l'atteindre, n'en trouve aucune sur ce segment, et abandonne silencieusement.

Les deux machines sont sur le même segment physique mais logiquement sur deux réseaux différents. Rien dans le câblage ni dans l'état des interfaces ne le signale.

#### Panne n° 2 — machine placée sur un autre réseau

```bash
sudo ip addr del 10.10.10.20/28 dev enp0s8
sudo ip addr add 10.10.20.20/24 dev enp0s8
```

![Réponses corrompues lors du ping vers un autre réseau](img/casse-reseau-different.png)

Le résultat attendu était un `Network is unreachable` immédiat. **Ce n'est pas ce qui s'est produit.** Le ping reçoit des réponses, mais corrompues :

```
64 bytes from 10.10.20.20: icmp_seq=0 ttl=64 time=0.000 ms (DUP!)
wrong data byte #16 should be 0x10 but was 0xa
ping: Warning: time of day goes back (-234972989206748 s), taking countermeasures
3 packets transmitted, 1 received, +2 duplicates, 66.6667% packet loss
```

Octets de charge utile erronés, duplicatas, horodatages aberrants — le tout accompagné d'un TTL de 64 qui suggère une réponse générée localement plutôt qu'à distance.

**Hypothèse la plus probable** : Kali dispose d'une route par défaut via `10.0.2.2`, la passerelle NAT de VirtualBox, héritée de son adaptateur `eth0`. L'adresse `10.10.20.20` n'appartenant à aucun sous-réseau local, la requête est routée vers cette passerelle — et le moteur NAT de VirtualBox y répond lui-même, avec une charge utile fabriquée qui ne correspond pas à celle émise.

Vérification proposée :

```bash
ip route get 10.10.20.20
```

Si la sortie mentionne `via 10.0.2.2 dev eth0`, l'hypothèse est confirmée. Ce comportement est propre à cette configuration de laboratoire : sur une machine sans route par défaut, l'échec serait bien immédiat.

### Ce que j'en retiens

**Deux machines sur le même câble peuvent être sur deux réseaux différents.** Le masque de sous-réseau ne se voit ni au câblage, ni à l'état des interfaces, ni à l'adresse IP prise isolément. C'est la panne la plus banale en entreprise et elle demande de comparer les préfixes des deux côtés, pas seulement de vérifier que « les adresses se ressemblent ».

**L'ARP peut réussir alors que la communication échoue.** L'intuition veut qu'une machine injoignable ne réponde pas à l'ARP. C'est faux : la résolution d'adresse est un mécanisme de couche 2 qui ignore les considérations de sous-réseau. Voir un `ARP Reply` dans une capture ne prouve donc rien sur la joignabilité réelle — c'est l'absence de réponse ICMP qui est le signal.

**Un symptôme attendu qui ne se produit pas est une information, pas un échec.** La seconde panne devait produire `Network is unreachable`. Elle a produit des réponses forgées par une passerelle intermédiaire. Documenter cet écart vaut mieux que de le passer sous silence : une middlebox qui répond à la place d'une machine inexistante est un comportement qu'on rencontre en analyse réseau réelle, et le reconnaître évite de tirer des conclusions fausses d'un ping qui « fonctionne ».

**Ne jamais casser avant d'avoir établi une référence.** La première tentative de ce projet a appliqué la panne n° 1 alors que le ping de référence n'avait jamais fonctionné : l'interface Ubuntu était restée `DOWN`, le mot-clé `up` ayant été omis dans la commande `ip link set`. Deux pannes se sont ainsi superposées, et il est devenu impossible de savoir laquelle produisait quel symptôme. La démarche a été reprise depuis zéro, avec un point de contrôle bloquant après chaque étape.

**Un outil de capture doit tourner pendant l'événement, pas autour.** Toujours lors de la première tentative, `tcpdump` a été lancé puis interrompu avant que le ping ne soit émis. La sortie `0 packets captured` ne signifiait pas « rien ne circule » mais « rien ne circulait pendant que j'écoutais ». Observer une panne réseau demande deux terminaux : un pour la capture, un pour l'action.

**Une configuration temporaire l'est vraiment.** Les adresses posées avec `ip addr add` disparaissent au redémarrage de la machine. Ce comportement a provoqué, au projet suivant, un diagnostic erroné : un service applicatif était accusé alors que la couche 3 avait simplement disparu. La persistance passe par Netplan sur Ubuntu — sujet de la section 2 du Network+.

---

## 🇬🇧 English

### Objective

Build an isolated network segment between two virtual machines, then deliberately inject addressing faults to observe and diagnose what happens.

### Environment

| Machine | `labnet` interface | Address |
|---|---|---|
| Kali Linux | `eth1` | `10.10.10.10/24` |
| Ubuntu Server 24.04 | `enp0s8` | `10.10.10.20/24` |

Each VM has a second adapter in NAT mode for Internet access. The `labnet` segment is a VirtualBox **internal network**: no DHCP server is present, so addresses are assigned manually.

### Method

**Static address assignment**

```bash
# Kali
sudo ip addr add 10.10.10.10/24 dev eth1
sudo ip link set eth1 up

# Ubuntu
sudo ip addr add 10.10.10.20/24 dev enp0s8
sudo ip link set enp0s8 up
```

**Mandatory check before anything else** — the interface must be in `state UP` and carry the expected address:

```bash
ip a show eth1
```

### Observations

#### Reference state

![Successful reference ping](img/ping-reference-reussi.png)

```
3 packets transmitted, 3 received, 0% packet loss
```

The two machines communicate. This is the state everything that follows is compared against.

#### Fault 1 — inconsistent subnet mask

Ubuntu's prefix changes from `/24` to `/28`; Kali's stays unchanged:

```bash
sudo ip addr del 10.10.10.20/24 dev enp0s8
sudo ip addr add 10.10.10.20/28 dev enp0s8
```

![tcpdump capture during the mask fault](img/casse-masque-28-tcpdump.png)

The ping fails with `100% packet loss`. The `tcpdump` capture running during the failure shows exactly what happens:

```
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, seq 1
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, seq 2
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, seq 3
ARP, Request who-has 10.10.10.20 tell 10.10.10.10
ARP, Reply 10.10.10.20 is-at 08:00:27:1b:fa:53
```

**ARP resolution succeeds.** This is the non-obvious part of the fault, and it contradicts what you would intuitively expect. Linux answers ARP requests even when the source lies outside its subnet, so Kali does obtain Ubuntu's MAC address and does send it ICMP packets.

It is **the reply** that cannot come back. With a `/28` prefix, Ubuntu considers its local range to be `10.10.10.16` → `10.10.10.31`. The address `10.10.10.10` falls outside it: Ubuntu treats it as remote, looks for a gateway to reach it, finds none on this segment, and silently gives up.

Both machines sit on the same physical segment but are logically on two different networks. Nothing in the cabling or in the interface state signals it.

#### Fault 2 — machine placed on another network

```bash
sudo ip addr del 10.10.10.20/28 dev enp0s8
sudo ip addr add 10.10.20.20/24 dev enp0s8
```

![Corrupted replies when pinging another network](img/casse-reseau-different.png)

The expected outcome was an immediate `Network is unreachable`. **That is not what happened.** The ping receives replies, but corrupted ones:

```
64 bytes from 10.10.20.20: icmp_seq=0 ttl=64 time=0.000 ms (DUP!)
wrong data byte #16 should be 0x10 but was 0xa
ping: Warning: time of day goes back (-234972989206748 s), taking countermeasures
3 packets transmitted, 1 received, +2 duplicates, 66.6667% packet loss
```

Wrong payload bytes, duplicates, aberrant timestamps — together with a TTL of 64 suggesting a locally generated reply rather than a remote one.

**Most likely hypothesis**: Kali has a default route via `10.0.2.2`, VirtualBox's NAT gateway, inherited from its `eth0` adapter. Since `10.10.20.20` belongs to no local subnet, the request is routed to that gateway — and VirtualBox's NAT engine answers it itself, with a fabricated payload that does not match what was sent.

Suggested verification:

```bash
ip route get 10.10.20.20
```

If the output mentions `via 10.0.2.2 dev eth0`, the hypothesis is confirmed. This behaviour is specific to this lab configuration: on a machine with no default route, the failure would indeed be immediate.

### Key takeaways

**Two machines on the same cable can be on two different networks.** The subnet mask is invisible in the cabling, in the interface state, and in an IP address taken in isolation. It is the most mundane failure in enterprise networks, and it requires comparing prefixes on both sides rather than merely checking that "the addresses look similar".

**ARP can succeed while communication fails.** Intuition suggests an unreachable machine would not answer ARP. That is wrong: address resolution is a layer 2 mechanism that ignores subnet considerations. Seeing an `ARP Reply` in a capture therefore proves nothing about actual reachability — the absence of an ICMP reply is the real signal.

**An expected symptom that fails to appear is information, not failure.** The second fault was supposed to produce `Network is unreachable`. It produced replies forged by an intermediate gateway instead. Documenting that gap beats glossing over it: a middlebox answering on behalf of a machine that does not exist is a behaviour you meet in real network analysis, and recognising it prevents drawing false conclusions from a ping that "works".

**Never break anything before establishing a reference.** The first attempt at this project applied fault 1 while the reference ping had never worked: Ubuntu's interface had stayed `DOWN`, the `up` keyword having been omitted from the `ip link set` command. Two faults thus stacked, and it became impossible to tell which produced which symptom. The procedure was restarted from scratch with a blocking checkpoint after each step.

**A capture tool must run during the event, not around it.** Still during the first attempt, `tcpdump` was started then stopped before the ping was issued. The `0 packets captured` output did not mean "nothing is flowing" but "nothing was flowing while I was listening". Observing a network fault requires two terminals: one for the capture, one for the action.

**A temporary configuration really is temporary.** Addresses set with `ip addr add` vanish when the machine reboots. That behaviour caused a mistaken diagnosis in the following project: an application service was blamed when layer 3 had simply disappeared. Persistence goes through Netplan on Ubuntu — a topic for Network+ section 2.
