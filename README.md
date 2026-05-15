# Librairies ICT — Pine Script v6

Ensemble de trois librairies Pine Script v6 implémentant les concepts **ICT (Inner Circle Trader)** pour TradingView : détection des Fair Value Gaps, détection des Order Blocks premium, et utilitaires visuels partagés.

> **Auteur :** jeremie_marvillier  
> **Version Pine Script :** 6  
> **Licence :** [Mozilla Public License 2.0](https://mozilla.org/MPL/2.0/)

---

## Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture](#architecture)
- [Ordre de publication](#ordre-de-publication)
- [ict\_utils](#ict_utils)
- [ict\_fvg](#ict_fvg)
- [ict\_ob](#ict_ob)
- [Exemple d'utilisation combinée](#exemple-dutilisation-combinée)
- [Concepts ICT implémentés](#concepts-ict-implémentés)
- [Choix de conception](#choix-de-conception)

---

## Vue d'ensemble

| Librairie | Rôle | Dépendances |
|---|---|---|
| `ict_utils` | Palette de couleurs, logique de mitigation, rendu des lignes communes | aucune |
| `ict_fvg` | Détection et suivi des Fair Value Gaps (FVG, IFVG, BISI, SIBI) | `ict_utils` |
| `ict_ob` | Détection et suivi des Order Blocks premium avec sweep de liquidité | `ict_utils` |

---

## Architecture

```
ict_utils          ← utilitaires partagés (palette, mitigation, lignes)
    ↑         ↑
ict_fvg      ict_ob
```

La logique de **mitigation** (déterminer si une zone est comblée et tracer le retest le plus profond) et le **rendu des lignes** médiane et de retest sont identiques dans les deux outils métier. Ces éléments sont centralisés dans `ict_utils` pour garantir la cohérence et faciliter la maintenance.

Chaque outil métier (`ict_fvg`, `ict_ob`) expose un **type exporté** (`FVG`, `OB`), quatre fonctions publiques (`effectiveBull`, `Get*`, `update`, `clear`, `process`, `draw`) et gère son propre cycle de vie via une machine à états interne.

---

## Ordre de publication

Sur TradingView, une librairie importée doit être publiée **avant** celle qui l'utilise.

```
1. Publier ict_utils   → note le numéro de version attribué (ex: 1)
2. Mettre à jour l'import dans ict_fvg et ict_ob si nécessaire :
       import jeremie_marvillier/ict_utils/<VERSION> as icu
3. Publier ict_fvg
4. Publier ict_ob
```

---

## ict_utils

**Fichier :** `ict_utils`  
**Import :** `import jeremie_marvillier/ict_utils/1 as icu`

Librairie de bas niveau. Ne contient aucun type exporté. Fournit la palette visuelle commune à tous les outils ICT, la logique de mitigation pure et deux helpers de rendu pour les lignes.

### Fonctions exportées

#### Palette de couleurs

Ces quatre fonctions renvoient les couleurs utilisées par l'ensemble des outils ICT pour les zones à l'état **fermé/mitigé**. Elles sont appelées dans les méthodes `draw()` de `ict_fvg` et `ict_ob`.

```pine
icu.midColor()       // → color  Ligne médiane (gris, transparence 30)
icu.retestColor()    // → color  Ligne de retest (orange, transparence 20)
icu.closedBgColor()  // → color  Fond d'une zone mitigée (gris, transparence 92)
icu.closedLineColor()// → color  Bordure d'une zone mitigée (gris, transparence 70)
```

#### Logique de mitigation

```pine
[isMitigated, newDeepestRetest] = icu.checkMitigation(
     effBull,        // bool  — direction effective de la zone (true = haussière)
     top,            // float — borne haute
     bottom,         // float — borne basse
     currentHigh,    // float — plus haut de la bougie courante
     currentLow,     // float — plus bas de la bougie courante
     deepestRetest)  // float — retest le plus profond connu (na si aucun)
```

Fonction **pure** (sans effet de bord). Détermine si une zone est entièrement mitigée sur la bougie courante et met à jour le niveau de retest le plus profond.

- La mitigation est évaluée sur la **mèche** (`currentLow` / `currentHigh`), conformément à la doctrine ICT.
- Si `isMitigated = true`, `newDeepestRetest` vaut la borne de clôture de la zone (bottom pour bull, top pour bear).
- Si la zone n'est pas encore mitigée mais que le prix y pénètre partiellement, `newDeepestRetest` est mis à jour si la pénétration est plus profonde que le retest précédent.

| Retour | Type | Description |
|---|---|---|
| `isMitigated` | `bool` | `true` si la zone est entièrement comblée sur cette bougie |
| `newDeepestRetest` | `float` | Retest le plus profond mis à jour (ou `deepestRetest` inchangé) |

#### Rendu — ligne médiane

```pine
ln := icu.updateMidLine(
     ln,           // line  — référence courante (na si pas encore créée)
     startTime,    // int   — timestamp de début de zone (xloc.bar_time)
     mid,          // float — niveau médian de la zone
     rightTime,    // int   — timestamp de fin (time ou closedTime)
     isClosed)     // bool  — true = passe la ligne en couleur "fermé"
// retourne : line — référence à réassigner côté appelant
```

Crée la ligne si elle n'existe pas encore (`na`), puis la met à jour à chaque bougie. **Doit être réassignée** : `this.middleLine := icu.updateMidLine(this.middleLine, ...)`.

#### Rendu — ligne de retest

```pine
ln := icu.updateRetestLine(
     ln,              // line  — référence courante (na si pas encore créée)
     startTime,       // int   — timestamp de début de zone (xloc.bar_time)
     deepestRetest,   // float — prix du retest le plus profond (na = pas encore retesté)
     rightTime,       // int   — timestamp de fin (time ou closedTime)
     isClosed)        // bool  — true = passe la ligne en couleur "fermé"
// retourne : line — référence à réassigner côté appelant
```

N'effectue aucune opération graphique si `deepestRetest` est `na`. **Doit être réassignée** : `this.retestLine := icu.updateRetestLine(this.retestLine, ...)`.

---

## ict_fvg

**Fichier :** `ict_fvg`  
**Import :** `import jeremie_marvillier/ict_fvg/1 as fvg`  
**Dépendance :** `ict_utils`

Détecte, qualifie et suit le cycle de vie des **Fair Value Gaps** (FVG) sur la timeframe courante.

### Concepts implémentés

| Concept | Description |
|---|---|
| FVG haussier / baissier | Motif à 3 bougies : gap entre le high de la bougie -2 et le low de la bougie 0 (ou inverse) |
| FVG vectoriel | La bougie de déplacement (bougie centrale du motif) a un volume > `volumeMult × moyenne` |
| BISI | *Buy-side Imbalance Sell-side Inefficiency* — FVG haussier dont les 3 bougies sont toutes haussières |
| SIBI | *Sell-side Imbalance Buy-side Inefficiency* — FVG baissier dont les 3 bougies sont toutes baissières |
| IFVG | *Inverted FVG* — FVG invalidé par une clôture au-delà de sa borne opposée ; change de polarité |
| Mitigation | La zone est comblée dès que la mèche d'une bougie traverse entièrement la borne opposée au sens effectif |

### Cycle de vie d'un FVG

```
ACTIF ──[clôture au-delà de la borne opposée]──► INVERSÉ (IFVG)
  │                                                    │
  └──[mèche traverse la borne]──► FERMÉ ◄─────────────┘
```

- Un IFVG **ne peut pas être ré-inversé** (simplification délibérée).
- Le champ `deepestRetest` est remis à `na` lors de l'inversion (le sens du retest change).

### Type exporté : `FVG`

```pine
export type FVG
    float top              // Borne haute du gap
    float bottom           // Borne basse du gap
    float middle           // Milieu du gap (equilibre)
    float amplitude        // Hauteur en points (top - bottom)
    float amplitudePct     // Hauteur en % du prix (référence : bottom)
    bool  isBull           // Nature D'ORIGINE (immuable) : true = haussier
    bool  isVector         // true si FVG vectoriel
    bool  isBisi           // true si BISI
    bool  isSibi           // true si SIBI
    bool  isInverted       // true si IFVG (polarité inversée)
    bool  isClosed         // true si entièrement mitigé
    float deepestRetest    // Retest le plus profond (na si aucun)
    int   startTime        // Timestamp de la bougie d'origine
    int   invertedTime     // Timestamp de l'inversion (na si non inversé)
    int   closedTime       // Timestamp de la mitigation (na si ouvert)
    int   vectorTime       // Timestamp de la bougie de déplacement
    box   visual           // Box graphique de la zone
    line  middleLine       // Ligne médiane
    line  retestLine       // Ligne du retest le plus profond
```

### Fonctions exportées

#### `GetFVG` — détection

```pine
FVG result = fvg.GetFVG(
     candlesOpens,    // series float
     candlesHighs,    // series float
     candlesLows,     // series float
     candlesCloses,   // series float
     candlesVolumes,  // series float
     volumeMult,      // simple float — défaut : 1.5
     volumeLen)       // simple int   — défaut : 20
// retourne : FVG ou na si aucun gap détecté sur cette bougie
```

Analyse les 3 dernières bougies. Utilise `ta.sma` en interne → **doit être appelée à chaque bougie**.

#### `effectiveBull` — direction effective

```pine
bool isCurrentlyBull = myFvg.effectiveBull()
```

Tient compte de l'inversion : un FVG baissier inversé retourne `true`.

#### `update` — mise à jour de l'état

```pine
myFvg.update(
     currentHigh,    // float
     currentLow,     // float
     currentClose)   // float — nécessaire pour détecter l'inversion
```

À appeler à chaque bougie pour chaque FVG actif. Met à jour `isInverted`, `isClosed`, `deepestRetest`.

#### `clear` — suppression graphique

```pine
myFvg.clear()
```

Supprime la box et les lignes associées au FVG. À appeler avant de retirer un FVG du tableau (éviction FIFO).

#### `process` — orchestration complète

```pine
FVG newFvg = fvg.process(
     fvgs,            // array<FVG> — déclaré avec var côté appelant
     candlesOpens,    // series float
     candlesHighs,    // series float
     candlesLows,     // series float
     candlesCloses,   // series float
     candlesVolumes,  // series float
     limit,           // simple int   — défaut : 20
     volumeMult,      // simple float — défaut : 1.5
     volumeLen)       // simple int   — défaut : 20
// retourne : le FVG nouvellement créé sur cette bougie, ou na
```

Regroupe détection + éviction FIFO + mise à jour de tous les FVG existants. C'est la fonction à utiliser dans la grande majorité des cas.

#### `draw` — rendu graphique

```pine
myFvg.draw()
```

Crée (si absent) ou met à jour la box, la ligne médiane et la ligne de retest. La couleur et le label suivent la **direction effective** :

| État | Couleur | Label |
|---|---|---|
| FVG haussier actif | Vert | `FVG+` |
| FVG baissier actif | Rouge | `FVG-` |
| BISI actif | Vert | `BISI` |
| SIBI actif | Rouge | `SIBI` |
| FVG vectoriel | Même couleur + `v` | `vFVG+` |
| Inversé (IFVG) | Couleur opposée + `I` | `IFVG-` |
| Mitigé | Gris pointillé | *(inchangé)* |

---

## ict_ob

**Fichier :** `ict_ob`  
**Import :** `import jeremie_marvillier/ict_ob/1 as ob`  
**Dépendance :** `ict_utils`

Détecte, qualifie et suit le cycle de vie des **Order Blocks premium** ICT (avec prérequis de prise de liquidité).

### Concepts implémentés

| Concept | Description |
|---|---|
| OB premium | Un Order Block n'est valide que s'il est précédé d'un **sweep de liquidité** (purge d'un swing récent) |
| Bullish OB | Dernière bougie **baissière** avant un sweep de plus bas récent (SSL purgé) |
| Bearish OB | Dernière bougie **haussière** avant un sweep de plus haut récent (BSL purgé) |
| Breaker Block | OB dont la borne opposée a été franchie par une clôture → change de polarité (support ↔ résistance) |
| Zone + corps | L'OB est visualisé en deux couches : zone complète (high–low) et corps de la bougie (open–close) |
| Mitigation | La zone est comblée dès que la mèche d'une bougie traverse entièrement la borne opposée au sens effectif |

### Cycle de vie d'un OB

```
ACTIF ──[clôture franchit la borne opposée]──► BREAKER BLOCK
  │                                                    │
  └──[mèche traverse la borne]──► FERMÉ ◄─────────────┘
```

- La mitigation est **bloquée sur la bougie d'inversion** (`justInverted`) pour éviter une fermeture instantanée : le prix vient de franchir la borne, il satisferait immédiatement la condition de mitigation dans le nouveau sens si elle était évaluée au même tick.
- Un Breaker Block **ne peut pas être ré-inversé**.
- Le champ `deepestRetest` est remis à `na` lors de l'inversion.

### Type exporté : `OB`

```pine
export type OB
    float top              // Borne haute de la zone (high de la bougie OB)
    float bottom           // Borne basse de la zone (low de la bougie OB)
    float bodyTop          // Haut du corps (max open/close)
    float bodyBottom       // Bas du corps (min open/close)
    float middle           // Milieu de la zone ((top + bottom) / 2)
    float amplitude        // Hauteur en points
    float amplitudePct     // Hauteur en % (référence : bottom)
    bool  isBull           // Nature D'ORIGINE (immuable) : true = haussier (support)
    bool  isInverted       // true si Breaker Block (polarité inversée)
    bool  isClosed         // true si entièrement mitigé
    float deepestRetest    // Retest le plus profond (na si aucun)
    int   startTime        // Timestamp de la bougie OB
    int   sweepTime        // Timestamp du sweep (prise de liquidité)
    int   invertedTime     // Timestamp du passage en Breaker (na si non inversé)
    int   closedTime       // Timestamp de la mitigation (na si ouvert)
    box   visual           // Box de la zone complète (high–low)
    box   bodyBox          // Box du corps (bodyTop–bodyBottom) avec label
    line  middleLine       // Ligne médiane
    line  retestLine       // Ligne du retest le plus profond
```

### Fonctions exportées

#### `GetPremiumOB` — détection

```pine
OB result = ob.GetPremiumOB(
     candlesOpens,   // series float
     candlesHighs,   // series float
     candlesLows,    // series float
     candlesCloses,  // series float
     swingLen)       // simple int — lookback du swing à purger, défaut : 10
// retourne : OB ou na si aucun OB premium détecté
```

Cherche la dernière bougie opposée dans les `swingLen` bougies précédant le sweep. Si le mouvement impulsif dépasse cette distance, augmenter `swingLen`.

#### `effectiveBull` — direction effective

```pine
bool isCurrentlyBull = myOb.effectiveBull()
```

Tient compte de l'inversion Breaker.

#### `update` — mise à jour de l'état

```pine
myOb.update(
     currentHigh,    // float
     currentLow,     // float
     currentClose)   // float — nécessaire pour l'inversion Breaker
```

À appeler à chaque bougie pour chaque OB actif. Met à jour `isInverted`, `isClosed`, `deepestRetest`. La mitigation est bloquée sur la bougie d'inversion.

#### `clear` — suppression graphique

```pine
myOb.clear()
```

Supprime la zone complète, la box du corps et les deux lignes.

#### `process` — orchestration complète

```pine
OB newOb = ob.process(
     obs,            // array<OB> — déclaré avec var côté appelant
     candlesOpens,   // series float
     candlesHighs,   // series float
     candlesLows,    // series float
     candlesCloses,  // series float
     limit,          // simple int — défaut : 20
     swingLen)       // simple int — défaut : 10
// retourne : le nouvel OB créé sur cette bougie, ou na
```

Contrairement à `ict_fvg`, la mise à jour des OB existants se fait **avant** la détection du nouvel OB, afin qu'un OB ne soit jamais évalué sur sa propre bougie de création. Déduplique par `startTime`.

#### `draw` — rendu graphique

```pine
myOb.draw()
```

Crée ou met à jour les deux boxes et les deux lignes. La couleur et le label suivent la direction effective :

| État | Couleur | Label corps |
|---|---|---|
| Bullish OB actif | Bleu | `OB+` |
| Bearish OB actif | Orange | `OB-` |
| Breaker haussier | Bleu | `BOB+` |
| Breaker baissier | Orange | `BOB-` |
| Mitigé | Gris pointillé | *(inchangé)* |

---

## Exemple d'utilisation combinée

Script indicateur affichant FVG et OB sur la timeframe courante :

```pine
//@version=6
indicator("ICT — FVG + OB", overlay = true, max_boxes_count = 500, max_lines_count = 500)

import jeremie_marvillier/ict_fvg/1   as fvg
import jeremie_marvillier/ict_ob/1    as ob

// ── Paramètres ────────────────────────────────────────────
int   fvgLimit     = input.int(15,  "Max FVG affichés",  minval = 1, maxval = 50)
float volumeMult   = input.float(1.5, "Multiplicateur volume (FVG vectoriel)", step = 0.1)
int   volumeLen    = input.int(20,  "Période moyenne volume")
int   obLimit      = input.int(10,  "Max OB affichés",   minval = 1, maxval = 50)
int   swingLen     = input.int(10,  "Lookback sweep OB", minval = 3, maxval = 50)

// ── État persistent ───────────────────────────────────────
var array<fvg.FVG> fvgs = array.new<fvg.FVG>()
var array<ob.OB>   obs  = array.new<ob.OB>()

// ── Orchestration ─────────────────────────────────────────
fvg.process(fvgs, open, high, low, close, volume, fvgLimit, volumeMult, volumeLen)
ob.process(obs,   open, high, low, close,          obLimit,  swingLen)

// ── Rendu ─────────────────────────────────────────────────
for f in fvgs
    f.draw()

for o in obs
    o.draw()
```

### Utilisation avancée : filtrage par type

```pine
// Récupérer le dernier FVG créé et filtrer
fvg.FVG lastFvg = fvg.process(fvgs, open, high, low, close, volume)

if not na(lastFvg)
    // Agir uniquement sur les BISI/SIBI vectoriels
    if (lastFvg.isBisi or lastFvg.isSibi) and lastFvg.isVector
        alert("FVG premium détecté à " + str.tostring(lastFvg.middle))

// Parcourir les OB actifs pour trouver des zones de confluence
for o in obs
    bool actif = not o.isClosed and not o.isInverted
    if actif and o.effectiveBull() and close > o.bottom and close < o.top
        // Le prix est dans un OB haussier actif
        label.new(bar_index, o.middle, "Dans OB+", style = label.style_label_down)
```

---

## Concepts ICT implémentés

### Choix conformes à la doctrine ICT

- **Motif FVG à 3 bougies** : gap entre `high[2]` et `low[0]` (bull) ou `low[2]` et `high[0]` (bear).
- **OB premium avec prérequis de sweep** : un OB sans prise de liquidité préalable n'est pas détecté.
- **Dernière bougie opposée** : l'OB est défini comme la *last opposing candle* avant le déplacement impulsif.
- **Anti-redétection du sweep** : un sweep n'est comptabilisé qu'au premier tick de transgression du niveau.

### Simplifications documentées

| Point | Choix retenu | Alternative stricte |
|---|---|---|
| Mitigation | Évaluée sur la **mèche** | Certains traders ICT utilisent la clôture |
| Inversion FVG | Bloquée après un premier IFVG (pas de ré-inversion) | Ré-inversion théoriquement possible |
| Breaker Block | Inversion déclenchée quand `close` franchit la borne de l'OB | Version stricte : franchissement du swing qui avait provoqué l'OB |
| Lookback OB | `swingLen` unique pour le sweep **et** la recherche de bougie opposée | Deux paramètres indépendants |

---

## Choix de conception

**Séparation état / rendu**  
La méthode `update()` ne touche pas aux objets graphiques ; `draw()` ne touche pas à l'état logique. Les deux peuvent être appelées indépendamment (ex : backtesting sans rendu).

**Logique de mitigation dans `ict_utils`**  
`checkMitigation()` est une fonction pure qui retourne `[isMitigated, newDeepest]`. Elle n'a aucun accès aux champs de l'objet appelant. L'appelant reste responsable de la mise à jour de `isClosed`, `closedTime` et `deepestRetest`. Cela permet de réutiliser la même logique dans tout nouvel outil ICT (Liquidity Levels, Order Blocks multi-TF, etc.).

**Pattern créer-ou-mettre-à-jour pour les lignes**  
`updateMidLine()` et `updateRetestLine()` retournent toujours la référence à la ligne. L'appelant doit la réassigner (`this.middleLine := icu.updateMidLine(...)`). Ce pattern évite la duplication de la logique `if na(ln) ... line.new() ... line.set_*()` dans chaque outil.

**`effectiveBull()` sur type concret**  
Pine Script v6 ne supporte pas les interfaces ni les types génériques. `effectiveBull()` est donc une méthode redéfinie sur chaque type (`FVG`, `OB`) mais avec une logique identique. La factoriser dans `ict_utils` nécessiterait de passer les champs individuellement, ce qui alourdirait les appels sans bénéfice réel.
