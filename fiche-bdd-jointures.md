# 🗄️ FICHE COMPLÈTE — Bases de données : jointures, GROUP BY, agrégations

> **Modèle de référence utilisé dans cette fiche** (cinéma + boutique) :
> ```
> Realisateur (id, nom, nationalite)
> Film        (id, titre, annee, duree, #realisateur_id)
> Acteur      (id, nom, date_naissance)
> Jouer       (#film_id, #acteur_id)              ← table de jonction (N,N)
>
> Client        (id, nom, email)
> Commande      (id, date, #client_id)
> Produit       (id, nom, prix, #categorie_id)
> Categorie     (id, nom)
> LigneCommande (#commande_id, #produit_id, quantite)   ← jonction AVEC attribut
> ```

---

## 1. Rappel — les 3 méthodes better-sqlite3

```typescript
db.prepare("SELECT ...").all(...)   // SELECT → tableau d'objets (plusieurs lignes)
db.prepare("SELECT ...").get(...)   // SELECT → un objet ou undefined (une ligne)
db.prepare("INSERT ...").run(...)   // INSERT/UPDATE/DELETE → { changes, lastInsertRowid }
```

**Lire plusieurs → `.all` · lire un / vérifier existence → `.get` · modifier → `.run`.**

---

## 2. SELECT, WHERE, LIKE (les bases)

```typescript
// Toutes les colonnes
db.prepare("SELECT * FROM Film").all();

// Colonnes choisies (à préférer)
db.prepare("SELECT titre, annee FROM Film").all();

// Filtre exact
db.prepare("SELECT * FROM Film WHERE annee = ?").all(2010);

// Recherche PARTIELLE (LIKE + %)
db.prepare("SELECT * FROM Film WHERE titre LIKE ?").all(`%${q}%`);   // contient q
// 'tol%'  = commence par   |   '%tol'  = finit par   |   '%tol%' = contient
```

**Toujours `?` paramétrique** (jamais de concaténation → injections SQL). Les `%` vont dans la **valeur**, pas dans la requête.

---

## 3. INNER JOIN — relation 1,N

Croiser deux tables pour récupérer une donnée liée. Exemple : chaque film **avec le nom de son réalisateur** (pas juste son id).

```typescript
db.prepare(`
  SELECT Film.titre, Film.annee, Realisateur.nom AS realisateur
  FROM Film
  JOIN Realisateur ON Film.realisateur_id = Realisateur.id
`).all();
```

- `JOIN Realisateur ON Film.realisateur_id = Realisateur.id` → relie chaque film à son réalisateur via la clé étrangère.
- `Realisateur.nom AS realisateur` → on récupère le **nom**, renommé proprement.
- On préfixe `Film.titre`, `Realisateur.nom` pour lever toute ambiguïté.

⚠️ L'ordre dans le `ON` n'a **aucune importance** : `Film.realisateur_id = Realisateur.id` ≡ `Realisateur.id = Film.realisateur_id`.

---

## 4. LEFT JOIN — garder TOUTES les lignes de gauche

`INNER JOIN` **cache** un film sans réalisateur. `LEFT JOIN` le garde (avec `null`).

```typescript
db.prepare(`
  SELECT Film.titre, Realisateur.nom AS realisateur
  FROM Film
  LEFT JOIN Realisateur ON Film.realisateur_id = Realisateur.id
`).all();
// un film sans réalisateur → { titre: "...", realisateur: null }
```

| | INNER JOIN | LEFT JOIN |
|---|---|---|
| film **sans** réalisateur | disparaît | apparaît (`realisateur: null`) |

⚠️ Avec `LEFT JOIN`, l'**ordre des tables** compte (`FROM Film LEFT JOIN ...` garde tous les films). Avec `INNER JOIN`, l'ordre est indifférent.

Côté front, gérer le `null` :
```typescript
p.textContent = `${it.titre} — ${it.realisateur ?? "Inconnu"}`;
```

---

## 5. Table de jonction — relation N,N

« Afficher la distribution d'un film » : les acteurs sont dans `Acteur`, le lien dans `Jouer`. On passe **par la jonction**.

```typescript
db.prepare(`
  SELECT Acteur.nom
  FROM Jouer
  JOIN Acteur ON Jouer.acteur_id = Acteur.id
  WHERE Jouer.film_id = ?
`).all(filmId);
```

Chemin : `FROM Jouer` (les associations) → `WHERE film_id = ?` (ce film) → `JOIN Acteur` (traduire les ids en noms).

**Dans l'autre sens** (la filmographie d'un acteur) :
```typescript
db.prepare(`
  SELECT Film.titre
  FROM Jouer
  JOIN Film ON Jouer.film_id = Film.id
  WHERE Jouer.acteur_id = ?
`).all(acteurId);
```

---

## 6. Triple jointure

Tous les films **avec réalisateur ET acteurs** (1,N + N,N en une requête) :

```typescript
db.prepare(`
  SELECT Film.titre,
         Realisateur.nom AS realisateur,
         Acteur.nom      AS acteur
  FROM Film
  JOIN Realisateur ON Film.realisateur_id = Realisateur.id
  JOIN Jouer       ON Jouer.film_id = Film.id
  JOIN Acteur      ON Jouer.acteur_id = Acteur.id
`).all();
```

Chaque `JOIN` suit un trait du modèle : Film → son réalisateur, puis Film → ses associations → leurs acteurs.

⚠️ **Multiplication des lignes** : le résultat a **une ligne par couple (film, acteur)**, le titre répété :
```
{ titre: "Inception", realisateur: "Nolan", acteur: "DiCaprio" }
{ titre: "Inception", realisateur: "Nolan", acteur: "Hardy" }
```

Pour **un objet par film** avec un tableau d'acteurs → regrouper en JS :
```typescript
const films: Record<string, { realisateur: string; acteurs: string[] }> = {};
for (const l of lignes) {
  if (!films[l.titre]) films[l.titre] = { realisateur: l.realisateur, acteurs: [] };
  films[l.titre].acteurs.push(l.acteur);
}
```

---

## 7. Fonctions d'agrégation

```typescript
db.prepare("SELECT COUNT(*) AS n FROM Film").get();              // compte les LIGNES
db.prepare("SELECT SUM(duree) AS total FROM Film").get();         // additionne les VALEURS
db.prepare("SELECT AVG(duree) AS moyenne FROM Film").get();       // moyenne
db.prepare("SELECT MIN(annee) AS plus_vieux FROM Film").get();    // minimum
db.prepare("SELECT MAX(annee) AS plus_recent FROM Film").get();   // maximum
```

⚠️ `COUNT` compte des **lignes**, `SUM` additionne des **valeurs**. Avec un calcul global → `.get()` (une seule ligne).

```typescript
const r = db.prepare("SELECT COUNT(*) AS n FROM Film").get() as { n: number };
console.log(r.n);   // 42
```

Toujours mettre `AS` sinon la colonne s'appelle `COUNT(*)` (pénible côté JS).

---

## 8. GROUP BY — calculer PAR catégorie

Sans `GROUP BY` → un seul total global. Avec → un total **par groupe**.

```typescript
// Combien de films PAR réalisateur ?
db.prepare(`
  SELECT realisateur_id, COUNT(*) AS nombre_films
  FROM Film
  GROUP BY realisateur_id
`).all();
// [ { realisateur_id: 1, nombre_films: 5 }, { realisateur_id: 2, nombre_films: 3 } ]
```

`GROUP BY realisateur_id` → "fais un paquet par réalisateur, applique `COUNT(*)` à chaque paquet". C'est le `grouper_par_matiere` fait par la base.

**Autres exemples :**
```typescript
// Durée totale des films par réalisateur
db.prepare("SELECT realisateur_id, SUM(duree) AS total FROM Film GROUP BY realisateur_id").all();

// Note moyenne par matière (modèle notes)
db.prepare("SELECT matiere, AVG(note) AS moyenne FROM note GROUP BY matiere").all();

// Nombre de chansons par album (modèle musique)
db.prepare("SELECT album_id, COUNT(*) AS nb FROM Chanson GROUP BY album_id").all();
```

⚠️ **Règle :** avec `GROUP BY` + agrégation, le `SELECT` ne contient que la colonne groupée et les fonctions. (Si tu groupes par réalisateur, tu ne peux pas afficher le titre d'**un** film — il y en a plusieurs.)

---

## 9. HAVING — filtrer les GROUPES

`WHERE` filtre les **lignes** (avant le groupement). `HAVING` filtre les **groupes** (après).

```typescript
// Réalisateurs ayant fait PLUS DE 3 films
db.prepare(`
  SELECT realisateur_id, COUNT(*) AS n
  FROM Film
  GROUP BY realisateur_id
  HAVING COUNT(*) > 3
`).all();
```

On ne peut pas utiliser `WHERE COUNT(*) > 3` (le `WHERE` ne connaît pas les agrégats) → c'est `HAVING`.

| | filtre | quand |
|---|---|---|
| `WHERE` | les lignes | avant le GROUP BY |
| `HAVING` | les groupes | après le GROUP BY |

---

## 10. ORDER BY — trier

```typescript
db.prepare("SELECT titre, annee FROM Film ORDER BY annee").all();       // croissant (défaut)
db.prepare("SELECT titre, annee FROM Film ORDER BY annee DESC").all();   // décroissant
db.prepare("SELECT titre FROM Film ORDER BY titre").all();               // alphabétique
```

On peut trier sur une colonne calculée (via son `AS`) :
```typescript
db.prepare(`
  SELECT realisateur_id, COUNT(*) AS n
  FROM Film GROUP BY realisateur_id
  ORDER BY n DESC
`).all();   // du plus prolifique au moins prolifique
```

`LIMIT` pour le top N :
```typescript
db.prepare("SELECT titre, annee FROM Film ORDER BY annee DESC LIMIT 5").all();   // les 5 plus récents
```

---

## 11. AS — renommer

```typescript
"SELECT nom AS realisateur ..."          // renomme une colonne
"SELECT COUNT(*) AS total ..."           // nomme une colonne CALCULÉE (indispensable)
"SELECT Gender.name AS genre ..."        // dans une jointure
```

Côté front, on lit le nom du `AS` : `it.genre`, `it.total`, pas `it.name` / `it.COUNT(*)`.
⚠️ Ton **type TypeScript** doit matcher le nom du `AS`, pas celui de la colonne.

---

## 12. COMBINAISONS COMPLÈTES

### 12.1 — Classement (JOIN + GROUP BY + COUNT + AS + ORDER BY)
```typescript
db.prepare(`
  SELECT Realisateur.nom AS realisateur, COUNT(*) AS nombre_films
  FROM Film
  JOIN Realisateur ON Film.realisateur_id = Realisateur.id
  GROUP BY Realisateur.id
  ORDER BY nombre_films DESC
`).all();
// [ { realisateur: "Spielberg", nombre_films: 8 }, ... ]
```

### 12.2 — Montant total d'une commande (jonction AVEC attribut + SUM + calcul)
```typescript
db.prepare(`
  SELECT SUM(LigneCommande.quantite * Produit.prix) AS total
  FROM LigneCommande
  JOIN Produit ON LigneCommande.produit_id = Produit.id
  WHERE LigneCommande.commande_id = ?
`).get(commandeId);
// SUM(quantite × prix) = somme des sous-totaux de chaque ligne
```

### 12.3 — Détail d'une commande (ligne par ligne)
```typescript
db.prepare(`
  SELECT Produit.nom,
         LigneCommande.quantite,
         Produit.prix,
         (LigneCommande.quantite * Produit.prix) AS sous_total
  FROM LigneCommande
  JOIN Produit ON LigneCommande.produit_id = Produit.id
  WHERE LigneCommande.commande_id = ?
`).all(commandeId);
```

### 12.4 — Nombre de produits par catégorie, triés
```typescript
db.prepare(`
  SELECT Categorie.nom AS categorie, COUNT(*) AS nb_produits
  FROM Produit
  JOIN Categorie ON Produit.categorie_id = Categorie.id
  GROUP BY Categorie.id
  ORDER BY nb_produits DESC
`).all();
```

### 12.5 — Chiffre d'affaires par client
```typescript
db.prepare(`
  SELECT Client.nom AS client, SUM(LigneCommande.quantite * Produit.prix) AS ca
  FROM Client
  JOIN Commande      ON Commande.client_id = Client.id
  JOIN LigneCommande ON LigneCommande.commande_id = Commande.id
  JOIN Produit       ON LigneCommande.produit_id = Produit.id
  GROUP BY Client.id
  ORDER BY ca DESC
`).all();   // quadruple jointure + SUM + GROUP BY !
```

---

## 13. INTÉGRÉ DANS DES ROUTES FASTIFY

```typescript
// Distribution d'un film
fastify.get("/films/:id/acteurs", (request, reply) => {
  try {
    const { id } = request.params as { id: string };
    const acteurs = db.prepare(`
      SELECT Acteur.nom
      FROM Jouer JOIN Acteur ON Jouer.acteur_id = Acteur.id
      WHERE Jouer.film_id = ?
    `).all(id);
    reply.send(acteurs);
  } catch (error) {
    console.log(error);
    reply.code(500).send();
  }
});

// Total d'une commande
fastify.get("/commandes/:id/total", (request, reply) => {
  try {
    const { id } = request.params as { id: string };
    const r = db.prepare(`
      SELECT SUM(LigneCommande.quantite * Produit.prix) AS total
      FROM LigneCommande JOIN Produit ON LigneCommande.produit_id = Produit.id
      WHERE LigneCommande.commande_id = ?
    `).get(id);
    reply.send(r);
  } catch (error) {
    console.log(error);
    reply.code(500).send();
  }
});
```

---

## 14. ⚠️ PIÈGES À ÉVITER

| Piège | Conséquence | Solution |
|---|---|---|
| `JOIN` au lieu de `LEFT JOIN` | les lignes sans correspondance disparaissent | `LEFT JOIN` si tu veux tout garder |
| `COUNT` vs `SUM` confondus | compte des lignes ≠ additionne des valeurs | bien choisir |
| Oublier `AS` sur un calcul | colonne nommée `COUNT(*)` | toujours `AS` |
| Type TS ≠ nom du `AS` | `it.xxx` = undefined | calquer le type sur le `AS` |
| `WHERE COUNT(*) > 3` | erreur (WHERE ignore les agrégats) | `HAVING` |
| Croire que triple JOIN = 1 ligne/film | lignes multipliées par acteur | regrouper en JS |
| Ordre du `ON` censé compter | (faux) il est symétrique | écris dans le sens lisible |
| Concaténer une valeur dans le SQL | injection SQL | `?` paramétrique |

---

## 15. 🎯 LE FIL CONDUCTEUR

- **Chaque `JOIN` suit un trait du modèle entité-relations.**
  - relation 1,N → un `JOIN` direct via la clé étrangère.
  - relation N,N → deux `JOIN` qui passent par la table de jonction.
- **Agrégations :** `COUNT`/`SUM`/`AVG` seuls → résultat global ; **avec `GROUP BY`** → résultat par catégorie.
- `WHERE` filtre les lignes, `HAVING` filtre les groupes, `ORDER BY` trie, `AS` nomme.
- Quand une jointure "multiplie" les lignes, on **regroupe côté JS** (comme `grouper_par_matiere`).

**On "marche" sur le schéma, table après table, en reliant chaque nouvelle table à une précédente avec `ON`.**
