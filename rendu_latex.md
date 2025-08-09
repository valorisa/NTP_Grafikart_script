## Calcul complet : Ping et Décalage (offset) en NTP

### Ping (aller-retour) :

$$
\text{Ping} = (T_2 - T_1) - (T'_2 - T'_1)
$$

$$
\text{Ping} = (1,408 - 1,000) - (1,207 - 1,205) = 0,408 - 0,002 = 0,406 \ \text{s} = 406 \ \text{ms}
$$

---

### Décalage (offset) :

$$
\text{Offset} = \frac{(T'_1 - T_1) + (T'_2 - T_2)}{2}
$$

$$
\text{Offset} = \frac{(1,205 - 1,000) + (1,207 - 1,408)}{2} = \frac{0,205 - 0,201}{2} = \frac{0,004}{2} = 0,002 \ \text{s} = 2 \ \text{ms}
$$

---

**Interprétation :**  
- Le *ping* représente le temps aller-retour du message, ici **406 millisecondes**.  
- Le *décalage* correspond à la correction nécessaire pour synchroniser l'horloge du client avec celle du serveur, ici une avance de **2 millisecondes**.

