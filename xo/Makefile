UXN2  = ../uxn2 -2x
UXN11  = ../uxn11
UXNCLI = ../uxncli
ASM    = ../drifloon.rom

.PHONY: all run run5 clean

all: xo.rom xo5.rom

xo.rom: xo.tal $(ASM)
	cat xo.tal | $(UXNCLI) $(ASM) > xo.rom

xo5.rom: xo5.tal $(ASM)
	cat xo5.tal | $(UXNCLI) $(ASM) > xo5.rom

run3: xo.rom
	$(UXN2) xo.rom

run: xo5.rom
	$(UXN2) xo5.rom

clean:
	rm -f xo.rom xo5.rom
