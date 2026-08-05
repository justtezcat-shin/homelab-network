# Projet 02 — Cartographie réseau / Network mapping

🇫🇷 [Français](#-français) · 🇬🇧 [English](#-english)

---

## 🇫🇷 Français

### Objectif

Relever les paramètres réseau d'une machine, calculer manuellement les limites de son sous-réseau, puis vérifier le calcul avec un outil.

### Environnement

| Élément | Valeur |
|---|---|
| Machine | Kali Linux (VirtualBox) |
| Mode réseau VirtualBox | NAT |
| Outils | `iproute2`, `ipcalc` |

### Démarche

```bash
ip a                    # adresse IP et préfixe
ip route                # passerelle par défaut
ip neigh                # table ARP (voisins connus)
cat /etc/resolv.conf    # serveurs DNS utilisés
```

![Sortie de ip a et ip route](img/ip-a-ip-route-nat.png)

### Paramètres relevés

| Paramètre | Valeur |
|---|---|
| Interface | `eth0` |
| Adresse MAC | `08:00:27:8a:35:d2` (OUI `08:00:27` = Oracle VirtualBox) |
| Adresse IP / préfixe | `10.0.2.15/24` |
| Masque de sous-réseau | `255.255.255.0` |
| Passerelle par défaut | `10.0.2.2` |
| Attribution | DHCP (bail de 86 173 s ≈ 24 h) |
| MTU | 1500 |
| Adresse lien-local IPv6 | `fe80::c4e1:89a7:aab:653a/64` |

### Calcul manuel du sous-réseau

Calcul effectué à la main avant toute vérification par outil :

| Élément | Résultat |
|---|---|
| Adresse de réseau | `10.0.2.0` |
| Adresse de diffusion | `10.0.2.255` |
| Première adresse utilisable | `10.0.2.1` |
| Dernière adresse utilisable | `10.0.2.254` |
| Nombre d'hôtes possibles | 254 (2⁸ − 2) |

Vérification :

```bash
ipcalc 10.0.2.15/24
```

Le calcul manuel correspondait au résultat de l'outil. Plusieurs observations méritent d'être relevées.

L'adresse de diffusion calculée à la main apparaît directement dans la sortie de `ip a`, au champ `brd` — une vérification immédiate sans passer par `ipcalc`.

Ce sous-réseau n'est pas un vrai réseau : c'est un segment virtuel fabriqué par l'hyperviseur. VirtualBox attribue systématiquement `10.0.2.15` à la machine invitée et `10.0.2.2` à sa passerelle NAT, quelle que soit l'installation. La machine peut atteindre l'extérieur, mais aucune machine du réseau physique ne peut l'atteindre — d'où l'isolation recherchée.

Fait notable, la passerelle occupe ici `.2` et non `.1`, contrairement à la convention habituelle. VirtualBox réserve `10.0.2.1` à un usage interne. C'est un rappel utile : les conventions d'adressage sont des habitudes très répandues, pas des règles, et une reconnaissance qui suppose que la passerelle est toujours en `.1` peut se tromper.

L'adresse MAC commence par `08:00:27`, un OUI attribué à Oracle et propre à VirtualBox. Ces trois premiers octets identifient le fabricant d'une carte réseau : lire une adresse MAC suffit donc souvent à déduire qu'une machine est virtualisée, et par quel hyperviseur.

Une adresse IPv6 lien-local en `fe80::/64` est enfin présente sur l'interface sans avoir été configurée. IPv6 s'auto-configure sur tout segment, et cette pile reste active même lorsque l'on ne travaille qu'en IPv4 — un point de surface d'attaque fréquemment oublié en durcissement système.

### Périmètre de l'exercice — et ce qui n'a pas été fait

Ce projet prévoyait initialement un balayage de découverte d'hôtes (`nmap -sn`) afin d'inventorier les machines du réseau local et d'en dessiner la topologie.

**Cette partie n'a volontairement pas été réalisée.** La session de travail s'est déroulée depuis un réseau universitaire, sur lequel je ne dispose d'aucune autorisation de balayage. Scanner une infrastructure dont on n'est ni propriétaire ni responsable — même par curiosité pédagogique, même avec un outil de découverte passif — relève au minimum du manquement au règlement, et potentiellement de l'infraction.

La règle appliquée est simple et sans exception : **aucun balayage en dehors de mon propre réseau domestique ou de mes propres machines virtuelles.**

La machine virtuelle a par ailleurs été basculée du mode pont vers le mode NAT pour la durée de la session, afin de garantir qu'aucun outil lancé depuis le laboratoire ne puisse atteindre le réseau environnant.

La découverte d'hôtes et le schéma de topologie seront réalisés depuis mon réseau personnel et ajoutés ultérieurement à ce dossier.

### Ce que j'en retiens

**Vérifier son contexte réseau avant de lancer un outil.** Un poste mobile change d'interface et de réseau sans prévenir entre le campus, un café et le domicile. Le réflexe correct consiste à exécuter `ip a` et `ip route` **avant** tout outil de découverte, pas après. La commande qu'on lance n'est jamais dangereuse en soi — c'est le contexte dans lequel on la lance qui l'est.

**Le mode réseau de la machine virtuelle détermine ce qu'on observe réellement.** En NAT, la VM vit dans un segment artificiel créé par l'hyperviseur et ne voit rien du réseau physique environnant. Une découverte d'hôtes menée dans ce mode retourne au mieux la passerelle virtuelle. Observer le réseau réel impose le mode pont — et donc une vigilance accrue sur le réseau auquel on est raccordé. Le mode NAT constitue de ce fait le réglage par défaut raisonnable pour tout travail de laboratoire hors du domicile.

**Le calcul manuel avant l'outil.** `ipcalc` donne la réponse instantanément, mais l'automatiser d'emblée empêche de repérer ses propres erreurs de raisonnement sur les frontières de sous-réseau. Faire le calcul d'abord, vérifier ensuite.

**Une sortie de commande contient plus que ce qu'on y cherche.** `ip a` était censé fournir une adresse et un préfixe. La même sortie révélait aussi l'hyperviseur employé (via l'OUI de l'adresse MAC), la durée du bail DHCP, la MTU du lien et l'existence d'une pile IPv6 active non configurée. Lire une sortie en entier plutôt que d'y prélever le champ attendu est une habitude directement transposable à l'analyse de journaux.

---

## 🇬🇧 English

### Objective

Collect a machine's network parameters, manually calculate its subnet boundaries, then verify the calculation with a tool.

### Environment

| Item | Value |
|---|---|
| Machine | Kali Linux (VirtualBox) |
| VirtualBox network mode | NAT |
| Tools | `iproute2`, `ipcalc` |

### Method

```bash
ip a                    # IP address and prefix
ip route                # default gateway
ip neigh                # ARP table (known neighbours)
cat /etc/resolv.conf    # DNS resolvers in use
```

![ip a and ip route output](img/ip-a-ip-route-nat.png)

### Parameters collected

| Parameter | Value |
|---|---|
| Interface | `eth0` |
| MAC address | `08:00:27:8a:35:d2` (OUI `08:00:27` = Oracle VirtualBox) |
| IP address / prefix | `10.0.2.15/24` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.0.2.2` |
| Assignment | DHCP (lease of 86,173 s ≈ 24 h) |
| MTU | 1500 |
| IPv6 link-local address | `fe80::c4e1:89a7:aab:653a/64` |

### Manual subnet calculation

Worked out by hand before any tool-based verification:

| Item | Result |
|---|---|
| Network address | `10.0.2.0` |
| Broadcast address | `10.0.2.255` |
| First usable address | `10.0.2.1` |
| Last usable address | `10.0.2.254` |
| Total usable hosts | 254 (2⁸ − 2) |

Verification:

```bash
ipcalc 10.0.2.15/24
```

The manual calculation matched the tool's output. Several observations are worth recording.

The broadcast address worked out by hand appears directly in the `ip a` output, in the `brd` field — an immediate check without reaching for `ipcalc`.

This subnet is not a real network: it is a virtual segment manufactured by the hypervisor. VirtualBox consistently assigns `10.0.2.15` to the guest machine and `10.0.2.2` to its NAT gateway, on every installation. The machine can reach the outside world, but no machine on the physical network can reach it — which is exactly the isolation being sought.

Notably, the gateway sits at `.2` here rather than `.1`, contrary to the usual convention. VirtualBox reserves `10.0.2.1` for internal use. A useful reminder: addressing conventions are widespread habits, not rules, and reconnaissance that assumes the gateway is always at `.1` can be wrong.

The MAC address begins with `08:00:27`, an OUI assigned to Oracle and specific to VirtualBox. These first three octets identify a network card's manufacturer, so reading a MAC address is often enough to infer that a machine is virtualised, and under which hypervisor.

Finally, an IPv6 link-local address in `fe80::/64` is present on the interface without having been configured. IPv6 self-configures on any segment, and this stack stays active even when working exclusively in IPv4 — an attack-surface consideration frequently overlooked during system hardening.

### Scope — and what was deliberately left out

This project originally included a host discovery sweep (`nmap -sn`) to inventory the machines on the local network and draw its topology.

**That portion was deliberately not carried out.** The working session took place on a university network, on which I hold no scanning authorisation. Scanning infrastructure you neither own nor administer — even out of educational curiosity, even with a passive discovery tool — is at minimum a policy violation and potentially a legal one.

The rule applied is simple and admits no exceptions: **no scanning outside my own home network or my own virtual machines.**

The virtual machine was additionally switched from bridged to NAT mode for the duration of the session, to guarantee that no tool launched from the lab could reach the surrounding network.

Host discovery and the topology diagram will be carried out from my personal network and added to this folder later.

### Key takeaways

**Check your network context before running a tool.** A mobile machine switches interfaces and networks without warning between campus, a café and home. The correct habit is to run `ip a` and `ip route` **before** any discovery tool, not after. The command itself is never the dangerous part — the context you run it in is.

**A virtual machine's network mode determines what you can actually observe.** In NAT mode, the VM lives in an artificial segment created by the hypervisor and sees nothing of the surrounding physical network. Host discovery run in this mode returns the virtual gateway at best. Observing the real network requires bridged mode — and therefore closer attention to which network you are attached to. NAT is consequently the sensible default for any lab work carried out away from home.

**Calculate by hand before reaching for the tool.** `ipcalc` gives the answer instantly, but automating it from the start hides your own reasoning errors about subnet boundaries. Work it out first, verify second.

**A command's output contains more than what you went looking for.** `ip a` was meant to supply an address and a prefix. The same output also revealed the hypervisor in use (via the MAC address OUI), the DHCP lease duration, the link MTU, and the presence of an active but unconfigured IPv6 stack. Reading output in full rather than extracting the expected field is a habit that transfers directly to log analysis.
