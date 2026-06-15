# 🎚️ FICHE COMPLÈTE — La balise `<select>` (HTML + TypeScript)

> **Idée maîtresse :** chaque `<option>` a **deux faces** — le **texte** (ce que l'humain voit) et la **`value`** (ce que le code récupère). On **affiche par le nom, on identifie par l'id**. Tout le reste découle de ça.

---

## 1. Anatomie de base

```html
<select id="genre">
  <option value="1">Roman</option>
  <option value="2">Science-fiction</option>
  <option value="3">Biographie</option>
</select>
```

- `<select>` = le conteneur (la liste déroulante).
- `<option>` = une entrée.
- texte entre les balises (`Roman`) = **affiché**.
- attribut `value` (`"1"`) = **récupéré par le code**.

**Sans `value`**, la valeur devient le texte :
```html
<option>Roman</option>   <!-- value implicite = "Roman" -->
```

---

## 2. Lire la valeur choisie (TypeScript)

```typescript
const select = document.getElementById("genre") as HTMLSelectElement;
const valeur = select.value;        // "2" si Science-fiction est choisi
const id = Number(select.value);     // 2  ← conversion pour un id
```

⚠️ **Trois règles d'or :**
1. Caster en `HTMLSelectElement` (pas `HTMLInputElement`).
2. `.value` est **TOUJOURS une string**, même `"2"` → `Number(...)` pour un id.
3. `.value` renvoie la value de l'option **actuellement sélectionnée**.

---

## 3. Remplir un `<select>` dynamiquement (depuis une API)

Le `<select>` est **vide** dans le HTML, le JS le remplit.

```html
<select id="genre"></select>
```

```typescript
async function charger_genres() {
  const response = await fetch("http://localhost:8080/Genders");
  if (!response.ok) return;
  const genres = await response.json() as Gender[];

  const select = document.getElementById("genre") as HTMLSelectElement | null;
  if (select === null) return;
  select.innerHTML = "";   // vider avant de remplir (évite les doublons)

  for (const g of genres) {
    const option = document.createElement("option");
    option.value = String(g.id);   // l'id (caché) → pour identifier
    option.textContent = g.name;    // le nom (affiché) → pour l'humain
    select.appendChild(option);
  }
}
```

⚠️ Ton API doit renvoyer `id` ET `name`. Si elle ne renvoie que `name`, `g.id` est `undefined` → `value="undefined"` → `Number(select.value)` = `NaN`.

---

## 4. Réagir au choix : l'événement `change`

```typescript
const select = document.getElementById("filtre") as HTMLSelectElement;
select.addEventListener("change", () => {
  const id = select.value;
  console.log("genre choisi :", id);
  filtrer(id);
});
```

- Pour un **bouton** → `click`.
- Pour un **select** → `change` (déclenché quand la sélection change).

---

## 5. Propriétés utiles

```typescript
const select = document.getElementById("genre") as HTMLSelectElement;

select.value;                                       // value de l'option choisie
select.selectedIndex;                                // position (0,1,2...) de l'option choisie
select.options[0].textContent;                       // texte de la 1re option
select.options[select.selectedIndex].textContent;    // TEXTE de l'option choisie (le nom affiché)
select.value = "2";                                  // FORCER la sélection depuis le code
```

Exemple : afficher le **nom** choisi (pas la value) :
```typescript
const nomChoisi = select.options[select.selectedIndex].textContent;
console.log("Vous avez choisi :", nomChoisi);   // "Roman"
```

---

## 6. L'option "tout" (sentinelle vide) — pour un filtre

```html
<select id="filtre-genre">
  <option value="">Tous les genres</option>
</select>
```

```typescript
select.addEventListener("change", () => {
  const genreId = select.value;
  if (genreId === "") {
    get_book();   // value vide → aucun filtre → tous les livres
  } else {
    fetch(`http://localhost:8080/Books?genre_id=${genreId}`)
      .then(r => r.json())
      .then((livres: Book[]) => display(livres));
  }
});
```

La chaîne vide `""` = sentinelle "aucun filtre".

---

## 7. `selected` : pré-sélectionner une option

```html
<select id="filtre">
  <option value="" selected>Tous les genres</option>   <!-- affiché par défaut -->
  <option value="1">Roman</option>
</select>
```

---

## 8. ⭐ PLUSIEURS SELECTS (listes déroulantes multiples)

### 8.1 — Plusieurs selects INDÉPENDANTS sur un même formulaire

Un livre qu'on ajoute avec **genre + langue + format**, chacun dans sa liste :

```html
<select id="select-genre"></select>
<select id="select-langue"></select>
<select id="select-format"></select>
<button id="btn-ajouter">Ajouter</button>
```

```typescript
// Remplir CHAQUE select (depuis sa propre source)
async function charger_listes() {
  remplir_select("select-genre",  await getJSON("/Genders"));
  remplir_select("select-langue", await getJSON("/Langues"));
  remplir_select("select-format", await getJSON("/Formats"));
}

// Fonction réutilisable : remplit n'importe quel select
function remplir_select(idSelect: string, items: { id: number; name: string }[]) {
  const select = document.getElementById(idSelect) as HTMLSelectElement | null;
  if (select === null) return;
  select.innerHTML = "";
  for (const it of items) {
    const option = document.createElement("option");
    option.value = String(it.id);
    option.textContent = it.name;
    select.appendChild(option);
  }
}

async function getJSON(route: string) {
  const r = await fetch(`http://localhost:8080${route}`);
  return await r.json();
}
```

```typescript
// Lire les TROIS selects à la soumission
function ajouter() {
  const genre_id  = Number((document.getElementById("select-genre")  as HTMLSelectElement).value);
  const langue_id = Number((document.getElementById("select-langue") as HTMLSelectElement).value);
  const format_id = Number((document.getElementById("select-format") as HTMLSelectElement).value);

  console.log({ genre_id, langue_id, format_id });
  // → envoyer tout ça dans le body du POST
}
```

**Clé :** une **fonction générique** `remplir_select(id, items)` évite de copier-coller le même code 3 fois. Tu l'appelles une fois par liste.

---

### 8.2 — Plusieurs selects EN CASCADE (dépendants)

Le choix du 1er select **détermine** le contenu du 2e.
Exemple : choisir un **pays** → remplir la liste des **villes de ce pays**.

```html
<select id="select-pays"></select>
<select id="select-ville"></select>
```

```typescript
// 1. Remplir le select des pays au chargement
async function charger_pays() {
  const pays = await getJSON("/Pays");
  remplir_select("select-pays", pays);
  // charger les villes du pays affiché par défaut
  const premierPays = (document.getElementById("select-pays") as HTMLSelectElement).value;
  charger_villes(Number(premierPays));
}

// 2. Quand le pays change → recharger les villes
function brancher_cascade() {
  const selectPays = document.getElementById("select-pays") as HTMLSelectElement;
  selectPays.addEventListener("change", () => {
    const paysId = Number(selectPays.value);
    charger_villes(paysId);   // ← le 2e select dépend du 1er
  });
}

// 3. Remplir le select des villes du pays choisi
async function charger_villes(paysId: number) {
  const villes = await getJSON(`/Villes?pays_id=${paysId}`);   // filtré par pays
  remplir_select("select-ville", villes);
}
```

**Mécanisme :** le `change` du 1er select déclenche le remplissage du 2e. Côté backend, `/Villes?pays_id=...` renvoie seulement les villes du pays demandé (`WHERE pays_id = ?`).

---

### 8.3 — Plusieurs lignes, chacune avec son select (générer N selects)

Exemple : un tableau où chaque ligne a son propre select de statut.

```typescript
function afficher_commandes(commandes: Commande[], statuts: Statut[]) {
  const conteneur = document.getElementById("liste") as HTMLElement;
  conteneur.innerHTML = "";

  for (const cmd of commandes) {
    const ligne = document.createElement("div");
    ligne.textContent = `Commande ${cmd.id} — `;

    // un select PAR commande
    const select = document.createElement("select");
    for (const s of statuts) {
      const option = document.createElement("option");
      option.value = String(s.id);
      option.textContent = s.name;
      if (s.id === cmd.statut_id) option.selected = true;  // pré-sélectionner le statut actuel
      select.appendChild(option);
    }
    // chaque select connaît SA commande (closure)
    select.addEventListener("change", () => {
      changer_statut(cmd.id, Number(select.value));
    });

    ligne.appendChild(select);
    conteneur.appendChild(ligne);
  }
}
```

**Clé :** chaque select est créé dans la boucle, donc il "capture" `cmd.id` (closure). `option.selected = true` pré-sélectionne le statut courant de chaque commande.

---

## 9. Exemple pratique COMPLET — formulaire d'ajout multi-listes

```html
<h2>Ajouter un livre</h2>
<input type="text"   id="titre"  placeholder="Titre" />
<input type="text"   id="auteur" placeholder="Auteur" />
<input type="number" id="annee"  placeholder="Année" />
<select id="select-genre"></select>
<select id="select-langue"></select>
<button id="btn-ajouter">Ajouter</button>
```

```typescript
window.addEventListener("load", () => {
  charger_listes();   // remplit les 2 selects
  brancher_ajout();
});

async function charger_listes() {
  remplir_select("select-genre",  await getJSON("/Genders"));
  remplir_select("select-langue", await getJSON("/Langues"));
}

function brancher_ajout() {
  const btn = document.getElementById("btn-ajouter");
  btn?.addEventListener("click", () => {
    const titre     = (document.getElementById("titre")  as HTMLInputElement).value;
    const auteur    = (document.getElementById("auteur") as HTMLInputElement).value;
    const annee     = Number((document.getElementById("annee") as HTMLInputElement).value);
    const genre_id  = Number((document.getElementById("select-genre")  as HTMLSelectElement).value);
    const langue_id = Number((document.getElementById("select-langue") as HTMLSelectElement).value);

    add_Book({ titre, auteur, annee, genre_id, langue_id });
  });
}

async function add_Book(livre: object) {
  const r = await fetch("http://localhost:8080/Books", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(livre)
  });
  if (r.ok) get_book();
}
```

---

## 10. Bonus — sélection MULTIPLE (`multiple`)

Permet de choisir **plusieurs** options à la fois.

```html
<select id="select-tags" multiple>
  <option value="1">Action</option>
  <option value="2">Aventure</option>
  <option value="3">Drame</option>
</select>
```

```typescript
const select = document.getElementById("select-tags") as HTMLSelectElement;

// Récupérer TOUTES les options sélectionnées
const choisis = Array.from(select.selectedOptions).map(o => Number(o.value));
console.log(choisis);   // [1, 3] si Action et Drame sont sélectionnés
```

`select.selectedOptions` = la liste des options choisies ; `Array.from(...).map(...)` les transforme en tableau d'ids.

---

## 11. ⚠️ PIÈGES À ÉVITER

| Piège | Conséquence | Solution |
|---|---|---|
| `.value` traité comme un nombre | comparaisons/calculs faux | `Number(select.value)` |
| API renvoie `name` sans `id` | `value="undefined"` → `NaN` | `SELECT id, name` côté back |
| Caster en `HTMLInputElement` | propriétés select manquantes | `HTMLSelectElement` |
| Oublier `select.innerHTML = ""` | options dupliquées à chaque rechargement | vider avant de remplir |
| Pas de garde `null` | crash `Cannot set ... of null` | `if (select === null) return` |
| Select vide non géré | `Number("")` = `0` → id invalide envoyé | tester `if (value === "")` |
| `<select>` rempli après le `<script>` | options jamais ajoutées | script en dernier, ou `load` |

---

## 12. 🎯 LE FIL CONDUCTEUR

```
API (.all → liste d'objets avec id)
   ↓ fetch GET
createElement("option") → option.value = id | option.textContent = nom
   ↓ appendChild
[utilisateur choisit]
   ↓
select.value → l'id (string)
   ↓ Number(...)
envoyé dans le body du POST
```

**L'id que tu récupères au clic est exactement celui que tu as rangé dans `value` au remplissage.** Le `<select>` est un casier : tu déposes l'id (createElement), tu le ressors (select.value). On n'a jamais besoin de "chercher l'id par le nom" — il est dans l'option depuis le début.
