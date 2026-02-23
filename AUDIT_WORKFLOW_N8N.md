# Audit Complet — Workflow N8N "Transfert automatique B2B → B2C (Candidat formation initiale)"

> **Date de l'audit** : 23/02/2026
> **Workflow ID** : `oYp95mhKJjFJXXOx`
> **Statut** : Actif en production (trigger quotidien 16h20)
> **Incident déclencheur** : Un lead sans adresse e-mail a crashé l'intégralité du flow

---

## 1. Cartographie du workflow actuel

### 1.1 Vue d'ensemble

Le workflow s'exécute chaque jour à 16h20. Il lit les leads depuis un Google Sheet (onglet `PROCESSED_B2C`), filtre ceux qui n'ont pas encore été transférés vers B2C, formate leurs données (téléphone, civilité, dates), met à jour le sheet, génère un fichier CSV et l'envoie par e-mail.

### 1.2 Chaîne complète des 12 nodes

```
Schedule Trigger (16h20)
  → Date & Time (⚠️ NE FAIT RIEN — config vide)
    → Get row(s) in sheet (lit TOUTES les lignes de PROCESSED_B2C)
      → If ("Date de transfer vers B2C" est vide ?)
        ├─ TRUE :
        │   → Code in JavaScript (formate téléphone → +33)
        │     → Code in JavaScript1 (formate civilité → M./Mme)
        │       → Append or update row in sheet (1ère écriture Sheet, match sur E-mail)
        │         → fonction utilitaire pour formater une Date (formate dates JJ/MM/AAAA)
        │           → Append or update row in sheet1 (2ème écriture Sheet, MÊME onglet, match E-mail)
        │             → Edit Fields (prépare champs CSV avec noms courts)
        │               → Convert to File (CSV délimiteur ;)
        │                 → Send a message (Gmail avec pièce jointe)
        └─ FALSE : rien (déjà traité)
```

### 1.3 Source de données

- **Document** : "Mapping Formation CFA" (`115qZE2TksAKcgm__MpZJ1KR957NOcQxwEehDRB-s5m4`)
- **Onglet** : `PROCESSED_B2C` (gid `58518804`)
- **Credentials Google Sheets** : `Google Sheets account 3` (QadzahQ1h7wOb4CD)
- **Credentials Gmail** : `Gmail account 2` (o0zR7wupxxmuuaf3)

---

## 2. Problèmes identifiés (14 issues)

### Synthèse

| Sévérité | Nombre | Description |
|----------|--------|-------------|
| **P0 — Critique** | 3 | Bugs bloquants en production |
| **P1 — Haut** | 4 | Performance, fiabilité, maintenabilité |
| **P2 — Moyen** | 7 | Robustesse, qualité du code |
| **Total** | **14** | |

---

### 2.1 P0 — CRITIQUES (bloquent le flow en production)

---

#### ISSUE #1 : Aucune validation E-mail avant les nodes Google Sheets

- **Nodes concernés** : `Append or update row in sheet`, `Append or update row in sheet1`
- **Sévérité** : P0 — CRITIQUE

**Description :**
Les deux nodes Google Sheets utilisent `E-mail` comme colonne de matching (`matchingColumns: ["E-mail"]`). Quand un lead arrive sans adresse e-mail (champ vide ou `null`), le node Google Sheets tente un `appendOrUpdate` avec une clé de matching vide, ce qui provoque une erreur API Google.

**Impact en production :**
- Le node crashe immédiatement
- **Tous les items suivants dans le batch ne sont jamais traités** (pas de `continueOnFail`)
- Le flow entier s'arrête — aucun lead n'est transféré ce jour-là
- Aucune alerte n'est envoyée
- **C'est exactement le bug qui s'est produit récemment**

**Correction :**
Ajouter un node IF après le filtre "Date de transfer vide" qui vérifie :
1. `E-mail` n'est pas vide/null/undefined
2. `E-mail` contient au minimum un `@`

Les leads invalides doivent être redirigés vers un onglet `REJECTS` pour traçabilité.

---

#### ISSUE #2 : Zéro error handling sur l'ensemble du workflow

- **Nodes concernés** : Tous (12 nodes)
- **Sévérité** : P0 — CRITIQUE

**Description :**
Aucun node du workflow n'a :
- `continueOnFail: true` (pour que les items suivants continuent même si un item échoue)
- Aucun Error Trigger / Error Workflow configuré
- Aucun try/catch dans les nodes Code

**Impact en production :**
Un seul item en erreur (parmi potentiellement des centaines) bloque le traitement de tous les items suivants. Le flow est en mode "tout ou rien".

**Correction :**
- Ajouter `continueOnFail: true` sur les nodes Google Sheets et Gmail
- Ajouter de la gestion d'erreur dans le node Code (try/catch avec valeurs par défaut)

---

#### ISSUE #3 : Le node "Date & Time" ne fait absolument rien

- **Node concerné** : `Date & Time`
- **Sévérité** : P0 — CRITIQUE (node fantôme = confusion + latence)

**Description :**
Configuration vide :
```json
{
  "includeTime": false,
  "options": {}
}
```
Aucune action, aucune conversion, aucune extraction. Il passe les données en transparence.

**Correction :**
Supprimer ce node. Connecter directement `Schedule Trigger` → `Get row(s) in sheet`.

---

### 2.2 P1 — HAUTS (performance, fiabilité, maintenabilité)

---

#### ISSUE #4 : Double écriture Google Sheets sur le même onglet

- **Nodes concernés** : `Append or update row in sheet` + `Append or update row in sheet1`
- **Sévérité** : P1 — HAUT

**Description :**
Le workflow écrit **deux fois** dans le même onglet `PROCESSED_B2C` du même document, pour chaque item :
1. **1ère écriture** : données brutes + `$today` dans "Date de transfer vers B2C"
2. **2ème écriture** : ré-écrit les mêmes données avec les dates formatées en JJ/MM/AAAA

La 1ère écriture est immédiatement écrasée par la 2ème.

**Impact :**
- Double la consommation du quota API Google Sheets (limité à 300 req/min par projet)
- Double la latence (~200-500ms par appel)
- La 1ère écriture est du **travail gaspillé** — elle est entièrement écrasée
- Risque de race condition si le batch est volumineux

**Correction :**
Fusionner en **une seule écriture** après avoir formaté toutes les données.

---

#### ISSUE #5 : Incohérence destinataire / corps du message Gmail

- **Node concerné** : `Send a message`
- **Sévérité** : P1 — HAUT

**Description :**
```
sendTo: "nelson.ngangonsoh@skillandyou.com"
message: "Bonjour Angélique, Anaïs, ..."
```
Le destinataire est Nelson mais le corps salue Angélique et Anaïs.

**Correction :**
Adapter le corps du message pour correspondre au destinataire réel (Nelson). Ex : "Bonjour Nelson,"

---

#### ISSUE #6 : Références fragiles vers des nodes distants

- **Node concerné** : `Append or update row in sheet` (1ère écriture)
- **Sévérité** : P1 — HAUT

**Description :**
Toutes les colonnes référencent un node situé 3 positions en amont :
```
={{ $('Get row(s) in sheet').item.json['E-mail'] }}
={{ $('Get row(s) in sheet').item.json.Nom }}
```
Au lieu de `$json['E-mail']` (données courantes).

**Impact :**
- Si le node `Get row(s) in sheet` est renommé → **tout casse**
- Les transformations de téléphone et civilité sont **ignorées** par cette écriture
- Pattern anti-N8N — les données devraient circuler naturellement d'un node au suivant

**Correction :**
Utiliser `$json` pour les données courantes. Ne référencer des nodes distants que si explicitement nécessaire.

---

#### ISSUE #7 : Noms de nodes par défaut — impossible à débugger

- **Nodes concernés** : `Code in JavaScript`, `Code in JavaScript1`, `Append or update row in sheet`, `Append or update row in sheet1`
- **Sévérité** : P1 — HAUT

**Correction :**

| Ancien nom | Nouveau nom proposé |
|------------|-------------------|
| `Code in JavaScript` | `Formater téléphone (+33)` |
| `Code in JavaScript1` | `Formater civilité (M./Mme)` |
| `Append or update row in sheet` | `Écriture Sheet — marquage transfert` |
| `Append or update row in sheet1` | `Écriture Sheet — données formatées` |

---

### 2.3 P2 — MOYENS (robustesse, qualité du code)

---

#### ISSUE #8 : Code civilité fragile — pas de normalisation

- **Node concerné** : `Code in JavaScript1`
- **Sévérité** : P2 — MOYEN

**Code actuel (fragile) :**
```javascript
if (civRaw === "monsieur" || civRaw === "Monsieur" || civRaw === "mr" || civRaw === "m") {
  civ = "M.";
}
```

**Cas non gérés :** `"MONSIEUR"`, `" Monsieur "`, `"Mr."`, `"M."`, `"Mlle"`, `null`, `undefined`

**Code corrigé :**
```javascript
const normalized = (civRaw || '').toString().toLowerCase().trim();
if (['monsieur', 'mr', 'mr.', 'm', 'm.'].includes(normalized)) {
  civ = 'M.';
} else {
  civ = 'Mme';
}
```

---

#### ISSUE #9 : Bug logique dans le formatage téléphone (code mort)

- **Node concerné** : `Code in JavaScript`
- **Sévérité** : P2 — MOYEN

**Code problématique :**
```javascript
// Cas 1 : commence par 33
if (digits.startsWith("33")) {
  digits = digits.substring(2);  // "33745..." → "745..."
}
// Cas 2 : commence par 330
if (digits.startsWith("330")) {  // ← JAMAIS VRAI après Cas 1 !
  digits = digits.substring(3);
}
```

**Problème :** Le Cas 2 est du **code mort**. Après le Cas 1, si le numéro commençait par `"330"`, le `"33"` a déjà été retiré.

**Correction :** Tester `"330"` AVANT `"33"` :
```javascript
if (digits.startsWith("330")) {
  digits = digits.substring(3);
} else if (digits.startsWith("33")) {
  digits = digits.substring(2);
}
```

---

#### ISSUE #10 : Espaces trailing dans les noms de colonnes

- **Colonnes** : `"ens "` et `"parurl "` (avec espace à la fin)
- **Sévérité** : P2 — MOYEN
- **Impact** : Un développeur qui écrit `$json['ens']` aura `undefined`. Bug invisible.
- **Correction** : Nettoyer les headers dans le Google Sheet source.

---

#### ISSUE #11 : Pas de gestion "0 résultats"

- **Localisation** : Après le node `If` (branche TRUE)
- **Sévérité** : P2 — MOYEN
- **Impact** : Si aucun lead à traiter → CSV vide envoyé par email = faux positif quotidien.
- **Correction** : Ajouter une vérification `$items().length > 0` avant Convert to File.

---

#### ISSUE #12 : Valeurs hardcodées dupliquées

- **Nodes** : `Append or update row in sheet1` + `Edit Fields`
- **Sévérité** : P2 — MOYEN
- **Valeurs dupliquées** :
  - `"ens " = "XX"` → dans 2 nodes
  - `"parurl " = "utm_source=hubspot&utm_medium=transfert"` → dans 2 nodes
- **Correction** : Définir une seule fois, dans le Code node consolidé.

---

#### ISSUE #13 : Nom de fichier CSV avec timestamp brut

- **Node** : `Convert to File`
- **Sévérité** : P2 — MOYEN
- **Actuel** : `leads_b2b_2025-12-04T16:20:00.000+00:00.csv` (illisible, `:` problématique sur Windows)
- **Correction** : `leads_b2b_{{ $now.format('dd-MM-yyyy') }}.csv` → `leads_b2b_04-12-2025.csv`

---

#### ISSUE #14 : executionOrder legacy "v1"

- **Localisation** : `settings.executionOrder: "v1"`
- **Sévérité** : P2 — MOYEN
- **Correction** : Vérifier la compatibilité avec le mode d'exécution actuel de N8N.

---

## 3. Tableau récapitulatif

| # | Sévérité | Problème | Effort | Impact |
|---|----------|----------|--------|--------|
| 1 | **P0** | Pas de validation E-mail → crash du flow | Moyen | Critique |
| 2 | **P0** | Zéro error handling | Faible | Critique |
| 3 | **P0** | Node Date & Time fantôme | Trivial | Moyen |
| 4 | **P1** | Double écriture Google Sheets | Moyen | Haut |
| 5 | **P1** | Incohérence destinataire/body Gmail | Trivial | Moyen |
| 6 | **P1** | Références fragiles vers nodes distants | Moyen | Haut |
| 7 | **P1** | Noms de nodes par défaut | Trivial | Moyen |
| 8 | **P2** | Code civilité fragile | Faible | Moyen |
| 9 | **P2** | Bug logique téléphone (code mort) | Faible | Moyen |
| 10 | **P2** | Espaces trailing colonnes | Faible | Faible |
| 11 | **P2** | Pas de gestion 0 résultats | Faible | Moyen |
| 12 | **P2** | Valeurs hardcodées dupliquées | Faible | Faible |
| 13 | **P2** | Nom CSV timestamp brut | Trivial | Faible |
| 14 | **P2** | executionOrder legacy | Trivial | Faible |

---

## 4. Architecture optimisée

### Avant (12 nodes, 0 protection)

```
Schedule Trigger → Date & Time (inutile) → Get rows → If (date vide?)
  → Code tél → Code civ → Sheet écriture 1 → Code dates → Sheet écriture 2
    → Edit Fields → Convert CSV → Gmail
```

### Après (11 nodes, résilience complète)

```
Schedule Trigger (16h20)
  → Lire leads (Google Sheet PROCESSED_B2C)
    → Filtrer non transférés (Date vide)
      → Valider E-mail (non vide + contient @)
        ├─ VALIDE :
        │   → Transformer données (1 seul Code : tél + civilité + dates)
        │     → Écriture unique Google Sheet
        │       → Préparer champs CSV
        │         → Générer CSV
        │           → Envoyer email Gmail
        └─ INVALIDE :
            → Logger dans onglet REJECTS
```

### Gains

| Métrique | Avant | Après |
|----------|-------|-------|
| Nombre de nodes | 12 | 11 |
| Appels API Google Sheets / item | 2 | **1 (−50%)** |
| Nodes Code JavaScript | 3 | **1** |
| Protection crash email vide | Non | **Oui** |
| Traçabilité leads rejetés | Non | **Oui (REJECTS)** |
| Error handling | 0 | **Tous les nodes critiques** |
| Envoi email si 0 leads | Oui (CSV vide) | **Non** |

---

## 5. Plan de test recommandé

| Scénario | Résultat attendu |
|----------|-----------------|
| Lead normal (tous champs remplis) | Traité, Sheet mis à jour, présent dans le CSV |
| Lead **SANS email** | Filtré vers REJECTS, ne bloque pas les autres |
| Lead sans téléphone | Traité avec téléphone vide, pas de crash |
| Lead civilité exotique ("MONSIEUR", "mr.", " madame ") | Formaté en M. ou Mme |
| Numéro "330612345678" | Formaté en +33612345678 |
| 0 leads à traiter | Pas d'email envoyé |
| Erreur API Google Sheets sur 1 item | Les autres items continuent |

---

## 6. Fichier optimisé

Le workflow optimisé est fourni dans :
**`Transfert automatique → B2C (Candidat formation initiale)_OPTIMIZED.json`**

Prêt à être importé dans N8N pour test en environnement staging avant mise en production.

---

*Rapport d'audit généré le 23/02/2026*
