# Murder Party 🔪

Application web mobile temps réel pour un jeu grandeur nature façon *Among Us*. Tout tient dans un seul fichier `index.html` (Tailwind + Firebase via CDN), sans installation pour les joueurs.

## Mise en place (une seule fois, ~10 min)

1. **Créer le projet** sur <https://console.firebase.google.com>.
2. **Realtime Database** → *Créer une base de données* → région `europe-west1` → mode verrouillé.
3. **Règles** : onglet *Règles* de la Realtime Database → coller le contenu de [`database.rules.json`](database.rules.json) → *Publier*.
4. **Authentification** → *Commencer* → *Sign-in method* → activer **Anonyme**.
5. **Application Web** : *Paramètres du projet* → *Vos applications* → icône `</>` → copier l'objet `firebaseConfig` et le coller en haut du `<script type="module">` de `index.html`.
6. **Mot de passe d'accès** : dans l'onglet *Données* de la Realtime Database, créer la clé `config/password` avec le mot de passe de la soirée comme valeur (texte).

### Mot de passe d'accès

- **Saisie :** l'app demande le mot de passe à la première ouverture, puis chaque téléphone le retient.
- **Stockage :** il n'est **jamais** écrit dans le code, qui est public. Il n'existe que dans `config/password`, illisible par tous.
- **Vérification :** quand un joueur saisit le mot de passe, l'app l'enregistre dans `members/{uid}`. Les règles n'acceptent cette écriture que si la valeur est identique à `config/password`, et toutes les autres règles (parties, rôles, catalogue) exigent ce passage.
- **Changement :** modifier `config/password` dans la console oblige tout le monde à saisir le nouveau mot de passe.

## Déploiement

N'importe quel hébergement statique fonctionne :

- **Netlify Drop** : glisser le dossier sur <https://app.netlify.com/drop>.
- **GitHub Pages** : pousser le dépôt puis *Settings → Pages → Deploy from branch*.
- **Vercel** : `vercel` à la racine du dossier.

Pour tester en local, servir le dossier en HTTP (par ex. `npx serve .`) plutôt que d'ouvrir le fichier directement.

## Déroulement d'une partie

1. L'hôte crée la partie : un code à 4 lettres et un **QR code** s'affichent dans son panneau.
2. Les joueurs scannent le QR code (le code est pré-rempli) et saisissent un pseudo. Ils arrivent dans le **salon d'attente**.
3. L'hôte règle :
   - le nombre de **tueurs** (au moins 1) et de **shérifs** ;
   - le **délai de sabotage**, avec un raccourci (15 à 90 s) ou une valeur exacte de 1 à 3600 s ;
   - le **nombre de tâches** par partie.

   Il peut aussi retirer un joueur absent avec ✕.
4. **Lancer la partie** tire au sort les tâches dans le catalogue et les rôles parmi tous les joueurs du salon. Chacun découvre son rôle en maintenant l'écran appuyé.

### Catalogue de tâches

- **Stockage :** le catalogue est enregistré dans Firebase (`/catalog`) et **partagé par toutes les parties**. Au tout premier lancement, il est rempli avec les 24 tâches de `DEFAULT_CATALOG` (dans `index.html`).
- **Gestion :** n'importe quel joueur l'ouvre depuis l'accueil avec **📚 Catalogue de tâches**, sans créer de partie. L'hôte peut aussi l'ouvrir depuis son panneau. On peut :
  - **décocher** une tâche pour l'exclure du tirage, par exemple si le matériel n'est pas installé ce soir-là ;
  - **toucher** une tâche pour modifier son nom, sa catégorie, son lieu, « À faire » et « Comment la saboter », ou pour la supprimer définitivement ;
  - ajouter une tâche avec **+ Nouvelle tâche**.
- **Effet sur la partie :** les modifications valent pour les parties suivantes. Une partie en cours garde sa copie des tâches tirées.
- **Détail en jeu :** sur chaque tâche, le bouton **ⓘ** affiche ce qu'il faut faire et comment la saboter. Il est visible par tout le monde.

### Rôles

| Rôle | Dans l'app | Dans la maison |
| --- | --- | --- |
| 🔪 Tueur | « Vérifier » sur une tâche verte programme un sabotage. S'il y a plusieurs tueurs, chacun connaît ses complices. | Élimine les survivants. |
| 🤠 Shérif | Comme un survivant. | Survivant armé : peut neutraliser le tueur. |
| 🙂 Survivant | Valide les tâches. | Survit. |

Un joueur qui arrive après le lancement joue survivant.

### Mécaniques

| Qui | Action | Effet |
| --- | --- | --- |
| Tout le monde | **Valider la tâche** (tâche grise ou rouge) | La tâche passe au vert. |
| Survivant / shérif | **Vérifier** (tâche verte) | « Rien à signaler ». Sert de camouflage. |
| Tueur | **Vérifier** (tâche verte) | Programme un sabotage. La tâche reste verte pour tous ; un décompte discret (`⏱ 23 s`) apparaît seulement sur le téléphone du tueur. |
| — | Fin du délai | La tâche devient rouge chez tout le monde : bandeau clignotant, son et vibration (Android). |

- Les interfaces des trois rôles sont **identiques**. Le rôle s'affiche en maintenant le bouton `👁 Rôle` appuyé.
- Pendant le délai, la tâche est verte, donc personne ne peut la revalider : le sabotage aboutit forcément.
- **Victoire des survivants** : automatique quand toutes les tâches sont vertes **et** qu'aucun sabotage n'est en attente. L'hôte peut aussi terminer la partie à tout moment (par exemple si les tueurs ont gagné « physiquement »).

## Choix techniques

- **Heure serveur** : l'échéance `sabotageTriggerAt` est calculée avec `.info/serverTimeOffset`, ce qui rend le jeu insensible aux horloges désynchronisées des téléphones.
- **Pas de client « arbitre »** : chaque téléphone calcule lui-même qu'une tâche est sabotée dès que l'échéance est passée. Le passage au rouge ne dépend donc pas du téléphone du tueur, qui peut être en veille. Le premier téléphone actif officialise ensuite le statut `sabotaged` dans la base via une transaction.
- **Transactions** sur chaque action (valider, saboter, officialiser), ce qui évite les conflits quand deux joueurs agissent en même temps.
- **Reconnexion automatique** : le salon et le pseudo sont mémorisés dans le navigateur, et un rechargement ou un écran verrouillé ramène directement dans la partie. Pendant la partie, l'écran reste allumé quand le navigateur le permet (Wake Lock).
- **Rôles privés** : le tirage est stocké dans `roles/{code}/{uid}`, un nœud séparé du salon. Les règles autorisent chaque joueur à lire **uniquement son propre rôle**.
- **Limites connues** (acceptables pour une soirée entre amis) :
  - le tirage est fait par le téléphone de l'hôte, donc un hôte expert, outils de développement ouverts, pourrait le voir ;
  - `sabotageTriggerAt` est lisible dans la base, donc un joueur expert pourrait repérer un sabotage en attente ;
  - le catalogue est modifiable par tout joueur connecté, puisqu'il n'y a pas de comptes. Les règles empêchent seulement de l'effacer d'un seul coup.

  Pour aller plus loin, il faudrait un serveur (Cloud Functions).

## Structure des données

```json
{
  "rooms": {
    "LOUV": {
      "code": "LOUV",
      "host": "Pierre",
      "hostUid": "uid-anonyme",
      "status": "lobby | playing | finished",
      "endReason": "survivors | host",
      "sabotageDelay": 30,
      "killers": 1,
      "sheriffs": 1,
      "taskCount": 8,
      "players": { "uid": { "pseudo": "Alice", "online": true } },
      "tasks": {
        "-Nx…": {
          "id": "-Nx…",
          "catalogId": "rideau",
          "title": "Le rideau",
          "location": "Chambre",
          "todo": "Ouvrir (ou fermer) entièrement le rideau…",
          "sabotage": "Tirer le rideau dans la position inverse…",
          "status": "pending | completed | sabotaged",
          "sabotageTriggerAt": 1760550000000,
          "order": 0
        }
      }
    }
  },
  "catalog": {
    "rideau": {
      "category": "Décor & murs",
      "title": "Le rideau",
      "location": "Chambre",
      "todo": "Ouvrir (ou fermer) entièrement le rideau…",
      "sabotage": "Tirer le rideau dans la position inverse…",
      "enabled": true,
      "order": 13
    }
  },
  "roles": {
    "LOUV": {
      "uid-alice": { "role": "TUEUR", "mates": ["Bob"] },
      "uid-bob": { "role": "TUEUR", "mates": ["Alice"] },
      "uid-chloe": { "role": "SHERIFF" }
    }
  }
}
```
