# Projet 04 — Clair contre chiffré / Cleartext vs encrypted

🇫🇷 [Français](#-français) · 🇬🇧 [English](#-english)

---

## 🇫🇷 Français

### Objectif

Montrer, capture à l'appui, ce que voit exactement quelqu'un qui intercepte du trafic réseau — selon que ce trafic est chiffré ou non.

### Environnement

Le segment `labnet` construit au projet 03, avec Kali en position d'observateur.

| Machine | Rôle | Adresse |
|---|---|---|
| Ubuntu Server | Serveur HTTP et SSH | `10.10.10.20` |
| Kali Linux | Client et point de capture | `10.10.10.10` |

Aucune donnée réelle n'a été utilisée. Le fichier servi ne contient qu'une phrase de démonstration, et le mot de passe saisi lors de la session SSH n'apparaît dans aucune capture.

### Démarche

**Sur Ubuntu** — un serveur HTTP minimal en arrière-plan, et le service SSH déjà actif :

```bash
echo "Contenu de demonstration - aucune donnee reelle" > demo.txt
python3 -m http.server 8000 &
```

**Sur Kali** — capture Wireshark sur `eth1`, puis génération du trafic :

```bash
curl http://10.10.10.20:8000/demo.txt      # non chiffré
ssh utilisateur@10.10.10.20                # chiffré
```

Analyse par clic droit → **Follow → TCP Stream** sur un paquet de chaque flux.

### Observations

#### HTTP — tout est lisible

![Flux HTTP reconstitué en clair](img/http-follow-stream.png)

Le flux reconstitué affiche l'échange entier : la requête avec ses en-têtes, la réponse du serveur, et le contenu du fichier.

```
GET /demo.txt HTTP/1.1
Host: 10.10.10.20:8000
User-Agent: curl/8.20.0

HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Content-type: text/plain

Contenu de demonstration - aucune donnee reelle
```

L'observateur n'a rien eu à décoder. La liste de paquets montre par ailleurs le cycle TCP complet — handshake, échange, `FIN, ACK` — sans qu'aucune étape ne masque la charge utile.

#### SSH — seule la négociation est visible

![Flux SSH reconstitué, illisible](img/ssh-follow-stream.png)

Le même traitement appliqué au flux SSH ne rend lisibles que deux lignes :

```
SSH-2.0-OpenSSH_10.3p1 Debian-4
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.18
```

Ce sont les bannières de version, échangées **avant** l'établissement du chiffrement pour que client et serveur s'accordent sur les algorithmes à employer. On distingue ensuite la liste des suites cryptographiques proposées, puis plus rien d'exploitable.

Le compteur en bas de fenêtre indique `281 client pkts, 283 server pkts, 28 kB`. Vingt-huit kilo-octets ont circulé — l'authentification, le mot de passe, chaque commande tapée et chaque réponse — et rien n'en est lisible.

### Ce que j'en retiens

**La différence entre les deux captures est la démonstration entière.** Aucune explication théorique du chiffrement en transit n'est aussi convaincante que ces deux fenêtres côte à côte : le même outil, le même segment, le même type d'échange, et dans un cas tout s'affiche, dans l'autre rien.

**Cette position d'observation n'a rien d'exceptionnel.** Kali n'est ici ni l'émetteur ni le destinataire : c'est une troisième machine sur le même segment. C'est exactement la situation de quelqu'un connecté au même réseau Wi-Fi public que vous. Le passage généralisé à HTTPS n'est pas une précaution abstraite, c'est la réponse directe à ce qui est visible sur la première capture.

**Le chiffrement ne cache pas tout.** Les bannières de version restent en clair, et elles sont exploitables : elles révèlent le système d'exploitation et la version exacte du démon SSH des deux côtés. C'est une information de reconnaissance réelle — une version obsolète y serait immédiatement repérable. Les métadonnées de connexion, elles, restent également visibles : qui parle à qui, sur quel port, à quel moment et pour quel volume.

**Un diagnostic descend les couches.** La capture HTTP a d'abord échoué avec `curl: (7) Failed to connect`. Le serveur applicatif était pourtant irréprochable — `ss -tlnp` confirmait l'écoute sur `0.0.0.0:8000` et un `curl` en local retournait bien le contenu. La cause réelle se situait en couche 3 : les adresses statiques du segment `labnet`, posées avec `ip addr add`, avaient disparu au redémarrage des machines. Tester `ping` avant `curl` aurait identifié le problème immédiatement. Un service applicatif ne peut pas fonctionner au-dessus d'une couche réseau absente, et l'erreur retournée par l'outil du haut ne désigne pas nécessairement la couche fautive.

**Prolongement de l'exercice.** Le même contraste avait déjà été rencontré de l'intérieur, en résolvant les niveaux 14 et 15 du wargame OverTheWire Bandit : l'un exigeait `nc` sur un service en clair, l'autre `openssl s_client` sur un service chiffré. Le refaire depuis la position de l'observateur, et non plus du client, change complètement ce qu'on en comprend.

---

## 🇬🇧 English

### Objective

Show, with evidence, exactly what someone intercepting network traffic sees — depending on whether that traffic is encrypted.

### Environment

The `labnet` segment built in project 03, with Kali in the observer's position.

| Machine | Role | Address |
|---|---|---|
| Ubuntu Server | HTTP and SSH server | `10.10.10.20` |
| Kali Linux | Client and capture point | `10.10.10.10` |

No real data was used. The file served contains only a demonstration sentence, and the password typed during the SSH session appears in no capture.

### Method

**On Ubuntu** — a minimal HTTP server in the background, with the SSH service already running:

```bash
echo "Contenu de demonstration - aucune donnee reelle" > demo.txt
python3 -m http.server 8000 &
```

**On Kali** — Wireshark capturing on `eth1`, then generating the traffic:

```bash
curl http://10.10.10.20:8000/demo.txt      # unencrypted
ssh user@10.10.10.20                       # encrypted
```

Analysis via right click → **Follow → TCP Stream** on a packet from each flow.

### Observations

#### HTTP — everything is readable

![Reassembled HTTP stream in cleartext](img/http-follow-stream.png)

The reassembled stream shows the entire exchange: the request with its headers, the server's response, and the file's contents.

```
GET /demo.txt HTTP/1.1
Host: 10.10.10.20:8000
User-Agent: curl/8.20.0

HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Content-type: text/plain

Contenu de demonstration - aucune donnee reelle
```

The observer had nothing to decode. The packet list also shows the full TCP cycle — handshake, exchange, `FIN, ACK` — with no step obscuring the payload.

#### SSH — only the negotiation is visible

![Reassembled SSH stream, unreadable](img/ssh-follow-stream.png)

The same treatment applied to the SSH flow leaves only two readable lines:

```
SSH-2.0-OpenSSH_10.3p1 Debian-4
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.18
```

These are the version banners, exchanged **before** encryption is established so that client and server can agree on which algorithms to use. The list of proposed cipher suites follows, and after that nothing usable.

The counter at the bottom of the window reads `281 client pkts, 283 server pkts, 28 kB`. Twenty-eight kilobytes travelled — the authentication, the password, every command typed and every response — and none of it is readable.

### Key takeaways

**The difference between the two captures is the whole demonstration.** No theoretical explanation of encryption in transit is as convincing as these two windows side by side: same tool, same segment, same kind of exchange, and in one case everything is displayed, in the other nothing.

**This observation position is nothing exceptional.** Kali here is neither sender nor recipient: it is a third machine on the same segment. That is exactly the situation of someone connected to the same public Wi-Fi network as you. The general move to HTTPS is not an abstract precaution, it is the direct answer to what the first capture makes visible.

**Encryption does not hide everything.** The version banners stay in cleartext, and they are actionable: they reveal the operating system and the exact SSH daemon version on both ends. That is genuine reconnaissance information — an outdated version would be immediately spottable. Connection metadata likewise remains visible: who is talking to whom, on which port, at what time and in what volume.

**Diagnosis moves down the layers.** The HTTP capture first failed with `curl: (7) Failed to connect`. The application server was nonetheless faultless — `ss -tlnp` confirmed it listening on `0.0.0.0:8000` and a local `curl` returned the contents. The real cause sat at layer 3: the `labnet` static addresses, set with `ip addr add`, had vanished when the machines rebooted. Testing `ping` before `curl` would have identified the problem immediately. An application service cannot work on top of an absent network layer, and the error returned by the upper tool does not necessarily point at the faulty layer.

**An extension of earlier work.** The same contrast had already been met from the inside, while solving levels 14 and 15 of the OverTheWire Bandit wargame: one required `nc` against a cleartext service, the other `openssl s_client` against an encrypted one. Repeating it from the observer's position rather than the client's changes what you take away from it entirely.
