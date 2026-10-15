# Murder Party 🔪

Application web mobile temps réel pour un jeu grandeur nature façon *Among Us*. Tout tient dans un seul fichier `index.html` (Tailwind + Firebase via CDN), sans installation pour les joueurs.

## Mise en place (une seule fois, ~10 min)

1. **Créer le projet** sur <https://console.firebase.google.com>.
2. **Realtime Database** → *Créer une base de données* → région `europe-west1` → mode verrouillé.
3. **Règles** : onglet *Règles* de la Realtime Database → coller le contenu de [`database.rules.json`](database.rules.json) → *Publier*.
4. **Authentification** → *Commencer* → *Sign-in method* → activer **Anonyme**.
5. **Application Web** : *Paramètres du projet* → *Vos applications* → icône `</>` → copier l'objet `firebaseConfig` et le coller en haut du `<script type="module">` de `index.html`.

## Déploiement

N'importe quel hébergement statique fonctionne :

- **Netlify Drop** : glisser le dossier sur <https://app.netlify.com/drop>.
- **GitHub Pages** : pousser le dépôt puis *Settings → Pages → Deploy from branch*.
- **Vercel** : `vercel` à la racine du dossier.

Pour tester en local, servir le dossier en HTTP (par ex. `npx serve .`) plutôt que d'ouvrir le fichier directement.

## Déroulement d'une partie

1. L'hôte crée la partie : un code à 4 lettres et un **QR code** s'affichent dans son panneau.
2. Les joueurs scannent le QR code (le code est pré-rempli), saisissent un pseudo et choisissent **secrètement** leur rôle.
3. L'hôte règle le délai de sabotage, ajuste la liste des tâches, puis lance la partie.

### Mécaniques

| Qui | Action | Effet |
| --- | --- | --- |
| Tout le monde | **Valider la tâche** (tâche grise ou rouge) | La tâche passe au vert. |
| Survivant | **Vérifier** (tâche verte) | « Rien à signaler ». Sert de camouflage. |
| Tueur | **Vérifier** (tâche verte) | Programme un sabotage. La tâche reste verte pour tous ; un décompte discret (`⏱ 23 s`) apparaît seulement sur le téléphone du tueur. |
| — | Fin du délai | La tâche devient rouge chez tout le monde : bandeau clignotant, son et vibration (Android). |

- Les interfaces du tueur et du survivant sont **identiques**. Le rôle s'affiche en maintenant le bouton `👁 Rôle` appuyé.
- Pendant le délai, la tâche est verte, donc personne ne peut la revalider : le sabotage aboutit forcément.
- **Victoire des survivants** : automatique quand toutes les tâches sont vertes **et** qu'aucun sabotage n'est en attente. L'hôte peut aussi terminer la partie à tout moment (par exemple si les tueurs ont gagné « physiquement »).

## Choix techniques

- **Heure serveur** : l'échéance `sabotageTriggerAt` est calculée avec `.info/serverTimeOffset`, ce qui rend le jeu insensible aux horloges désynchronisées des téléphones.
- **Pas de client « arbitre »** : chaque téléphone calcule lui-même qu'une tâche est sabotée dès que l'échéance est passée. Le passage au rouge ne dépend donc pas du téléphone du tueur, qui peut être en veille. Le premier téléphone actif officialise ensuite le statut `sabotaged` dans la base via une transaction.
- **Transactions** sur chaque action (valider, saboter, officialiser), ce qui évite les conflits quand deux joueurs agissent en même temps.
- **Reconnexion automatique** : le salon, le pseudo et le rôle sont mémorisés dans le navigateur, et un rechargement ou un écran verrouillé ramène directement dans la partie. Pendant la partie, l'écran reste allumé quand le navigateur le permet (Wake Lock).
- **Le rôle n'est jamais envoyé à Firebase** : il reste uniquement sur le téléphone du joueur.
- **Limite connue** : `sabotageTriggerAt` est lisible dans la base. Un joueur expert, outils de développement ouverts, pourrait repérer un sabotage en attente. C'est acceptable pour une soirée entre amis.

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
      "players": { "uid": { "pseudo": "Alice", "online": true } },
      "tasks": {
        "-Nx…": {
          "id": "-Nx…",
          "title": "Multiprise",
          "location": "Salon",
          "status": "pending | completed | sabotaged",
          "sabotageTriggerAt": 1760550000000,
          "order": 0
        }
      }
    }
  }
}
```
