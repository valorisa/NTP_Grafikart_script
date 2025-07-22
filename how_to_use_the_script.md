# Guide : How to install and use the script 

Voici comment sauvegarder et exécuter le script Node.js de Jonathan, étape par étape :

### 1. Récupérer et sauvegarder le script

- **Ouvre ton éditeur de texte** préféré (par exemple GNU nano si tu es sur Windows ou Android, ou un autre éditeur de ton choix[1]).
- **Copie le script** (adapte-le selon les besoins, voici un exemple réduit fonctionnel :)
  
```js
// ntp.js
const dgram = require('dgram');

const NTP_SERVER = 'time.apple.com';
const NTP_PORT = 123;
const NTP_TO_UNIX = 2208988800;

const socket = dgram.createSocket('udp4');
const message = Buffer.alloc(48);
message[0] = 0b00_100_011; // LI=0, Version=4, Mode=3 (client)

const t1 = Date.now(); // t1: heure locale avant l'envoi

socket.send(message, 0, message.length, NTP_PORT, NTP_SERVER, (err) => {
  if (err) {
    console.error('Erreur lors de l’envoi :', err);
    socket.close();
    return;
  }
});

// Fonction pour décoder un timestamp NTP
function readTimestamp(buffer, offset) {
  const seconds = buffer.readUInt32BE(offset);
  const fraction = buffer.readUInt32BE(offset + 4);
  const ms = ((seconds - NTP_TO_UNIX) * 1000) + Math.round((fraction / 2 ** 32) * 1000);
  return ms;
}

socket.on('message', (msg) => {
  const t2 = Date.now(); // t2: heure locale à la réception

  const t1p = readTimestamp(msg, 24); // Originate Timestamp (T1)
  const t2p = readTimestamp(msg, 32); // Receive Timestamp (T'1)
  const t3p = readTimestamp(msg, 40); // Transmit Timestamp (T'2)

  const ping = (t2 - t1) - (t3p - t2p);
  const offset = ((t2p - t1) + (t3p - t2)) / 2;

  const timeServer = new Date(t3p + offset);

  console.log(`Heure locale (avant) : ${new Date(t1).toISOString()}`);
  console.log(`Heure reçue du serveur : ${new Date(t3p).toISOString()}`);
  console.log(`Heure locale (ajustée) : ${timeServer.toISOString()}`);
  console.log(`Décalage estimé : ${offset.toFixed(2)} ms`);
  console.log(`Ping approximatif : ${ping.toFixed(2)} ms`);

  socket.close();
});
 ```

- **Sauvegarde le fichier**, par exemple sous le nom :  
  `ntp.js`

### 2. Installer Node.js

- **Vérifie si Node.js est installé** :  
  ```bash
  node -v
  ```
  Si ce n’est pas le cas, [installe-le depuis le site officiel](https://nodejs.org/), ou via ton gestionnaire de paquets :
  - Sur Windows avec [Chocolatey] :  
    ```bash
    choco install nodejs
    ```
  - Sur Mac/Linux :  
    ```bash
    brew install node      # pour Homebrew
    sudo apt install nodejs # pour apt
    ```
  - Sur Android (via Termux) :  
    ```bash
    pkg install nodejs
    ```

### 3. Exécuter le script

- **Place-toi dans le dossier** où le fichier a été sauvegardé :
  ```bash
  cd chemin/vers/le/dossier
  ```
- **Lance le script avec Node.js** :
  ```bash
  node ntp.js
  ```

L’exécution affichera l’heure reçue du serveur NTP ainsi qu’un calcul du décalage avec l’horloge locale.  
Ces instructions sont compatibles avec tous les OS sur lesquels Node.js et UDP sont disponibles[1].

[1] tools.text_editors
