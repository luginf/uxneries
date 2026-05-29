# Uxn — Morpion / Tic-tac-toe

Deux versions d'un morpion jouable sur la plateforme [Varvara/Uxn](https://wiki.xxiivv.com/site/uxntal.html) de Hundredrabbits.

| ROM | Grille | Victoire | Écran |
|---|---|---|---|
| `xo.rom` | 3×3 | 3 alignés | 107×107 |
| `xo5.rom` | 5×5 | **4 alignés** | 128×128 |

---

## Lancer

```sh
make run    # 3×3
make run5   # 5×5
```

Ou directement :

```sh
./uxn11 xo.rom
./uxn11 xo5.rom
```

---

## Compiler depuis les sources

```sh
make        # compile xo.rom et xo5.rom
make clean  # supprime les ROMs
```

Prérequis : `uxncli` et `drifloon.rom` dans le même répertoire.

---

## Jouer

**Clavier**

| Touche | Action |
|---|---|
| ←↑→↓ | Déplacer le curseur |
| `A` / `Espace` | Poser une pièce |

**Souris**

- Survoler une case : déplace le curseur
- Clic gauche sur une case libre : pose la pièce
- Clic sur le bouton **R** (bas droite) : redémarrer à tout moment

---

## Règles

- Le joueur joue **X**, l'IA joue **O**
- X commence toujours
- **3×3** : aligner 3 symboles identiques (ligne, colonne ou diagonale)
- **5×5** : aligner **4** symboles identiques
- Match nul si le plateau est rempli sans vainqueur (symbole triste affiché)

---

## IA

L'IA suit une heuristique par ordre de priorité :

1. **Gagner** — compléter un alignement gagnant
2. **Bloquer** — empêcher le joueur de gagner
3. **Centre** — prendre la case centrale
4. **Coin** — prendre un coin libre
5. **Case libre** — n'importe quelle case disponible

---

## Architecture technique

Écrit en [Uxntal](https://wiki.xxiivv.com/site/uxntal.html), assemblé avec [drifloon](https://git.sr.ht/~rabbits/uxn11).

- Deux couches d'affichage Varvara : BG (plateau + pièces) et FG (curseur souris)
- Détection d'entrée souris via front montant sur `Mouse/state`
- Le bouton R est actif en permanence, même en fin de partie

---

## Fichiers

```
xo.tal          source 3×3
xo5.tal         source 5×5
xo.rom          ROM compilée 3×3
xo5.rom         ROM compilée 5×5
Makefile
uxn11           émulateur graphique (X11)
uxncli          émulateur headless
drifloon.rom    assembleur (stdin→stdout)
drifblim.rom    assembleur (fichiers)
```
