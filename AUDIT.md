# PallyPowerTW — Audit complet & plan d'amélioration

> Document de travail (le « prompt ») pour guider les corrections et améliorations de l'addon.
> Cible : WoW 1.12 / Turtle WoW. Fichier principal : `PallyPower.lua` (~3810 lignes).
> Version TOC : 1.40 — Auteur d'origine : ivanovlk.

---

## 1. Vue d'ensemble

PallyPowerTW coordonne les **bénédictions de paladin** (et auras/sceaux) dans un groupe/raid :
assignations par classe et individuelles, synchronisation entre paladins via messages addon,
scan des buffs présents, et cast en un clic (Grande Bénédiction 30 min = clic gauche,
bénédiction 10 min = clic droit, avec override individuel).

### Systèmes principaux
| Système | Fonctions clés | Rôle |
|---------|----------------|------|
| Assignations | `*_PerformCycle*`, `GetNormalBlessings`, `SetNormalBlessings` | Choix des bénés par classe / par joueur |
| Sync réseau | `PallyPower_SendSelf`, `PallyPower_ParseMessage`, `*_TacticaParseMessage` | Partage état entre paladins |
| Scan buffs | `PallyPower_ScanRaid`, `ScanOneUnit`, `RebuildRoster`, `PruneCurrentBuffs` | État des buffs présents |
| Scan sorts | `PallyPower_ScanSpells`, `PallyPowerManaCost.lua` | Rangs/talents dispo du joueur |
| Cast | `PallyPowerBuffButton_OnClick`, `PallyPower_AutoBless`, `PallyPower_CastSeal` | Lancement des sorts |
| UI | `PallyPower_UpdateUI`, `PallyPowerGrid_Update`, `PallyPower_UpdateLayout`, `PallyPower.xml` | Affichage |
| Boucle | `PallyPower_OnUpdate`, `PallyPower_OnEvent` | Timers, debounce UI, événements |

### Protocole réseau (préfixe `PP_PREFIX`, canal PARTY/RAID)
- `SELF`/`ASELF`/`SSELF` : rangs+talents + assignations du paladin émetteur (bénés/auras/sceaux).
- `ASSIGN`/`AASSIGN`/`SASSIGN`/`MASSIGN`/`NASSIGN` : assignation (classe / aura / sceau / toutes / individuelle).
- `SYMCOUNT`, `COOLDOWNS`, `FREEASSIGN`, `TANK`/`CLTNK`, `HEALER`/`CLHLR`, `CLEAR`, `VERSION`, `REQ`.

---

## 2. Bugs & risques de crash (par sévérité)

### 🔴 Critiques (crash confirmé ou très probable, déclenché par des messages d'autres versions)

1. **`COOLDOWNS` indexe `AllPallys[sender]` sans garde** — `PallyPower.lua:2223-2224`.
   `AllPallys[sender]["DivineIntervention"] = ...` plante si aucun `SELF` n'a encore été reçu de
   `sender` (table nil). `SYMCOUNT` juste au-dessus se protège (`if AllPallys[sender]`), pas COOLDOWNS.
   → **Fix : garder, sinon `REQ`.**

2. **`VERSION` : comparaison sur valeur potentiellement nil** — `PallyPower.lua:2295`.
   `if msgVer > PallyPower_Version` plante si `msgVer` est nil (message `VERSION` malformé).
   De plus la comparaison est lexicographique (`"1.9" > "1.10"` = vrai à tort).
   → **Fix : garde nil + comparaison numérique.**

3. **`SELF`/`ASELF`/`SSELF` : `tmp + 0` sur caractère non validé** — `PallyPower.lua:2063, 2089, 2114`.
   `tmp` = un caractère extrait du message ; s'il n'est ni un chiffre ni `n`, `tmp + 0` plante
   (même classe de bug que `ASSIGN` déjà corrigé en `ddd46f0`).
   → **Fix : `tonumber()` + fallback -1.**

4. **`talent + 0` dans l'UI** — `PallyPower.lua:832, 846`.
   `skills[id]["talent"]` / `AllPallysAuras[name][id-6].talent` viennent des messages `SELF`/`ASELF`.
   Si un talent stocké n'est pas numérique, le crash se produit **à chaque frame** pendant le
   rendu de la grille. → **Fix racine : valider rank/talent au parse (les stocker en nombres).**

### 🟠 Moyens (robustesse / correction)

5. **Variables globales involontaires dans le parser** — `rank`, `talent`, `tmp`, `numbers`
   (`PallyPower.lua:2049-2114`) ne sont pas `local` → pollution du namespace global, risque de
   collision. → Les passer en `local`.

6. **Regex gloutonnes pour les noms** — `ASSIGN`/`AASSIGN`/… utilisent `(.*) (.*)`
   (`PallyPower.lua:2119, 2142, 2155, 2168`). `NASSIGN` (2193) utilise `(%S+)` (mieux).
   Risque de mauvais découpage. → Standardiser sur `(%S+)`.

7. **`pnum/class + 0` sans garde si regex échoue** — `PallyPower.lua:2417-2418, 3716-3717`.
   Entrée contrôlée (noms de frames) donc faible risque, mais `nil + 0` planterait.
   → Garde `tonumber` défensive.

8. **`SYMCOUNT` stocke une chaîne** — `PallyPower.lua:2211-2213` : `["symbols"]` = string.
   Incohérent si comparé numériquement ailleurs. → `tonumber`.

### 🟡 Mineurs / qualité

9. `local type, id` inutilisés dans `OnEvent` (`PallyPower.lua:463`) — `type` masque la fonction
   native `type()`. → Retirer.
10. `print` global redéfini (`PallyPower.lua:108`) — peut surprendre d'autres addons. → Préfixer/retirer.
11. Itérations `for k in t do` (style Lua 4) mélangées avec `pairs()` — cohérence stylistique.

---

## 3. Robustesse réseau

L'addon reçoit des messages d'**autres versions de PallyPower** (joueurs avec des forks différents).
Tout handler doit traiter chaque champ comme **non fiable** :
- Toujours `tonumber()` + garde avant arithmétique ou indexation numérique.
- Toujours vérifier l'existence de `AllPallys[sender]` avant d'y écrire des sous-clés.
- Ne jamais supposer qu'un `string.find` a réussi (les captures peuvent être nil).

**Principe :** un message malformé doit être **ignoré silencieusement** (`return false`), jamais planter.

---

## 4. Performance

- `OnUpdate` met `uiDirty=true` tant qu'un timer existe (`PallyPower.lua:355`) → reconstruction UI
  toutes les `scanfreq` (1 s) en continu pendant un raid (timers GB de 30 min). Acceptable mais
  optimisable (ne rafraîchir que les libellés de temps, pas tout le layout).
- `UNIT_AURA` → `ScanOneUnit` ciblé (bon).
- `RRAID_ROSTER_UPDATE` reconstruit tout le roster + re-scan (correct, peu fréquent).

---

## 5. Plan d'action priorisé

| # | Action | Risque | Test en jeu requis |
|---|--------|--------|--------------------|
| 1 | Durcir parser (COOLDOWNS, VERSION, SELF/ASELF/SSELF, globals→local) | Très faible (gardes pures) | Non |
| 2 | Durcir `talent + 0` UI + valider au parse | Très faible | Non |
| 3 | Garde `tonumber` sur `pnum/class + 0` | Très faible | Léger |
| 4 | Standardiser regex noms `(%S+)` | Faible | Léger |
| 5 | Nettoyage (dead locals, `print`) | Très faible | Non |
| 6 | Optimisation refresh UI timers | Moyen | Oui |

---

## 6. Corrigé dans cette passe

Toutes ces corrections sont du **durcissement pur** (ajout de gardes / validation) : elles ne
changent pas le comportement nominal, donc faible risque de régression.

- ✅ **COOLDOWNS** (`~2223`) : garde `if AllPallys[sender] and cooldowns` avant indexation, sinon `REQ`.
- ✅ **VERSION** (`~2295`) : `tonumber()` + garde nil sur `msgVer` et `PallyPower_Version`
  (plus de crash sur message malformé, comparaison numérique correcte).
- ✅ **SELF / ASELF / SSELF** (`~2045-2103`) :
  - `rank`, `talent`, `tmp`, `numbers` passés en **`local`** (fin de la pollution globale) ;
  - `talent` stocké en **nombre** (`tonumber(talent) or 0`) → corrige le crash racine de `talent + 0` ;
  - garde `rank ~= ""` (n'enregistre plus d'entrées vides) ;
  - ASELF/SSELF : l'entrée n'est créée **que si** `PallyPower_AuraID/SealID[id]` existe
    (corrige un cas où `.talent` restait nil → crash à l'affichage).
- ✅ **Affichage talents** (`832`, `846`) : garde défensive `tonumber(...) or 0` (l'UI ne plante
  plus jamais sur un talent non numérique).
- ✅ **Boutons grille** (`~2417`, `~3716`) : `tonumber(pnum/class)` + early-return si la regex échoue
  (plus de `nil + 0`).
- ✅ **SYMCOUNT** (`~2210`) : stocké en nombre (`tonumber(count) or 0`).
- ✅ **Nettoyage** : `local type, id` mort retiré de `OnEvent` (ne masque plus la fonction `type()`).

### Recommandé mais NON fait (risque/à tester en jeu)
- Standardiser les regex de noms `(.*)` → `(%S+)` sur ASSIGN/AASSIGN/SASSIGN/MASSIGN (déjà
  crash-safe grâce aux `tonumber` de `ddd46f0`, donc gain faible — laissé pour ne pas toucher au réseau).
- Optimiser le refresh UI des timers (ne rafraîchir que les libellés de temps, pas tout le layout).
- Revoir l'override global de `print` (`108`).

