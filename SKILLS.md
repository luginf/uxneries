# Skills Uxn

Commandes et recettes pour travailler avec la plateforme Uxn.

## Assembler un fichier .tal

```sh
# Mode graphique (uxn11 + drifblim)
./uxn11 drifblim.rom mon_prog.tal mon_prog.rom

# Mode headless (uxncli + drifloon)
# IMPORTANT : ne jamais rediriger stderr dans la ROM (2>&1 corrompt le binaire)
cat mon_prog.tal | ./uxncli drifloon.rom > mon_prog.rom
```

## Lancer un ROM

```sh
./uxn11 mon_prog.rom
```

## Assembler et lancer en une commande

```sh
./uxn11 drifblim.rom mon_prog.tal mon_prog.rom && ./uxn11 mon_prog.rom
```

## Makefile type pour un projet Uxntal

```makefile
ROM    = mon_prog.rom
SRC    = mon_prog.tal
UXN11  = ./uxn11
UXNCLI = ./uxncli
ASM    = drifloon.rom

.PHONY: all run clean

all: $(ROM)

$(ROM): $(SRC) $(ASM)
	cat $(SRC) | $(UXNCLI) $(ASM) > $(ROM)

run: $(ROM)
	$(UXN11) $(ROM)

clean:
	rm -f $(ROM)
```

## Recompiler uxn11 depuis les sources

```sh
git clone https://git.sr.ht/~rabbits/uxn11
cd uxn11
cc -DNDEBUG -O2 -g0 -s src/uxn11.c -lX11 -lutil -o bin/uxn11
```

## Recompiler uxn2 depuis les sources

```sh
git clone https://git.sr.ht/~rabbits/uxn2
cd uxn2
cc $(sdl2-config --cflags) -DNDEBUG -O2 -g0 -s src/uxn2.c -o bin/uxn2 $(sdl2-config --libs)
```

## Recompiler uxncli depuis les sources

```sh
git clone https://git.sr.ht/~rabbits/uxncli
cd uxncli
cc -DNDEBUG -O3 -g0 -s src/uxncli.c -o bin/uxncli
```

## Recréer drifblim.rom / drifloon.rom

Les ROMs d'assembleur sont distribuées sous forme hex dans les repos.

```sh
# drifblim.rom (pour uxn11) — depuis le repo uxn11
python3 -c "
data = open('etc/utils/drifblim.rom.txt').read()
import sys; sys.stdout.buffer.write(bytes.fromhex(''.join(c for c in data if c in '0123456789abcdef')))
" > bin/drifblim.rom

# drifloon.rom (pour uxncli) — depuis le repo uxncli
python3 -c "
data = open('etc/utils/drifloon.rom.txt').read()
import sys; sys.stdout.buffer.write(bytes.fromhex(''.join(c for c in data if c in '0123456789abcdef')))
" > bin/drifloon.rom
```

## Repos officiels

| Outil | URL |
|---|---|
| uxn11 (VM graphique X11) | https://git.sr.ht/~rabbits/uxn11 |
| uxn2 (VM graphique SDL2) | https://git.sr.ht/~rabbits/uxn2 |
| uxncli (VM headless) | https://git.sr.ht/~rabbits/uxncli |
| drifblim (assembleur) | https://git.sr.ht/~rabbits/drifblim |
| awesome-uxn (ressources) | https://github.com/hundredrabbits/awesome-uxn |
| Référence Uxntal | https://wiki.xxiivv.com/site/uxntal.html |
| Varvara (devices) | https://wiki.xxiivv.com/site/varvara.html |

## Syntaxe Uxntal — rappels

| Élément | Exemple | Description |
|---|---|---|
| Hex littéral | `#2a` | Pousse l'octet 0x2a |
| Hex court | `#0034` | Pousse le short 0x0034 |
| Char littéral | `"x` | Pousse le code ASCII de 'x' |
| Label absolu | `;label` | Adresse 16-bit du label |
| Label relatif | `,label` | Offset 8-bit signé (portée ±127 octets) |
| Label zp | `.label` | Adresse zero-page 8-bit (device port) |
| Label local | `&label` | Scoped au `@` courant → `fonction/label` |
| Macro | `%NOM { ... }` | Substitution textuelle |
| Commentaire | `( texte )` | Ignoré par l'assembleur |
| Device write | `DEO` / `DEO2` | Écriture 8/16-bit sur device |
| Device read | `DEI` / `DEI2` | Lecture 8/16-bit depuis device |

## Sauts en Uxntal

| Instruction | Type | Portée |
|---|---|---|
| `,&label JCN` | Conditionnel relatif | ±127 octets |
| `;&label JCN2` | Conditionnel absolu | Toute adresse |
| `,&label JMP` | Inconditionnel relatif | ±127 octets |
| `;&label JMP2` | Inconditionnel absolu | Toute adresse |
| `JMP2r` | Retour de sous-routine | Return stack |

## Devices Varvara courants

| Adresse | Device | Ports clés |
|---|---|---|
| `00` | System | `/r /g /b` (palette 4 couleurs) |
| `10` | Console | `/write` |
| `20` | Screen | `/width /height /x /y /addr /pixel /sprite` |
| `80` | Controller | `/vector /button /key` |
| `90` | Mouse | `/vector /x /y /state` (bit0=gauche) |
| `a0` | File A | — |
| `b0` | File B | — |
| `c0` | DateTime | — |

## Couches d'affichage Varvara (Screen)

Le Screen Varvara a deux couches (BG et FG). Le byte de commande pour `/pixel` et `/sprite` :

| Bit 6 | Couche | Effet couleur 0 |
|---|---|---|
| 0 | BG (fond) | Efface (couleur de fond) |
| 1 | FG (avant) | **Transparent** (laisse voir le BG) |

Exemples pour `/sprite DEO` en mode 1bpp :
- `#01` = BG, couleur 1 pour les pixels "1", transparent pour les "0"
- `#03` = BG, couleur 3
- `#40` = FG, couleur 0 → pixels 1-bits **transparents** (efface la FG)
- `#41` = FG, couleur 1
- `#43` = FG, couleur 3

**Pattern curseur souris** (non-destructif) :
```
( effacer ancien curseur sans toucher le BG )
;fill .Screen/addr DEO2  #40 .Screen/sprite DEO
( dessiner nouveau curseur sur la FG )
;mouse-cursor .Screen/addr DEO2  #43 .Screen/sprite DEO
```

## Passage de valeur 1-octet vers Screen/x ou Screen/y (2 octets)

```
( ptr-px est 1 octet (0..106), Screen/x attend un short )
#00 ;ptr-px LDA .Screen/x DEO2
( ou avec sauvegarde simultanée : )
.Mouse/x DEI2 NIP DUP ;ptr-px STA #00 SWP .Screen/x DEO2
```

## Détection front montant souris (edge detection)

```
( retourne 1 seulement au moment de l'appui, pas en maintien )
.Mouse/state DEI #01 AND   ( état courant )
;mouse-last-state LDA      ( état précédent )
OVR ;mouse-last-state STA  ( sauvegarder l'état courant )
EQU ;&done JCN2            ( pas de changement → ignorer )
;mouse-last-state LDA #00 EQU ;&done JCN2  ( relâchement → ignorer )
( ici : appui confirmé )
```
