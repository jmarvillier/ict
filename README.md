# Librairies ICT — Pine Script v6

Ensemble de quatre librairies Pine Script v6 implémentant les concepts **ICT (Inner Circle Trader)** pour TradingView : détection des Fair Value Gaps, détection des Order Blocks premium, détection de la liquidité (BSL/SSL avec classification HRLR/LRLR) et du Dealing Range (Equilibrium / Premium / Discount), plus une librairie d'utilitaires visuels partagés.

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
- [ict\_liquidity](#ict_liquidity)
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
| `ict_liquidity` | Détection de la liquidité (BSL/SSL, HRLR/LRLR) et du Dealing Range (Equilibrium, Premium, Discount) | `ict_utils` |

---

## Architecture

```
ict_utils          ← utilitaires partagés (palette, mitigation, lignes)
    ↑         ↑          ↑
ict_fvg    ict_ob    ict_liquidity
```

La logique de **mitigation** (déterminer si une zone est comblée et tracer le retest le plus profond) et le **rendu des lignes** médiane et de retest sont identiques dans `ict_fvg` et `ict_ob`. Ces éléments sont centralisés dans `ict_utils` pour garantir la cohérence et faciliter la maintenance.

`ict_liquidity` réutilise la **palette d'état fermé** (`closedLineColor`) d'`ict_utils` pour les niveaux swept, mais n'a pas besoin de la logique de mitigation (un niveau de liquidité ne se "comble" pas — il est simplement pris ou non).

Chaque outil métier (`ict_fvg`, `ict_ob`, `ict_liquidity`) expose un ou plusieurs **types exportés** (`FVG`, `OB`, `Liquidity`, `DealingRange`), des fonctions publiques (`effectiveBull`, `Get*`, `update`, `clear`, `process`, `draw`) et gère son propre cycle de vie via une machine à états interne.

---

## Ordre de publication

Sur TradingView, une librairie importée doit être publiée **avant** celle qui l'utilise.

```
1. Publier ict_utils       → note le numéro de version attribué (ex: 1)
2. Mettre à jour l'import dans ict_fvg, ict_ob, ict_liquidity si nécessaire :
       import jeremie_marvillier/ict_utils/<VERSION> as icu
3. Publier ict_fvg
4. Publier ict_ob
5. Publier ict_liquidity
```

Les trois librairies métier (`ict_fvg`, `ict_ob`, `ict_liquidity`) sont indépendantes les unes des autres et peuvent être publiées dans n'importe quel ordre une fois `ict_utils` en place.

---

## ict_utils

**Fichier :** `ict_utils`  
**Import :** `import jeremie_marvillier/ict_utils/1 as icu`

Librairie de bas niveau. Ne contient aucun type exporté. Fournit la palette visuelle commune à tous les outils ICT, la logique de mitigation pure et deux helpers de rendu pour les lignes.

### Fonctions exportées

#### Palette de couleurs

Ces quatre fonctions renvoient les couleurs utilisées par l'ensemble des outils ICT pour les zones à l'état **fermé/mitigé** (ou **swept** pour la liquidité). Elles sont appelées dans les méthodes `draw()` de `ict_fvg`, `ict_ob` et `ict_liquidity`.

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

## ict_liquidity

**Fichier :** `ict_liquidity`  
**Import :** `import jeremie_marvillier/ict_liquidity/1 as liq`  
**Dépendance :** `ict_utils`

Détecte la **liquidité ICT** (BSL/SSL avec classification HRLR/LRLR) et le **Dealing Range** (Equilibrium 50% + zones Premium/Discount).

### Concepts implémentés

| Concept | Description |
|---|---|
| BSL (Buy-Side Liquidity) | Niveau de résistance au-dessus du prix où sont placés les stops des shorts — détecté via `ta.pivothigh` |
| SSL (Sell-Side Liquidity) | Niveau de support en-dessous du prix où sont placés les stops des longs — détecté via `ta.pivotlow` |
| LRLR (*Low Resistance Liquidity Run*) | Niveau **isolé** : aucun autre pivot du même type dans la tolérance — sweep facile, peu de stops empilés |
| HRLR (*High Resistance Liquidity Run*) | **Cluster** de 2+ pivots dans la tolérance (égaux highs ou lows) — sweep difficile, aimant à liquidité |
| Reclassification rétroactive | Lorsqu'un nouveau pivot tombe dans la tolérance d'un LRLR existant non-swept, tous deviennent simultanément HRLR |
| Sweep | Un niveau est marqué `isSwept` dès qu'une **mèche** le traverse (`high > BSL.price` ou `low < SSL.price`) |
| Dealing Range | Plus haut et plus bas sur un **lookback glissant** (`ta.highest` / `ta.lowest`) |
| Equilibrium | Niveau 50 % du Dealing Range — bascule entre Premium et Discount |
| Zone Premium | Moitié haute du DR (EQ → High) — le prix est "cher", zone où ICT cherche des **shorts** |
| Zone Discount | Moitié basse du DR (Low → EQ) — le prix est "pas cher", zone où ICT cherche des **longs** |

Deux pivots sont considérés "égaux" si `|p1 - p2| / p2 × 100 ≤ equalTolerancePct` (paramètre `equalTolerancePct`, défaut **0.1 %** du prix de référence).

### Cycle de vie d'un niveau de liquidité

```
DÉTECTION (LRLR par défaut)
        │
        ├──[ nouveau pivot ÉGAL dans la tolérance ]──► RECLASSIFICATION en HRLR
        │
        └──[ mèche traverse le niveau ]──► SWEPT (fin de vie)
```

- Un niveau **swept** ne peut pas redevenir actif (pas de "ré-activation").
- L'éviction FIFO du tableau finit par retirer les vieux niveaux swept de la mémoire et de l'affichage.
- Contrairement à FVG et OB, **pas d'inversion de polarité** : un BSL swept reste un BSL swept (la logique post-sweep relève de la stratégie, pas de la détection).

### Type exporté : `Liquidity`

```pine
export type Liquidity
    float price        // Niveau de la liquidité
    bool  isBSL        // true = BSL, false = SSL
    bool  isCluster    // true = HRLR (≥2 égaux), false = LRLR (isolé)
    int   touches      // Nombre de niveaux dans le cluster (1 si LRLR)
    bool  isSwept      // true si pris par une mèche
    int   startTime    // Timestamp du pivot d'origine
    int   sweptTime    // Timestamp du sweep (na si non swept)
    line  visual       // Référence à la ligne graphique
    label labelObj     // Référence au label
```

### Type exporté : `DealingRange`

```pine
export type DealingRange
    float high         // Plus haut du range
    float low          // Plus bas du range
    float equilibrium  // 50 % du range
    int   highTime     // Timestamp du plus haut
    int   lowTime      // Timestamp du plus bas
    box   premiumBox   // Box graphique zone Premium
    box   discountBox  // Box graphique zone Discount
    line  eqLine       // Ligne d'équilibre
```

### Fonctions exportées

#### `process` — orchestration liquidité

```pine
liq.Liquidity newLiq = liq.process(
     liqs,              // array<Liquidity> — déclaré avec var côté appelant
     candlesHighs,      // series float (généralement `high`)
     candlesLows,       // series float (généralement `low`)
     pivotLen,          // simple int   — défaut : 5
     limit,             // simple int   — défaut : 30
     equalTolerancePct) // simple float — défaut : 0.1
// retourne : le nouveau Liquidity créé sur cette bougie, ou na
```

Effectue dans l'ordre :

1. Mise à jour du statut `isSwept` de tous les niveaux existants.
2. Détection d'un éventuel nouveau pivot haut → BSL.
3. Détection d'un éventuel nouveau pivot bas → SSL.
4. Pour chaque nouveau pivot : reclassification rétroactive du cluster.
5. Éviction FIFO si `array.size(liqs) > limit`.

> ⚠️ Le pivot détecté à la bougie courante correspond au prix `pivotLen` bougies en arrière (délai de confirmation droite intrinsèque à `ta.pivothigh` / `ta.pivotlow`).

#### `update` — mise à jour du Dealing Range

```pine
dr.update(
     candlesHighs,   // series float
     candlesLows,    // series float
     lookback)       // simple int — défaut : 50
```

Recalcule `high`, `low`, `equilibrium`, `highTime`, `lowTime` à chaque appel sur un lookback glissant (`ta.highest` / `ta.lowest`). À appeler à chaque bougie.

#### `clear` — suppression graphique

```pine
myLiq.clear()   // supprime la ligne et le label d'un niveau de liquidité
dr.clear()      // supprime les deux boxes et la ligne EQ du Dealing Range
```

`process()` appelle automatiquement `clear()` sur les niveaux évincés en FIFO.

#### `draw` — rendu graphique

```pine
myLiq.draw()   // dessine ou met à jour un niveau de liquidité
dr.draw()      // dessine ou met à jour le Dealing Range
```

Visualisation de la liquidité :

| État | Style de ligne | Couleur | Label |
|---|---|---|---|
| LRLR BSL actif | Pointillé fin | Rouge transparent | `LRLR BSL` |
| HRLR BSL actif | Plein épais (width 2) | Rouge | `HRLR BSL ×N` |
| LRLR SSL actif | Pointillé fin | Vert transparent | `LRLR SSL` |
| HRLR SSL actif | Plein épais (width 2) | Vert | `HRLR SSL ×N` |
| Swept (BSL ou SSL) | Pointillé gris | Gris (via `icu.closedLineColor()`) | `… ✗` |

Où `N` (`touches`) = nombre de niveaux dans le cluster.

Visualisation du Dealing Range :

| Élément | Couleur | Position |
|---|---|---|
| Zone **Premium** | Rouge transparent (bgcolor 92) | EQ → High |
| Zone **Discount** | Vert transparent (bgcolor 92) | Low → EQ |
| Ligne **Equilibrium** | Gris pointillé | 50 % du range |

---

## Exemple d'utilisation combinée

Script indicateur affichant FVG, OB, BSL/SSL et Dealing Range sur la timeframe courante :

```pine
//@version=6
indicator("ICT — Dashboard complet", overlay = true,
     max_boxes_count  = 500,
     max_lines_count  = 500,
     max_labels_count = 500)

import jeremie_marvillier/ict_fvg/1       as fvg
import jeremie_marvillier/ict_ob/1        as ob
import jeremie_marvillier/ict_liquidity/1 as liq

// ── Activation des couches ────────────────────────────────
bool showFvg = input.bool(true, "Afficher les FVG",       group = "Activation")
bool showOb  = input.bool(true, "Afficher les OB",        group = "Activation")
bool showLiq = input.bool(true, "Afficher BSL/SSL",       group = "Activation")
bool showDr  = input.bool(true, "Afficher Dealing Range", group = "Activation")

// ── Paramètres FVG ────────────────────────────────────────
int   fvgLimit   = input.int(15,    "Max FVG affichés",   minval = 1, maxval = 50, group = "FVG")
float volumeMult = input.float(1.5, "Mult. volume (vFVG)", step  = 0.1,            group = "FVG")
int   volumeLen  = input.int(20,    "Période moy. volume",                          group = "FVG")

// ── Paramètres OB ─────────────────────────────────────────
int obLimit  = input.int(10, "Max OB affichés",   minval = 1, maxval = 50, group = "OB")
int swingLen = input.int(10, "Lookback sweep OB", minval = 3, maxval = 50, group = "OB")

// ── Paramètres Liquidité ──────────────────────────────────
int   pivotLen = input.int(5,     "Pivot length (BSL/SSL)",     minval = 2,  maxval = 20,  group = "Liquidité")
int   liqLimit = input.int(30,    "Max niveaux liquidité",      minval = 5,  maxval = 100, group = "Liquidité")
float eqTolPct = input.float(0.10,"Tolérance HRLR (% du prix)", step   = 0.05,             group = "Liquidité")

// ── Paramètres Dealing Range ──────────────────────────────
int drLookback = input.int(60, "Lookback Dealing Range", minval = 10, maxval = 500, group = "Dealing Range")

// ── État persistent ───────────────────────────────────────
var array<fvg.FVG>       fvgs = array.new<fvg.FVG>()
var array<ob.OB>         obs  = array.new<ob.OB>()
var array<liq.Liquidity> liqs = array.new<liq.Liquidity>()
var liq.DealingRange     dr   = liq.DealingRange.new()

// ── Orchestration (toujours appelée pour cohérence des séries ta.*) ──
fvg.process(fvgs, open, high, low, close, volume, fvgLimit, volumeMult, volumeLen)
ob.process(obs,  open, high, low, close,          obLimit,  swingLen)
liq.process(liqs,           high, low,            pivotLen, liqLimit, eqTolPct)
dr.update(high, low, drLookback)

// ── Rendu (conditionnel) ──────────────────────────────────
if showFvg
    for f in fvgs
        f.draw()
if showOb
    for o in obs
        o.draw()
if showLiq
    for l in liqs
        l.draw()
if showDr
    dr.draw()
```

### Utilisation avancée : confluence ICT mécanique

```pine
// Récupérer le nouveau pivot créé
liq.Liquidity newLiq = liq.process(liqs, high, low, pivotLen, liqLimit, eqTolPct)

// Alerte sur création d'un HRLR (aimant fort à liquidité)
if not na(newLiq) and newLiq.isCluster
    alert("Nouveau HRLR " + (newLiq.isBSL ? "BSL" : "SSL") +
          " à " + str.tostring(newLiq.price) +
          " (×" + str.tostring(newLiq.touches) + ")",
          alert.freq_once_per_bar)

// Setup short ICT : prix en Premium, HRLR BSL au-dessus, FVG vectoriel actif comme zone d'entrée
bool inPremium = dr.equilibrium > 0 and close > dr.equilibrium

bool hasBslTarget = false
for l in liqs
    if l.isBSL and l.isCluster and not l.isSwept and l.price > close
        hasBslTarget := true
        break

for f in fvgs
    bool actif = not f.isClosed and not f.isInverted
    if actif and not f.effectiveBull() and f.isVector and inPremium and hasBslTarget
        if close >= f.bottom and close <= f.top
            // Confluence : Premium + HRLR BSL cible + vFVG baissier traversé → setup short
            label.new(bar_index, high, "SHORT setup", style = label.style_label_down, color = color.red)
```

---

## Concepts ICT implémentés

### Choix conformes à la doctrine ICT

- **Motif FVG à 3 bougies** : gap entre `high[2]` et `low[0]` (bull) ou `low[2]` et `high[0]` (bear).
- **OB premium avec prérequis de sweep** : un OB sans prise de liquidité préalable n'est pas détecté.
- **Dernière bougie opposée** : l'OB est défini comme la *last opposing candle* avant le déplacement impulsif.
- **Anti-redétection du sweep** : un sweep n'est comptabilisé qu'au premier tick de transgression du niveau.
- **Liquidité par pivots structurels** : BSL/SSL ancrés sur `ta.pivothigh` / `ta.pivotlow` — niveaux ayant une histoire de réjection.
- **Classification HRLR/LRLR par clusters d'égaux** : conformément à la doctrine ICT, plusieurs pivots dans une tolérance prix forment un aimant à liquidité (cluster de stops empilés).
- **Sweep par mèche** (*"tag the level"*) : un niveau de liquidité est pris dès qu'une mèche le traverse, indépendamment de la clôture.
- **Premium / Discount / Equilibrium** : zone haute = vente, zone basse = achat, équilibre = niveau pivot — base de la logique "buy low, sell high" ICT.

### Simplifications documentées

| Point | Choix retenu | Alternative stricte |
|---|---|---|
| Mitigation FVG / OB | Évaluée sur la **mèche** | Certains traders ICT utilisent la clôture |
| Inversion FVG | Bloquée après un premier IFVG (pas de ré-inversion) | Ré-inversion théoriquement possible |
| Breaker Block | Inversion déclenchée quand `close` franchit la borne de l'OB | Version stricte : franchissement du swing qui avait provoqué l'OB |
| Lookback OB | `swingLen` unique pour le sweep **et** la recherche de bougie opposée | Deux paramètres indépendants |
| Dealing Range | Fenêtre glissante `ta.highest` / `ta.lowest` | Pivots structurels stricts (range non défini avant le premier pivot) |
| Polarité post-sweep | Pas d'inversion de polarité des niveaux de liquidité | Un BSL pris peut être traité comme support — à gérer côté stratégie via `isSwept` / `sweptTime` |
| Éviction Liquidity | FIFO simple (ancienneté pure) | Éviction préférentielle des swept |

---

## Choix de conception

**Séparation état / rendu**  
La méthode `update()` (et `process()` pour la liquidité) ne touche pas aux objets graphiques ; `draw()` ne touche pas à l'état logique. Les deux peuvent être appelées indépendamment (ex : backtesting sans rendu).

**Logique de mitigation dans `ict_utils`**  
`checkMitigation()` est une fonction pure qui retourne `[isMitigated, newDeepest]`. Elle n'a aucun accès aux champs de l'objet appelant. L'appelant reste responsable de la mise à jour de `isClosed`, `closedTime` et `deepestRetest`. Cela permet de réutiliser la même logique dans tout nouvel outil ICT.

**Pattern créer-ou-mettre-à-jour pour les lignes**  
`updateMidLine()` et `updateRetestLine()` retournent toujours la référence à la ligne. L'appelant doit la réassigner (`this.middleLine := icu.updateMidLine(...)`). Ce pattern évite la duplication de la logique `if na(ln) ... line.new() ... line.set_*()` dans chaque outil.

**`effectiveBull()` sur type concret**  
Pine Script v6 ne supporte pas les interfaces ni les types génériques. `effectiveBull()` est donc une méthode redéfinie sur chaque type (`FVG`, `OB`) mais avec une logique identique. La factoriser dans `ict_utils` nécessiterait de passer les champs individuellement, ce qui alourdirait les appels sans bénéfice réel. `Liquidity` n'a pas d'`effectiveBull()` car il n'y a pas d'inversion de polarité.

**Pivots stricts pour la liquidité, fenêtre glissante pour le Dealing Range**  
La liquidité est ancrée à des pivots structurels (`ta.pivothigh` / `ta.pivotlow`) pour identifier des niveaux ayant une histoire de réjection — c'est là que les ordres stops réels s'accumulent. Le Dealing Range utilise `ta.highest` / `ta.lowest` sur fenêtre glissante pour rester stable et toujours défini, évitant le scintillement qu'aurait une approche purement pivot.

**Reclassification rétroactive des clusters**  
Lorsqu'un nouveau pivot tombe dans la tolérance d'un ou plusieurs niveaux existants non-swept, tous sont reclassés en HRLR ensemble. Cela évite qu'un ancien LRLR reste isolé alors qu'un nouveau pivot vient s'y coller : il devient HRLR rétroactivement à partir du tick de détection. C'est la version la plus fidèle à la doctrine ICT (le cluster se forme par ajouts successifs).

**Pas d'inversion de polarité de la liquidité**  
Contrairement aux FVG (IFVG) et aux OB (Breaker Block), un niveau de liquidité ne change pas de polarité une fois swept. Il reste swept définitivement. La "polarité" post-sweep (un BSL swept pouvant servir de support) relève de la stratégie, pas de la détection — l'appelant peut utiliser les champs `isSwept` et `sweptTime` pour implémenter sa propre logique post-sweep.
