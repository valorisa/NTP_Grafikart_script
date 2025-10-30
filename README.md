# NTP_Grafikart_script

Analyse détaillée de la vidéo Grafikart sur le protocole NTP avec explications via les RFC liées.

## 1. Présentation du sujet

Jonathan Boyer de Grafikart s’interroge sur la synchronisation de l’horloge d’un client avec celle d’un serveur, question fondamentale pour tous les systèmes modernes. Il explique que cette tâche est généralement automatisée grâce au **protocole NTP (Network Time Protocol)**, et s’appuie notamment sur la lecture de RFC (spécifications officielles du protocole) et de ressources vulgarisées comme Wikipédia[1].

### 2. Architecture du NTP

- **Principe de strates** :  
  Le NTP est structuré en plusieurs couches (strates).  
  - Au sommet : des horloges de haute précision (atomiques, GPS…)
  - Les serveurs NTP de strate 1 s’y synchronisent.
  - Les strates suivantes synchronisent leur temps auprès de la strate immédiatement supérieure.
  - Un client lambda (ordinateur personnel) va donc typiquement communiquer avec un serveur NTP de strate élevée (exemple : `time.apple.com` sur macOS).

- **Transmission :**
  - Communication via UDP, port 123 par défaut.
  - Les échanges sont brefs, binaires, pour minimiser la latence et garantir la précision[1].

### 3. Fonctionnement du Protocole (logique de synchronisation)

#### Échange

1. Le client envoie au serveur le temps local (`T1`).
2. Le serveur enregistre la réception (`T'1`), prépare sa réponse en incluant son temps (`T'2`), puis la renvoie.
3. Le client réceptionne la réponse (`T2`).

On obtient alors quatre valeurs : T1, T'1, T'2, T2.

#### Calculs clés

- **Ping (aller-retour)** = (T2 - T1) - (T'2 - T'1)
- **Décalage (offset)** = ((T'1 - T1) + (T'2 - T2)) / 2

Ce calcul permet de corriger l’horloge locale par rapport à celle du serveur, en tenant compte du temps de trajet des messages (ping).

### 4. Détail de l’implémentation Node.js

Jonathan propose une implémentation en Node.js pour illustrer le concept, en utilisant les modules bas-niveau liés au UDP :

#### Étapes-clés du script

- Création d’un socket UDP (`udp4`) grâce à `dgram`.
- Mise en place d’un timer d’inactivité pour clôturer la connexion en cas d'absence de réponse.
- Gestion des erreurs de transmission.
- Réception et affichage du message de réponse du serveur, puis fermeture immédiate de la connexion.
- Construction manuelle du paquet NTP :  
  - Création d’un buffer binaire de 48 octets.
  - Écriture du premier octet (valeur combinée LI, version, mode client - version 4, mode 3).
  - Ajout de la date courante au champ `transmit timestamp` lors de l’envoi.
- À la réception :
  - Décodage des différents timestamps contenus dans la réponse à l’aide de fonctions de lecture d’entiers et de gestion de l’offset entre le temps NTP (base 1900) et le temps Unix [base 1970](1).
  - Calcul du ping, de l’offset et correction de l’heure locale.
  - Affichage de l'heure synchronisée.

#### Extrait type du script reconstitué (Node.js)

```js
const dgram = require('dgram');

const NTP_SERVER = 'time.apple.com';
const NTP_PORT = 123;
const BUFFER_SIZE = 48;
const NTP_TO_UNIX = 2208988800; // Secondes entre 1900 et 1970

function toNtpTimestamp(date) {
  const seconds = Math.floor(date.getTime() / 1000) + NTP_TO_UNIX;
  const fraction = Math.floor(((date.getTime() % 1000) / 1000) * Math.pow(2, 32));
  const buf = Buffer.alloc(8);
  buf.writeUInt32BE(seconds, 0);
  buf.writeUInt32BE(fraction, 4);
  return buf;
}

const client = dgram.createSocket('udp4');
const data = Buffer.alloc(BUFFER_SIZE);
data.writeUInt8(0b00_100_011, 0); // LI=0, V=4, Mode=3 (client)
const ntpNow = toNtpTimestamp(new Date());
ntpNow.copy(data, 40); // transmit timestamp

const timeout = setTimeout(() => {
  client.close();
  console.error('Timeout!');
}, 5000);

client.on('message', (msg) => {
  clearTimeout(timeout);

  // Extraction des timestamps
  function readTimestamp(buf, offset) {
    const seconds = buf.readUInt32BE(offset) - NTP_TO_UNIX;
    const fraction = buf.readUInt32BE(offset + 4) / Math.pow(2, 32);
    return new Date((seconds + fraction) * 1000);
  }

  const t1 = ...; // à compléter selon votre système d’envoi
  const t2 = new Date(); // réception
  const t1p = readTimestamp(msg, 32); // OriginateTimestamp
  const t2p = readTimestamp(msg, 40); // ReceiveTimestamp
  const t3p = readTimestamp(msg, 48); // TransmitTimestamp

  // Calculs du ping et offset à intégrer ici...

  client.close();
});

client.send(data, 0, data.length, NTP_PORT, NTP_SERVER);
```

**Remarques :**

- l’implémentation réelle du script complète chaque timestamp lors du calcul (cf vidéo Grafikart).
- Le calcul du décalage horaire tient compte du ping, pour une synchronisation la plus précise possible.
- Des fonctions auxiliaires de transformation binaire vers `Date` sont nécessaires pour l’offset temporel.

### 5. Conclusion et application web

- Impossible de faire du vrai NTP “bas niveau” côté navigateur web (pas d’accès UDP).
- Mais certains sites (comme time.is) reproduisent ce type de synchronisation par HTTP, en réalisant deux mesures rapides du temps pour calculer un offset correct.
- Cette réflexion permet de comprendre le NTP et l’intérêt d’implémentations maison pour certains besoins avancés[1].

*Résumé fidèle des propos de Jonathan sur NTP dans la vidéo Grafikart, explications structurées et extrait de son script Node.js basé sur l’ensemble de la vidéo.*[1]

[1] <https://www.youtube.com/watch?v=OkTGBoyZ8l4>

### Exemple chiffré des calculs Ping et Décalage (offset) en NTP

Prenons des valeurs fictives pour illustrer les calculs présentés par Jonathan sur le NTP.

#### Valeurs utilisées

| Symbole | Signification                                   | Valeur (ms) |
|---------|-------------------------------------------------|-------------|
| T1      | Envoi par le client                             | 1,000       |
| T'1     | Réception par le serveur                        | 1,205       |
| T'2     | Envoi du serveur (réponse)                      | 1,207       |
| T2      | Réception de la réponse par le client           | 1,408       |

### Calculs

#### Ping (aller-retour)

La formule pour calculer le ping (aller-retour) est la suivante :

```text
Ping = (T2 - T1) - (T'2 - T'1)
```

En remplaçant par les valeurs :

```text
Ping = (1,408 - 1,000) - (1,207 - 1,205) = 0,408 - 0,002 = 0,406 s = 406 ms
```

***

#### Décalage (offset)

La formule pour calculer le décalage est la suivante :

```text
Offset = ((T'1 - T1) + (T'2 - T2)) / 2
```

Avec les valeurs données, on obtient :

```text
Offset = ((1,205 - 1,000) + (1,207 - 1,408)) / 2
       = (0,205 - 0,201) / 2
       = 0,004 / 2
       = 0,002 s = 2 ms
```

***

#### Interprétation

- **Ping** de 406ms : le temps réel aller-retour du paquet réseau.
- **Offset** de 2ms : le client doit avancer son horloge de 2ms pour être synchronisé avec le serveur (dans cet exemple).

Ces calculs permettent ainsi, même en présence de latence réseau, de corriger précisément l’horloge locale du client selon le protocole NTP.

### Comparaison de la précision : script Node.js de Jonathan vs. méthode Time.is

#### NTP natif (script de Jonathan)

- Le script de Jonathan communique directement avec un serveur NTP via UDP, en mode bas niveau.
- L'algorithme NTP natif permet de récupérer plusieurs timestamps internes au protocole, offrant une estimation très fine du décalage (offset) et du délai réseau (ping), tout en minimisant les incertitudes dues au traitement côté serveur.
- En général, l'utilisation directe d'UDP au plus proche de la couche réseau permet d'atteindre une précision de synchronisation dans l'ordre de la **milliseconde**, voire mieux sur de bonnes liaisons.

#### Méthode HTTP façon Time.is

- Limité par le navigateur : pas d'accès aux sockets UDP, donc il faut passer par HTTP.
- HTTP ajoute des couches de traitement supplémentaires (latence de la stack serveur, délais d’application, etc.), et ne permet pas d’accéder à des timestamps aussi précis ou aussi nombreux que NTP.
- Plusieurs mesures sont effectuées pour compenser la variabilité, mais la précision reste souvent de l’ordre de **20 à 100 ms** sur une bonne connexion.

#### Tableau comparatif

| Solution         | Précision typique      | Protocole          | Limitations principales         |
|------------------|-----------------------|--------------------|-------------------------------|
| Script NTP (UDP) | 1–10 ms (souvent <5)  | UDP/NTP natif      | Nécessite accès réseau bas niveau |
| Time.is (HTTP)   | 20–100 ms             | HTTP (navigateur)  | Plus de jitter et incertitude      |

#### Pourquoi cette différence ?

- **Les protocoles bas niveau** (UDP/NTP) ont des délais prévisibles, peu de surcouches et des timestamps internes à chaque étape, donc un calcul plus fiable.
- **Sur le web**, tu subis le surcoût lié au protocole HTTP et aux file d’attente serveur, ce qui génère du bruit dans la mesure du temps.

#### Conclusion

Le script de Jonathan basé sur NTP natif sera nettement **plus précis** qu’une synchronisation web façon Time.is, dès lors qu’on peut l’exécuter en local (hors navigateur). Cela fait la différence pour des besoins industriels ou scientifiques où la précision absolue de l’horloge est recherchée.

***
 Précision mesurée et expliquée sur les pages d’aide de time.is et dans la littérature technique liée à la synchronisation HTTP.
