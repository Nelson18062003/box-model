# Audit Complet - Workflow N8N "Transfert automatique B2B → B2C (Candidat formation initiale)"

**Date d'audit** : 23/02/2026
**Workflow ID** : `oYp95mhKJjFJXXOx`
**Statut** : Actif en production
**Declencheur** : Schedule Trigger quotidien a 16h20

---

## 1. Cartographie du workflow actuel

### 1.1 Vue d'ensemble

Le workflow transfere automatiquement des leads B2B vers un processus B2C. Il lit un Google Sheet, filtre les leads non encore transferes, formate leurs donnees (telephone, civilite, dates), met a jour le sheet, genere un CSV et l'envoie par email.

### 1.2 Chaine de nodes (12 nodes)

```
Schedule Trigger (16h20)
  → Date & Time (VIDE - ne fait rien)
    → Get row(s) in sheet (lit TOUTES les lignes de PROCESSED_B2C)
      → If ("Date de transfer vers B2C" est vide ?)
        → TRUE :
          → Code in JavaScript (formate telephone → +33)
            → Code in JavaScript1 (formate civilite → M./Mme)
              → Append or update row in sheet (1ere ecriture Sheet, match sur E-mail)
                → fonction utilitaire pour formater une Date (formate dates)
                  → Append or update row in sheet1 (2eme ecriture Sheet, MEME onglet)
                    → Edit Fields (prepare champs CSV)
                      → Convert to File (CSV ; delimiter)
                        → Send a message (Gmail + piece jointe)
        → FALSE : rien (lead deja traite)
```

### 1.3 Source de donnees

- **Document** : "Mapping Formation CFA" (`115qZE2TksAKcgm__MpZJ1KR957NOcQxwEehDRB-s5m4`)
- **Onglet** : `PROCESSED_B2C` (gid `58518804`)
- **Credentials** : Google Sheets OAuth2 (`QadzahQ1h7wOb4CD`)

---

## 2. Problemes identifies

### Synthese

| Severite | Nombre | Description |
|----------|--------|-------------|
| **P0 - Critique** | 3 | Bugs bloquants en production |
| **P1 - Haut** | 4 | Performance, fiabilite, maintenabilite |
| **P2 - Moyen** | 7 | Robustesse, qualite du code |
| **Total** | **14** | |

---

### 2.1 P0 - CRITIQUES (bloquent le flow en production)

#### P0-1 : Aucune validation E-mail avant les nodes Google Sheets

- **Nodes concernes** : `Append or update row in sheet`, `Append or update row in sheet1`
- **Description** : Les 2 nodes Google Sheets utilisent `E-mail` comme `matchingColumns` (cle de correspondance pour l'upsert). Si un lead arrive sans email (champ vide ou absent), le node Google Sheets echoue car il ne peut pas matcher sur une valeur vide.
- **Impact** : **C'est exactement le bug qui a ete rencontre recemment.** Un seul lead sans email provoque l'arret complet du workflow. Tous les leads suivants dans le batch ne sont pas traites.
- **Correction** : Ajouter un node `IF` apres le filtre actuel pour verifier que `E-mail` n'est pas vide ET contient un `@`. Router les leads invalides vers un onglet `REJECTS`.

#### P0-2 : Zero error handling sur tout le workflow

- **Nodes concernes** : Tous (12 nodes)
- **Description** : Aucun mecanisme de gestion d'erreur n'est configure :
  - Pas de `continueOnFail` sur les nodes critiques (Google Sheets, Gmail)
  - Pas de node `Error Trigger` pour alerter en cas d'echec
  - Pas de `try/catch` dans les nodes Code
- **Impact** : Une erreur sur un seul item (ex: format de donnees inattendu, timeout API Google) bloque l'ensemble du batch. Il n'y a aucune visibilite sur les echecs sans aller manuellement dans l'historique d'execution N8N.
- **Correction** : Ajouter `continueOnFail: true` sur les nodes Google Sheets et Gmail. Ajouter une gestion `try/catch` dans le node Code consolide.

#### P0-3 : Le node "Date & Time" ne fait absolument rien

- **Node concerne** : `Date & Time`
- **Description** : Ce node est configure avec `includeTime: false` et `options: {}`. Il ne realise aucune transformation. C'est un node fantome.
- **Impact** : Ajoute de la latence inutile a chaque execution (appel API interne N8N). Source de confusion pour quiconque maintient le workflow ("Pourquoi ce node est-il la ? Que fait-il ?").
- **Correction** : Supprimer ce node. Connecter directement `Schedule Trigger` a `Get row(s) in sheet`.

---

### 2.2 P1 - HAUTS (performance, fiabilite, maintenabilite)

#### P1-1 : Double ecriture Google Sheets sur le meme onglet

- **Nodes concernes** : `Append or update row in sheet` + `Append or update row in sheet1`
- **Description** : Le workflow ecrit DEUX FOIS dans le meme onglet `PROCESSED_B2C` du meme document, avec le meme `matchingColumns: ["E-mail"]` :
  1. **1ere ecriture** : ecrit `$today` dans "Date de transfer vers B2C" + copie toutes les autres colonnes depuis `Get row(s) in sheet` (reference distante)
  2. **2eme ecriture** : re-ecrit les memes donnees mais avec les dates formatees en `JJ/MM/AAAA` + civilite formatee
- **Impact** :
  - Consomme 2x le quota API Google Sheets (inutilement)
  - Double la latence d'ecriture
  - La 1ere ecriture est completement ecrasee par la 2eme → travail gaspille
  - Risque de condition de course si le batch est gros
- **Correction** : Fusionner en 1 seule ecriture apres que toutes les transformations soient faites.

#### P1-2 : Incoherence destinataire / contenu email

- **Node concerne** : `Send a message`
- **Description** : Le `sendTo` est `nelson.ngangonsoh@skillandyou.com` mais le corps du message dit "Bonjour Angelique, Anais,". Le contenu ne correspond pas au destinataire.
- **Impact** : Confusion pour le destinataire. Manque de professionnalisme.
- **Correction** : Adapter le corps du message pour qu'il corresponde au destinataire reel (Nelson), ou utiliser un message generique.

#### P1-3 : References fragiles vers des nodes distants

- **Node concerne** : `Append or update row in sheet` (1ere ecriture)
- **Description** : Toutes les colonnes referencent `$('Get row(s) in sheet').item.json[...]` — remontant 3 nodes en arriere dans la chaine. Cela court-circuite les transformations faites par les nodes intermediaires (formatage telephone, civilite).
- **Impact** :
  - Si un node intermediaire est renomme, deplace ou supprime → toutes ces references cassent
  - Les transformations de telephone et civilite sont **ignorees** par la 1ere ecriture (elle lit les donnees brutes d'origine)
  - Ce pattern rend le debugging extremement difficile
- **Correction** : Utiliser `$json` pour lire les donnees du node precedent dans la chaine, ou consolider en une seule ecriture apres toutes les transformations.

#### P1-4 : Noms de nodes par defaut - impossible a debugger

- **Nodes concernes** : `Code in JavaScript`, `Code in JavaScript1`, `Append or update row in sheet`, `Append or update row in sheet1`
- **Description** : Les noms sont les noms par defaut generes par N8N. Quand une erreur survient sur "Code in JavaScript1", il est impossible de savoir ce que ce node fait sans l'ouvrir.
- **Impact** : Temps de debug multiplie. Difficulte pour toute personne autre que l'auteur de comprendre le flow.
- **Correction** : Renommer tous les nodes avec des noms descriptifs (ex: "Formater telephone +33", "Formater civilite M./Mme", "Ecrire PROCESSED_B2C", etc.)

---

### 2.3 P2 - MOYENS (robustesse, qualite du code)

#### P2-1 : Code civilite fragile - pas de normalisation

- **Node concerne** : `Code in JavaScript1`
- **Code actuel** :
```javascript
if (civRaw === "monsieur" || civRaw === "Monsieur" || civRaw === "mr" || civRaw === "m") {
  civ = "M.";
} else if (civRaw === "madame" || civRaw === "Madame" || civRaw === "mme" || civRaw === "m me") {
  civ = "Mme";
} else {
  civ = "Mme"; // defaut
}
```
- **Probleme** : Ne gere pas `"MONSIEUR"`, `"Mr."`, `"  monsieur  "` (espaces), `"mlle"`, `"Mademoiselle"`, etc.
- **Correction** : Utiliser `civRaw.toLowerCase().trim()` puis matcher contre des listes normalisees.

#### P2-2 : Bug logique dans le formatage telephone (code mort)

- **Node concerne** : `Code in JavaScript`
- **Code problematique** :
```javascript
// Cas 1 : commence par 33
if (digits.startsWith("33")) {
  digits = digits.substring(2);  // "33745..." → "745..."
}
// Cas 2 : commence par 330
if (digits.startsWith("330")) {  // ← JAMAIS VRAI apres Cas 1
  digits = digits.substring(3);
}
```
- **Probleme** : Le Cas 2 est du **code mort**. Apres le Cas 1, si le numero commencait par `"330..."`, le `"33"` a deja ete retire, donc il ne commence plus par `"330"`. Un numero comme `330612345678` sera transforme en `+330612345678` au lieu de `+33612345678`.
- **Correction** : Tester `"330"` AVANT `"33"`, ou utiliser un seul regex pour gerer tous les cas.

#### P2-3 : Espaces trailing dans les noms de colonnes

- **Colonnes concernees** : `"ens "` et `"parurl "` (avec espace a la fin)
- **Probleme** : Ces espaces sont invisibles dans l'interface N8N mais peuvent causer des bugs intermittents (comparaisons qui echouent, colonnes non trouvees). Toute personne qui tenterait de referencer ces colonnes par leur nom "logique" (`ens`, `parurl`) echouerait.
- **Correction** : Si le Google Sheet source a ces espaces, il faut les corriger dans le Sheet ET dans le workflow. Sinon, documenter clairement cette particularite.

#### P2-4 : Pas de gestion "0 resultats"

- **Localisation** : Entre le node `If` et le reste du flow
- **Probleme** : Si aucun lead n'a une "Date de transfer vers B2C" vide (tous deja traites), le flux TRUE du `If` ne recoit aucun item. Selon la configuration N8N, cela peut :
  - Generer un CSV vide et envoyer un email avec une piece jointe vide
  - Ou ne rien faire silencieusement (pas d'alerte)
- **Correction** : Ajouter un node IF qui verifie `$items().length > 0` avant la generation CSV + email.

#### P2-5 : Valeurs hardcodees dupliquees

- **Nodes concernes** : `Append or update row in sheet1` + `Edit Fields`
- **Valeurs** :
  - `"ens " = "XX"` → present dans les 2 nodes
  - `"parurl " = "utm_source=hubspot&utm_medium=transfert"` → present dans les 2 nodes
- **Probleme** : Si on modifie la valeur dans un node mais oublie l'autre → incoherence entre le Google Sheet et le CSV. Violation du principe DRY.
- **Correction** : Definir ces valeurs une seule fois (dans le Code node consolide) et les propager.

#### P2-6 : Nom de fichier CSV avec timestamp brut

- **Node concerne** : `Convert to File`
- **Valeur actuelle** : `fileName: "leads_b2b_{{$now}}.csv"`
- **Resultat** : `leads_b2b_2025-12-04T16:20:00.000+00:00.csv` — illisible, contient des `:` qui posent probleme sur certains OS.
- **Correction** : Utiliser un format propre : `leads_b2b_{{ $now.format('dd-MM-yyyy') }}.csv` → `leads_b2b_04-12-2025.csv`

#### P2-7 : executionOrder legacy "v1"

- **Localisation** : `settings.executionOrder: "v1"`
- **Probleme** : Le workflow utilise l'ancien mode d'execution de N8N. Le mode actuel offre un meilleur comportement pour les workflows avec des branches multiples.
- **Correction** : Passer en mode d'execution par defaut (supprimer le parametre ou le mettre a jour).

---

## 3. Recommandations d'optimisation

### 3.1 Architecture cible

```
Schedule Trigger (16h20)
  → Lire leads (Google Sheet PROCESSED_B2C)
    → Filtrer non transferes (Date vide)
      → Valider E-mail (non vide + contient @)
        → VALIDE :
          → Transformer donnees (1 Code node : tel + civilite + dates)
            → Ecriture unique Google Sheet PROCESSED_B2C
              → Preparer champs CSV
                → Generer CSV
                  → Envoyer email Gmail
        → INVALIDE :
          → Logger dans onglet REJECTS
```

### 3.2 Gains attendus

| Metrique | Avant | Apres |
|----------|-------|-------|
| Nombre de nodes | 12 | 10 + 1 branche rejet |
| Appels API Google Sheets (ecriture) | 2 par lead | 1 par lead |
| Nodes Code JavaScript | 3 separes | 1 consolide |
| Gestion erreur | Aucune | `continueOnFail` + logging REJECTS |
| Robustesse email vide | Crash total | Filtre + log |
| Nommage nodes | Noms par defaut | Noms descriptifs |
| Debug time (estime) | Long (noms cryptiques) | Rapide (noms clairs) |

### 3.3 Matrice de priorite d'implementation

| Priorite | Action | Effort | Impact |
|----------|--------|--------|--------|
| 1 | Ajouter validation E-mail + branche REJECTS | Moyen | Critique |
| 2 | Ajouter `continueOnFail` | Faible | Critique |
| 3 | Supprimer node Date & Time inutile | Faible | Faible |
| 4 | Fusionner 3 nodes Code en 1 | Moyen | Haut |
| 5 | Supprimer la double ecriture Sheet | Moyen | Haut |
| 6 | Corriger bug telephone (code mort) | Faible | Moyen |
| 7 | Robustifier code civilite | Faible | Moyen |
| 8 | Renommer tous les nodes | Faible | Moyen |
| 9 | Ajouter check "0 resultats" | Faible | Moyen |
| 10 | Adapter body email | Faible | Faible |
| 11 | Corriger nom fichier CSV | Faible | Faible |
| 12 | Corriger espaces trailing colonnes | Faible | Faible |
| 13 | Mettre a jour executionOrder | Faible | Faible |

---

## 4. Checklist de validation post-implementation

- [ ] Lead normal (tous champs remplis) → traite correctement, sheet mis a jour, CSV genere, email envoye
- [ ] Lead SANS email → filtre vers REJECTS, ne crashe PAS le flow, les autres leads passent
- [ ] Lead sans telephone → passe avec telephone vide, pas de crash
- [ ] Lead avec civilite exotique ("MONSIEUR", "mr.", " madame ") → correctement normalise
- [ ] Numero commencant par "330..." → correctement formate en +33...
- [ ] 0 leads a traiter → pas d'email envoye, pas de CSV vide
- [ ] Verifier 1 seul appel API Google Sheets par lead (pas 2)
- [ ] Verifier nom CSV lisible (format `leads_b2b_JJ-MM-AAAA.csv`)
- [ ] Verifier que le body email correspond au destinataire (Nelson)

---

## 5. Fichier optimise

Le workflow optimise est fourni dans le fichier :
`Transfert automatique → B2C (Candidat formation initiale)_OPTIMIZED.json`

Ce fichier est pret a etre importe dans N8N pour test en environnement de staging avant mise en production.
