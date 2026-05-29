# Uxn — Projet local

Environnement de développement pour la plateforme [Varvara/Uxn](https://wiki.xxiivv.com/site/uxntal.html) de Hundredrabbits.

## Fichiers du projet

| Fichier | Description |
|---|---|
| `uxn11` | VM graphique (X11), compilée depuis [`~rabbits/uxn11`](https://git.sr.ht/~rabbits/uxn11) |
| `uxn2` | VM graphique (SDL2), source locale dans `src/uxn2.c` (fork de [`~rabbits/uxn2`](https://git.sr.ht/~rabbits/uxn2)) |
| `src/uxn2.c` | Source modifiée de uxn2 : options `-2x` / `-3x` pour zoom au démarrage |
| `uxncli` | VM headless (terminal), compilée depuis [`~rabbits/uxncli`](https://git.sr.ht/~rabbits/uxncli) |
| `drifblim.rom` | Assembleur pour uxn11/uxn2 (fichiers en entrée/sortie) |
| `drifloon.rom` | Assembleur pour uxncli (stdin/stdout) |
| `xo.tal` | Morpion 3×3 avec IA, souris et bouton restart (source Uxntal) |
| `xo.rom` | ROM compilée du jeu 3×3 |
| `xo5.tal` | Morpion 5×5 — victoire en 4 alignés, IA étendue (source Uxntal) |
| `xo5.rom` | ROM compilée du jeu 5×5 |
| `Makefile` | `make` compile les deux ROMs, `make run` / `make run5` les lance |

## Compiler et lancer

```sh
make           # compile xo.rom et xo5.rom
make run       # compile + lance xo.rom (3×3)
make run5      # compile + lance xo5.rom (5×5)
make clean     # supprime les ROMs
```

## Assembleur

Deux workflows selon l'environnement :

```sh
# Avec affichage X11
./uxn11 drifblim.rom source.tal output.rom

# Sans affichage (headless) — IMPORTANT : ne pas utiliser 2>&1, ça corrompt la ROM
cat source.tal | ./uxncli drifloon.rom > output.rom
```

## Lancer un ROM

```sh
./uxn11 output.rom          # zoom ×1 (défaut)
./uxn2 output.rom           # zoom ×1 (défaut)
./uxn2 -2x output.rom       # zoom ×2
./uxn2 -3x output.rom       # zoom ×3
```

F1 pendant l'exécution cycle entre ×1 / ×2 / ×3.

**Mécanisme zoom uxn2** : les options `-2x`/`-3x` simulent 1 ou 2 appuis F1 juste après `emu_init()`. SDL2 ne corrige pas automatiquement les coordonnées souris avec `SDL_RenderSetLogicalSize` — ce mécanisme contourne le problème.

## Recompiler les VM

```sh
# uxn11 (nécessite libX11-dev)
cc -DNDEBUG -O2 -g0 -s src/uxn11.c -lX11 -lutil -o uxn11

# uxn2 — source locale modifiée (nécessite libsdl2-dev)
cc $(sdl2-config --cflags) -DNDEBUG -O2 -g0 -s src/uxn2.c -o uxn2 $(sdl2-config --libs)

# uxncli
cc -DNDEBUG -O3 -g0 -s src/uxncli.c -o uxncli
```

## Bootstrap des assembleurs (drifblim/drifloon)

Les ROMs sont distribuées en hex encodé. Pour les décoder :

```sh
python3 -c "
data = open('drifblim.rom.txt').read()
import sys; sys.stdout.buffer.write(bytes.fromhex(''.join(c for c in data if c in '0123456789abcdef')))
" > drifblim.rom
```

## Syntaxe Uxntal — notes importantes

- Les littéraux de caractères s'écrivent `"x` (pas `'x`, ancienne syntaxe supprimée)
- Référence : [wiki.xxiivv.com/site/uxntal.html](https://wiki.xxiivv.com/site/uxntal.html)

## uxnasm

`uxnasm` (l'assembleur original en C) a été **archivé le 28 décembre 2024**. Il n'est plus maintenu. L'assembleur recommandé est drifblim/drifloon.

---

## Architecture de xo.tal (3×3)

### Fonctionnement général

- **Joueur** : X (toujours en premier)
- **IA** : O (joue automatiquement après chaque coup du joueur)
- **Souris** : survol met à jour le curseur de cellule, clic gauche pose la pièce
- **Clavier** : flèches déplacent le curseur, A ou Espace pose la pièce
- **Bouton restart** : sprite "R" à (x=82, y=94), clic = nouveau jeu immédiat

### Couches d'affichage (Varvara)

| Couche | Contenu | Byte sprite |
|---|---|---|
| BG (fond) | Plateau, pièces X/O, curseur de cellule, bouton R | `#01`–`#03` |
| FG (avant) | Pointeur souris (flèche) | `#40`–`#43` (bit 6 = 1) |

Effacer le curseur FG = sprite `@fill` (tout à 1) avec `#40` → pixels FG transparents, BG intact.

### Coordonnées du plateau (3×3)

- Offset : 29 px (`GAMEBOARD-OFFSET2 = #001d`)
- Taille de cellule : 16 px (`CELL-SIDE = #10`)
- Grille : lignes à x/y ∈ {29, 45, 61, 77}
- Pièces X/O : offset supplémentaire +5 px (`HW-TO-PX-FOR-XO`)
- Conversion pixel → cellule : `(pixel - 29) / 16 + 1`

### IA (heuristique, 3×3)

Stratégie (par priorité) :
1. Gagner si possible (2 O + 1 vide sur une ligne)
2. Bloquer le joueur (2 X + 1 vide)
3. Prendre le centre
4. Prendre un coin
5. Toute case vide

Fonctions : `@ai-move`, `@ai-scan-for`, `@ai-check-perm`, `@ai-set-and-check`

Indices plateau : `(h-1)*3 + (w-1)` → 0..8

### Gestion souris

Device Mouse à `|90`. Format de `Mouse/state` : bit 0 = bouton gauche.

`@on-mouse` :
1. Curseur FG : efface à l'ancienne position, dessine à la nouvelle
2. Détection front montant (OVR + comparaison état précédent)
3. Bouton restart x=[82..89] y=[94..101] : toujours actif
4. Logique de jeu (seulement si `Controller/vector == on-controller`)

### Reset de partie

`@reset-game` efface d'abord les 9 sprites visuels + l'indicateur fin de partie (position h=4,w=2) via `place-figure` avec `cur-player="f"`, puis réinitialise les données. Appelé par `@off-controller` et par le bouton restart.

---

## Architecture de xo5.tal (5×5)

### Différences clés avec 3×3

| Paramètre | 3×3 | 5×5 |
|---|---|---|
| Condition de victoire | 3 alignés | **4 alignés** |
| Écran | 107×107 | **128×128** |
| Offset grille | 29 px | **16 px** |
| Lignes de grille | 4 × 2 directions | **6 × 2 directions** |
| Cellules | 9 | **25** |
| Bornes curseur clavier | 1–3 | **1–5** |
| Zone souris | x/y ∈ [29..76] | **x/y ∈ [16..95]** |
| Indicateur fin | h=4, w=2 | **h=6, w=3** |
| Bouton restart | x=82, y=94 | **x=104, y=100** |

### IA (heuristique, 5×5)

- Victoire en 4 alignés → vérifie **28 fenêtres de 4** (10 horiz + 10 vert + 4 diag SE + 4 diag NE)
- `@ai-scan-for` : 28 fenêtres × 4 permutations (3 cases pleines + 1 vide) = **112 appels**
- `@ai-check-perm4` / `@ai-set-and-check4` : variantes 4-cellules de la 3×3
- `@four-same` : remplace `three-same`, vérifie 4 valeurs identiques via `STH`/`STHr`
- `@board-full`, `@clear-cursors`, `@ai-any-empty` : tous implémentés en **boucles** (25 cellules)
- `@reset-game` : boucles imbriquées 5×5 pour le nettoyage visuel + zeroing board

### Indices plateau (5×5)

Nommage cellules : `cell_WH` (W=colonne 1-5, H=ligne 1-5).
Index flat : `(H-1)*5 + (W-1)` → 0..24, stocké dans `@board`.
