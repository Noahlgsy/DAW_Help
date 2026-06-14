# 🎯 FICHE DE RÉVISION — Développement d'applications web
### TypeScript · Node.js · Fastify · SQLite · HTML/CSS

> **Comment utiliser cette fiche :** les sections 1 et 2 sont les pièges qui reviennent le plus — relis-les en dernier juste avant l'épreuve. La section 11 est ton "SOS débogage" : quand une erreur tombe, va directement la chercher là.

---

## 1. ⚙️ LE CYCLE DE TRAVAIL (à ne JAMAIS oublier)

Le bug n°1 de tout le contrôle : modifier le code et oublier de le réappliquer.

```
modifier src/*.ts  →  npx tsc (dans api/)  →  Ctrl+C sur le serveur  →  node dist/index.js
```

- **Node exécute le `.js` du dossier `dist`, jamais ton `.ts`.** Si tu ne transpiles pas, le serveur tourne avec l'ancien code.
- **Transpiler ne suffit pas : il faut REDÉMARRER le serveur** (Ctrl+C puis relancer). Un processus node garde en mémoire le code chargé au démarrage.
- **Astuce :** lance `npx tsc -w` dans un terminal dédié → transpilation automatique à chaque sauvegarde. Il ne reste qu'à redémarrer le serveur.

➡️ **Symptôme typique d'un oubli :** `Route GET:/... not found` (404) alors que la route existe bien dans ton code.

---

## 2. 🔥 LES ERREURS RÉCURRENTES (le top des pièges)

### Piège A — « Préparer/référencer ≠ exécuter » (les parenthèses `()`)
Une fonction sans `()` est juste *référencée*, pas *exécutée*.

```typescript
boucle;            // ❌ ne fait rien
boucle();          // ✅

query.all;         // ❌ référence la méthode
query.all();       // ✅ exécute

const v = db.prepare("SELECT ...");   // ❌ v = objet requête (toujours truthy !)
const v = db.prepare("SELECT ...").get(mail);  // ✅ v = résultat ou undefined
```

### Piège B — `reply.send()` N'INTERROMPT PAS la fonction
Après chaque réponse dans un cas particulier (erreur, sortie anticipée), il FAUT un `return`.

```typescript
if (existant) {
  reply.code(409).send();
  return;            // ✅ SANS ce return, le code continue et envoie une 2e réponse → crash
}
```

### Piège C — Double `reply.send()`
Une requête HTTP = **une seule** réponse. `reply.send(data)` envoie déjà un code 200.

```typescript
reply.send(results);       // ✅ 200 implicite
reply.code(200).send();    // ❌ NE JAMAIS ajouter ça après → "Reply already sent"
```

### Piège D — Une route doit TOUJOURS répondre
`console.log` affiche dans **le terminal du serveur**, ce n'est PAS une réponse pour le client. Sans `reply`, le navigateur attend dans le vide.

### Piège E — On envoie l'objet requête au lieu du résultat
```typescript
query.all(id);          // ❌ résultat exécuté mais JETÉ
reply.send(query);      // ❌ envoie l'objet "requête préparée", pas les données
// ✅ correct :
const notes = query.all(id);
reply.send(notes);
```

### Piège F — Backticks vs guillemets pour les variables dans une URL
```typescript
fetch("http://localhost:8080/tasks/${id}")    // ❌ texte brut "${id}"
fetch(`http://localhost:8080/tasks/${id}`)     // ✅ backticks → interpolation
fetch("http://localhost:8080/Student/:id/notes")  // ❌ :id n'existe QUE côté serveur
fetch(`http://localhost:8080/Student/${id}/notes`) // ✅ vraie valeur côté client
```

---

## 3. 🖥️ BACKEND — checklist d'une route propre

Structure type d'une route avec gestion d'erreurs :

```typescript
fastify.post("/api/etudiants/:id/notes", (request, reply) => {
  try {
    const { id } = request.params as { id: string };       // ← donnée de l'URL
    const { matiere, note } = request.body as Notes;        // ← donnée du body

    // 1. Vérifier l'existence dans la BONNE table
    const etudiant = db.prepare("SELECT id FROM Etudiant WHERE id = ?").get(id);
    if (!etudiant) {
      reply.code(404).send();
      return;                    // ← TOUJOURS return après un reply de garde
    }

    // 2. Agir
    db.prepare("INSERT INTO note (matiere, note, etudiant_id) VALUES (?, ?, ?)")
      .run(matiere, note, id);

    reply.code(201).send();      // ← code adapté (création = 201)
  } catch (error) {
    console.log(error);          // ← visible dans le terminal serveur
    reply.code(500).send();      // ← erreur technique imprévue
  }
});
```

**D'où vient chaque donnée (à mémoriser !) :**
| Source | Comment la lire | Exemple |
|---|---|---|
| URL (paramètre `:id`) | `request.params` | `/Student/:id` |
| Body JSON | `request.body` | POST/PUT |
| Query string (`?matiere=...`) | `request.query` | filtre |
| En-tête (token) | `request.headers.authorization` | auth |

⚠️ **Vérifier l'existence dans la BONNE table.** Chercher un étudiant dans la table `note` échoue pour un étudiant sans note → faux 404/409. Toujours vérifier dans `Etudiant`.

---

## 4. 📟 CODES HTTP (les connaître par cœur)

| Code | Sens | Quand |
|---|---|---|
| **200** | OK (lecture réussie) | GET qui renvoie des données |
| **201** | Créé | POST qui crée une ressource |
| **204** | OK, **sans body** | succès sans contenu (ne JAMAIS mettre de `.send("texte")` avec) |
| **404** | Not Found | ressource **introuvable** |
| **409** | Conflict | la ressource **existe déjà** (doublon) |
| **401** | Unauthorized | token **invalide/absent** |
| **500** | Erreur serveur | exception, erreur SQL (dans le `catch`) |

- `response.ok` est vrai pour **200–299** (donc 200, 201, 204 inclus).
- ⚠️ **404 ≠ 409** : "introuvable" vs "déjà existant". Ne pas confondre.
- Les textes dans la doc ("OK : account created") sont des **descriptions**, PAS des bodies à envoyer.

---

## 5. 🗄️ SQL / better-sqlite3

### Les 3 méthodes — choisir la bonne
| Méthode | Pour quel SQL | Renvoie |
|---|---|---|
| `.all(...)` | `SELECT` (plusieurs lignes) | **tableau** d'objets |
| `.get(...)` | `SELECT` (une ligne) | **un objet** ou `undefined` |
| `.run(...)` | `INSERT` / `UPDATE` / `DELETE` | compte-rendu : `changes`, `lastInsertRowid` |

> Question à se poser : **lire ou modifier ?** Lire plusieurs → `.all` · lire un / vérifier existence → `.get` · modifier → `.run`.

### Requêtes paramétriques `?` (sécurité + fonctionnement)
```typescript
db.prepare("SELECT * FROM note WHERE etudiant_id = ?").all(id);   // ✅
```
- **SQLite ne voit PAS tes variables TypeScript.** Dans la requête, un mot = un nom de colonne.
- `WHERE col = etudiant_id` → SQLite cherche une colonne `etudiant_id` (faux !).
- Le `?` est le **pont** : on passe la *valeur* dans `.all()/.get()/.run()`, dans l'ordre.
- **Protège des injections SQL** → ne JAMAIS concaténer (`"... = " + valeur`).

### Pièges SELECT
- `SELECT *` → renvoie **tout**, y compris les mots de passe → fuite de données. **Sélectionner explicitement les colonnes.**
- `SELECT nom` seul → le front manque `id` et `prenom`. Renvoyer ce dont le front a besoin (`SELECT id, nom, prenom`).

### Autres réflexes SQL
- `.get()` renvoie **`undefined`** (pas `null`) si rien trouvé → `if (x === null)` ne marche jamais ; utiliser `if (!x)`.
- L'**id auto-incrément** est créé par la base à l'`INSERT` → on le récupère avec `result.lastInsertRowid`.
- La **clé étrangère** (`etudiant_id`) vit du côté « plusieurs » (sur `note`).
- **Unicité composée** : un doublon = `WHERE nom = ? AND prenom = ?` (la paire, pas un seul champ).

---

## 6. 🔐 JWT / AUTHENTIFICATION

**Analogie du bracelet de festival :** on montre sa carte d'identité UNE fois à l'entrée (`/auth`), on reçoit un bracelet (token), on le présente partout ensuite (routes protégées).

```typescript
// À la connexion : on FABRIQUE le token avec l'id dedans
const token = JWT.sign({ id: user.id }, privateKey);   // pas le password !
reply.code(200).send(token);                            // ← renvoyer le token au client

// Sur une route protégée : on VÉRIFIE et on extrait
const token = request.headers.authorization?.replace("Bearer ", "");  // ⚠️ espace !
if (!token) { reply.code(401).send(); return; }

let from_user: number;
try {
  const payload = JWT.verify(token, privateKey) as { id: number };
  from_user = payload.id;        // ⚠️ .id, pas payload entier
} catch (error) {
  reply.code(401).send();        // ⚠️ verify LÈVE une exception → 401 ici
  return;                        // ← le return prouve aussi à TS que from_user est défini
}
```

**À retenir absolument :**
- `JWT.sign` = **fabrique** · `JWT.verify` = **vérifie + extrait**.
- Le token n'est **PAS chiffré**, juste signé → ne JAMAIS y mettre le mot de passe.
- Le token n'est **PAS stocké en base** → toute l'info (l'id) est dedans.
- `JWT.verify` sur un token invalide **LÈVE une exception** (ne renvoie pas `false`) → il faut un **try/catch dédié** qui renvoie **401**, séparé du try/catch métier (qui renvoie 500).
- `.replace("Bearer ", "")` → l'**espace** est obligatoire, sinon "invalid token".
- L'id de l'émetteur vient du **token** (sécurisé) ; le destinataire vient du **body**.

---

## 7. 🌐 FRONTEND (DOM, fetch, affichage)

### Le pattern de base (identique partout)
```typescript
async function action(...) {
  const response = await fetch(url, { method, headers, body });
  if (response.ok) {
    rafraichir();   // re-télécharge et réaffiche → la BDD est la SEULE source de vérité
  }
}
```
- Ne jamais "gérer la liste" à la main côté front : on agit → si OK → on recharge.
- ⚠️ POST/PUT/DELETE avec body → **toujours** `headers: { "Content-Type": "application/json" }`, sinon le body n'est pas lu côté serveur.

### Afficher dynamiquement (createElement)
```typescript
function display(items: Student[]) {
  const conteneur = document.getElementById("Eleves");
  if (conteneur === null) return;     // ⚠️ vérifier null
  conteneur.innerHTML = "";           // vider avant de remplir
  for (const it of items) {
    const p = document.createElement("p");
    p.textContent = `${it.nom} ${it.prenom}`;   // ⚠️ écrire dans le NOUVEL élément
    p.addEventListener("click", () => get_Notes(it.id));  // id capturé par la closure
    conteneur.appendChild(p);          // ⚠️ SANS appendChild → rien ne s'affiche
  }
}
```

**Pièges fréquents du DOM :**
- `conteneur.textContent = x` dans une boucle → **écrase** à chaque tour (seul le dernier reste). Écrire dans le **nouvel** élément, pas le conteneur.
- Oublier `appendChild` → l'élément existe en mémoire mais **invisible**.
- `innerHTML +=` **efface les `addEventListener`** déjà posés → pour de l'interactif, utiliser `createElement` + `addEventListener`.
- Appeler la **bonne** fonction d'affichage (`display_notes` pour les notes, pas `display` qui attend des étudiants).
- Oublier d'appeler le `GET` au chargement (`window.addEventListener("load", ...)`).
- Un `clear_page` qui vide **trop** (efface la liste des étudiants quand on affiche les notes).

### Liste déroulante `<select>` : identifier vs afficher
```typescript
option.value = String(etu.id);                  // l'ID (caché) → pour identifier
option.textContent = `${etu.nom} ${etu.prenom}`; // le NOM → pour afficher
// au choix :
const id = Number(select.value);                 // ⚠️ value est une STRING → Number()
```
> **Règle d'or : on identifie par l'id, on affiche par le nom.** Jamais chercher par nom (homonymes, fautes de frappe).

⚠️ Tout ce qui vient du DOM (`input.value`, `select.value`) est une **string** → `Number(...)` pour un nombre.

---

## 8. 🎨 HTML / CSS À NE PAS OUBLIER

### Tableau lisible
```css
table { border-collapse: collapse; width: 100%; }  /* ⚠️ collapse = pas de double bordure */
th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
thead { background-color: #f0f0f0; }
```

### Colonnes côte à côte → Flexbox
```css
#conteneur { display: flex; gap: 16px; }   /* ⚠️ flex = enfants en ligne (côte à côte) */
.colonne   { flex: 1; }                      /* chaque colonne = part égale */
```
> Par défaut les `<div>` s'empilent **verticalement**. `display: flex` sur le parent → ses enfants se placent **horizontalement**.

### Autres réflexes
- `cursor: pointer` → indique qu'un élément est cliquable.
- **HTML valide** : pas de `<div>` dans un `<p>` (le navigateur casse la mise en page). Conteneur de blocs = `<div>`.
- **Casse sensible** : `getElementById("Eleves")` doit matcher EXACTEMENT l'`id` du HTML (`Eleves` ≠ `eleves`).

---

## 9. 🧩 ERREURS TYPESCRIPT COURANTES

| Erreur | Cause | Solution |
|---|---|---|
| `'string \| undefined' is not assignable to 'string'` | valeur potentiellement absente (query param, fonction sans `return` dans tous les cas) | gérer le cas absent : `if (x)` ou `return ""` en secours |
| `Variable 'x' is used before being assigned` | `x` assigné dans un `try`, utilisé après, mais le `catch` ne stoppe pas | ajouter `return` dans le `catch` |
| `'user' refers to a value, but is being used as a type` | confusion **valeur** (`const`) / **type** (`type`) | définir un `type User = {...}`, pas un objet |

**Les types vivent à la compilation, les valeurs à l'exécution.** `as Type` = "fais-moi confiance sur la forme". Les types sont effacés du `.js` final.

---

## 10. 🔧 ENVIRONNEMENT (modules, base, paquets)

### Imports entre fichiers : extension `.js` !
```typescript
import db from "./database.js";   // ✅ .js même si le fichier source est database.ts
```
> Au lancement, il n'y a plus que des `.js` dans `dist`. L'import doit pointer le fichier **tel qu'il existera à l'exécution**.
- `export default db;` → import **sans accolades** : `import db from "..."`.
- `export const x` (nommé) → import **avec accolades** : `import { x } from "..."`.

### Créer / ouvrir la base SQLite
```typescript
import fs from "fs";
fs.mkdirSync("database", { recursive: true });   // ⚠️ better-sqlite3 crée le FICHIER, pas le DOSSIER
const db = Sqlite3("database/app.sqlite");
db.pragma("foreign_keys = ON");                   // ⚠️ SQLite IGNORE les FK par défaut !
db.exec(`CREATE TABLE IF NOT EXISTS ...`);        // IF NOT EXISTS = script rejouable
```
- **Chemin relatif** = résolu depuis le dossier où on lance `node`, PAS depuis le fichier source. Depuis `api/dist` → `../database/db.sqlite`.

### Erreurs d'environnement fréquentes
| Erreur | Cause | Solution |
|---|---|---|
| `MODULE_NOT_FOUND` | paquet non installé / installé dans le mauvais dossier | `npm i <paquet>` **dans `api/`** |
| `ERR_MODULE_NOT_FOUND` | import sans extension ou en `.ts` | mettre `.js` dans l'import |
| `directory does not exist` | dossier de la base absent | `mkdir database` ou `fs.mkdirSync(..., {recursive:true})` |
| `unable to open database file` | mauvais chemin relatif selon le dossier de lancement | ajuster le chemin / le point de lancement |

### CORS
```typescript
import cors from "@fastify/cors";
fastify.register(cors);   // autorise le front (autre origine) à appeler l'API
```

---

## 11. 🚑 SOS DÉBOGAGE — « j'ai une erreur, je fais quoi ? »

### ❌ Erreur 500 (Internal Server Error)
➡️ **Lis le terminal du SERVEUR** (ton `console.log(error)` y écrit la cause exacte). Causes fréquentes : nom de colonne faux dans une requête, `JWT.verify` dans le mauvais try/catch, valeur `undefined` insérée, contrainte SQL violée.

### ❌ Erreur 404 "Route not found"
➡️ Le serveur ne connaît pas la route → **pas retranspilé / pas redémarré** (vérifier `dist/index.js`), OU **URL mal orthographiée** (casse : `/Student` ≠ `/student`), OU serveur zombie sur le port.

### ❌ Erreur 409 inattendue (sur l'ajout)
➡️ Vérification d'existence dans la **mauvaise table** (chercher dans `Etudiant`, pas `note`). + Ne pas confondre avec un vrai doublon.

### ❌ Le front n'affiche rien
➡️ Ouvrir la **console (F12)** : le `console.log` des données affiche-t-il un tableau ? Les objets ont-ils bien un `id` ?
➡️ Curseur "main" au survol ? → les éléments sont bien créés. Sinon → `display` sort en `return` (élément `getElementById` introuvable, casse de l'`id`).
➡️ Onglet **Network (F12)** : une requête part-elle ? Vers quelle URL ? `/Student/undefined/notes` → l'id manque (souvent : serveur pas redémarré après `SELECT id, ...`).

### ❌ `response.ok` est faux côté front
➡️ Ajouter temporairement `console.log("code:", response.status)`. 404 = URL ; 500 = erreur serveur (lire le terminal serveur) ; rien = pas de réponse (oubli de `reply`).

### ❌ "invalid token"
➡️ `.replace("Bearer ", "")` avec l'**espace** ; régénérer un token frais (reconnexion) si l'ancien a été signé autrement.

### ❌ Rien ne se passe au clic / à l'appel
➡️ Oubli des **parenthèses** `()` (référence au lieu d'exécuter), ou `reply` sans `return` qui casse le flux.

---

## 12. 💡 CONCEPTS CLÉS (rappels express)

- **Full-stack = 3 couches :** Frontend (navigateur, HTML/CSS/JS) ↔ Backend (serveur Node/Fastify, routes, logique) ↔ Base de données (SQLite, mémoire durable). Elles communiquent en **HTTP** ; une API qui respecte les conventions (méthode = action, URL = ressource, code = résultat) est **REST**.
- **Node.js** = moteur JavaScript hors du navigateur (exécute du JS côté serveur, accède aux fichiers/réseau, écoute un port). **npm** = gestionnaire de paquets.
- **TypeScript** = JavaScript + types vérifiés **avant** l'exécution. Doit être **transpilé** en JS (les types sont effacés). Filet de sécurité pour le dev, pas une protection à l'exécution.
- **Fastify ≠ Node** : Fastify est une bibliothèque (routes faciles) par-dessus Node. **Express** = équivalent.
- **La BDD est la seule source de vérité** : le front ne stocke rien de durable → après une action, on recharge.
- **CRUD ↔ HTTP ↔ SQL :** Create→POST→INSERT · Read→GET→SELECT · Update→PUT→UPDATE · Delete→DELETE→DELETE.

### Méthode pour un Modèle Entité-Relations (MCD)
1. **Les noms** du sujet → les **entités** (étudiant, note).
2. **Les verbes/actions** → les **relations** (un étudiant *possède* des notes).
3. « **Combien de X pour un Y ?** » dans les **deux sens** → la **cardinalité**.
4. La cardinalité donne le placement de la **clé étrangère** : côté « plusieurs ».
   - Ex : 1 étudiant → N notes ; 1 note → 1 étudiant ⇒ `etudiant_id` sur `note`.

### Sécurité (mots de passe — bonus du sujet)
- Ne JAMAIS stocker en clair. Hacher (SHA-256 via Web Crypto, sans dépendance).
- **Salage** : ajouter une valeur unique par utilisateur avant de hacher → deux mots de passe identiques donnent des hash différents.

---

## 13. ✅ CHECKLIST FINALE AVANT DE RENDRE

- [ ] Chaque route est dans un **try/catch** (→ 500).
- [ ] Chaque `reply` de garde est suivi d'un **`return`**.
- [ ] **Un seul** `reply.send()` par chemin d'exécution.
- [ ] Chaque route **répond** dans tous les cas.
- [ ] Bons **codes HTTP** (201 création, 404 introuvable, 409 doublon...).
- [ ] Requêtes **paramétriques** (`?`) partout, jamais de concaténation.
- [ ] `SELECT` ne renvoie **que les colonnes utiles** (id inclus pour le front).
- [ ] Vérifications d'existence dans la **bonne table**.
- [ ] URLs front en **backticks** avec `${id}`.
- [ ] `Content-Type: application/json` sur les POST/PUT avec body.
- [ ] `appendChild` présent dans les boucles d'affichage.
- [ ] `db.pragma("foreign_keys = ON")` activé.
- [ ] Imports en **`.js`**.
- [ ] Code **retranspilé** et serveur **redémarré** avant le test final.
- [ ] (Sujet notes) **filtre par matière** (`?matiere=`) + **calcul de la moyenne** dans la réponse GET.
- [ ] README + commits Git réguliers.
