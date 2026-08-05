# Homelab réseau — parcours CompTIA Network+

🇫🇷 [Français](#-français) · 🇬🇧 [English](#-english)

---

## 🇫🇷 Français

Laboratoire personnel construit en parallèle de ma préparation au CompTIA Network+ (N10-009).

L'objectif n'est pas de reproduire des tutoriels, mais de vérifier chaque concept théorique par l'observation directe : voir les couches OSI empilées dans un paquet réel plutôt que les mémoriser, provoquer volontairement une panne de sous-réseau plutôt que lire sa description, comparer un flux HTTP et un flux SSH capturés côte à côte plutôt que retenir que « HTTPS est plus sûr ».

### Environnement

| Composant | Détail |
|---|---|
| Hyperviseur | Oracle VirtualBox |
| Machine d'analyse | Kali Linux 2026.1 |
| Machine cible | Ubuntu Server 24.04 LTS |
| Segments réseau | NAT (accès Internet) · Réseau interne `labnet` (segment isolé) |
| Outils | Wireshark · tcpdump · nmap · ip / iproute2 · ipcalc · curl · OpenSSH |

### Projets

| # | Projet | Concepts couverts |
|---|---|---|
| [01](01-osi-wireshark/) | Dissection de paquets | Modèle OSI, encapsulation, DNS sur UDP, handshake TCP, numéro de séquence initial |
| [02](02-cartographie/) | Cartographie réseau | Adressage IPv4, masques et sous-réseaux, DHCP, ARP, modes réseau d'un hyperviseur |
| [03](03-lab-deux-vm/) | Lab à deux VM et pannes provoquées | Masques de sous-réseau, résolution ARP, routage, méthode de diagnostic |
| [04](04-clair-vs-chiffre/) | Clair contre chiffré | HTTP vs SSH, chiffrement en transit, position de l'observateur réseau |

### Fil conducteur

Les quatre projets se lisent dans l'ordre. Le premier établit ce qu'est un paquet et comment ses couches s'emboîtent. Le deuxième applique ces notions à une machine réelle et à son sous-réseau. Le troisième construit un segment isolé à deux machines et le casse délibérément pour observer ce qui se produit quand l'adressage devient incohérent. Le quatrième utilise ce même segment pour montrer, image à l'appui, ce que voit quelqu'un qui intercepte du trafic — selon que ce trafic est chiffré ou non.

### Ce que ce dépôt cherche à démontrer

Chaque projet se termine par une section « ce que j'en retiens », rédigée après coup et sans recopier de source. Les écarts entre le résultat attendu et le résultat observé y sont documentés tels quels, y compris lorsqu'ils contredisent la procédure suivie — c'est le cas notamment du projet 03, où deux pannes provoquées n'ont pas produit les symptômes annoncés.

Les erreurs de méthode le sont également. Toujours dans le projet 03, la première tentative a consisté à casser la configuration avant d'avoir établi un état de référence fonctionnel : le diagnostic est alors devenu impossible, deux pannes se superposant sans qu'on puisse les distinguer. La correction de cette démarche est décrite dans le projet concerné.

### Périmètre et éthique

Tous les tests documentés ici ont été réalisés exclusivement sur des machines virtuelles m'appartenant ou sur mon propre réseau domestique. Aucun système tiers n'a été scanné ni testé.

Une partie du projet 02 — la découverte d'hôtes par balayage réseau — a été volontairement écartée pour cette raison : la session s'est déroulée depuis un réseau universitaire, sur lequel je ne dispose d'aucune autorisation. La machine virtuelle a été basculée en mode NAT pour la durée de la session, afin qu'aucun outil lancé depuis le laboratoire ne puisse atteindre le réseau environnant.

### Suite

Section 2 du Network+ : VLAN, routage, configuration d'équipements et adressage permanent via Netplan. Puis préparation du CompTIA Security+.

---

## 🇬🇧 English

A personal lab built alongside my preparation for the CompTIA Network+ (N10-009).

The goal is not to reproduce tutorials but to verify each theoretical concept through direct observation: seeing the OSI layers stacked inside a real packet rather than memorising them, deliberately causing a subnet failure rather than reading its description, comparing an HTTP stream and an SSH stream captured side by side rather than simply accepting that "HTTPS is safer".

### Environment

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Analysis machine | Kali Linux 2026.1 |
| Target machine | Ubuntu Server 24.04 LTS |
| Network segments | NAT (Internet access) · Internal network `labnet` (isolated segment) |
| Tools | Wireshark · tcpdump · nmap · ip / iproute2 · ipcalc · curl · OpenSSH |

### Projects

| # | Project | Concepts covered |
|---|---|---|
| [01](01-osi-wireshark/) | Packet dissection | OSI model, encapsulation, DNS over UDP, TCP handshake, initial sequence number |
| [02](02-cartographie/) | Network mapping | IPv4 addressing, masks and subnets, DHCP, ARP, hypervisor network modes |
| [03](03-lab-deux-vm/) | Two-VM lab and injected faults | Subnet masks, ARP resolution, routing, diagnostic method |
| [04](04-clair-vs-chiffre/) | Cleartext vs encrypted | HTTP vs SSH, encryption in transit, the network observer's position |

### Narrative thread

The four projects are meant to be read in order. The first establishes what a packet is and how its layers nest. The second applies those notions to a real machine and its subnet. The third builds an isolated two-machine segment and deliberately breaks it to observe what happens when addressing becomes inconsistent. The fourth uses that same segment to show, with evidence, what someone intercepting traffic actually sees — depending on whether that traffic is encrypted.

### What this repository sets out to demonstrate

Each project ends with a "key takeaways" section, written afterwards and without copying from any source. Gaps between the expected result and the observed result are documented as they occurred, including where they contradict the procedure followed — notably in project 03, where two injected faults did not produce the announced symptoms.

Method errors are recorded too. Again in project 03, the first attempt broke the configuration before a working reference state had been established: diagnosis then became impossible, with two faults overlapping and no way to tell them apart. The correction to that approach is described in the relevant project.

### Scope and ethics

All tests documented here were carried out exclusively on virtual machines I own or on my own home network. No third-party system was scanned or tested.

Part of project 02 — host discovery through a network sweep — was deliberately left out for that reason: the session took place on a university network on which I hold no authorisation. The virtual machine was switched to NAT mode for the duration of the session, so that no tool launched from the lab could reach the surrounding network.

### Next

Network+ section 2: VLANs, routing, device configuration and persistent addressing via Netplan. Then preparation for the CompTIA Security+.
