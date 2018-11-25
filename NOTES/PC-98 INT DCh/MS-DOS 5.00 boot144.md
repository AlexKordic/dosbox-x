Entry point (MS-DOS 5.00) 1.44MB disk image (on my hard drive, boot144.dsk). Configuration menu option "nothing". 0ADC segment may change location based on configuration.

--

    0060:0124 WORD ACFh BX value (?))
    0060:0128 BYTE (?)
    0060:014E BYTE some sort of flag
    0060:0214 WORD:WORD 16-bit far pointer (0ADC:3126)
    0060:05E1 WORD stored DS value from caller
    0060:05DB WORD stored AX value from caller
    0060:05DD WORD stored SS value from caller
    0060:05DF WORD stored SP value from caller (after INT DCh int frame and PUSH DS)
    0060:05E3 WORD stored DX value from caller
    0060:05E5 WORD stored BX value from caller
    0060:0767 Stack pointer (from DOS segment), stack switches to on entry to procedure
    0060:36B3 INT DCh entry point
    0060:3B30 Subroutine called on INT DCh if 0060:014E is nonzero

--

INT DC = 60:36B3

    0060:36B3:
        if BYTE PTR cs:[014E] == 0 jmp 0060:36BE
        call 3b30
    0060:36BE:
        jmp far WORD PTR cs:[0214]

--

    0060:3B30: (called by INT DCh entry point if BYTE PTR CS:[014E] != 0
        PUSH DS, ES, CX, SI, DI
        DS:SI = WORD PTR cs:[3B28] (FFFF:0090)
        ES:DI = WORD PTR cs:[3B2C] (0000:0080)
        ZF = ((result=_fmemcmp(DS:SI,ES:DI,16)) == 0) (CX = 8, REP CMPSW)   ; <- Testing A20 gate, apparently?
        POP DI, SI, CX, ES, DS
        IF result == 0 JMP 3B4Ch (if ZF=1)
        return
    0060:3B4C: (if _fmemcmp(DS:SI,ES:DI,16) == 0)
        PUSH AX, BX
        AH = 5
        CALL FAR WORD PTR cs:[014F]     ; <- ??? Pointer so far has been either 0000:0000 or FFFF:FFFF
        POP BX, AX
        return
    ; ^ NOTE: Not sure, but this may be a call into HIMEM.SYS (XMS), in which case AH = 5 means Local Enable A20

--

    0ADC:00000A7C E2 12 F2 12 E2 12 CA 12 BC 12 F2 12 C3 12 D1 12  ................
    0ADC:00000A8C C3 12 D8 12 E8 01 00 CB B8 00 01 C3 E8 01 00 CB  ................
    
    0ADC:0A7C table (routine 0ADC:1201 return from CALL 129Dh)
        AX = 0x0000 DL = 0x00    0x12E2
        AX = 0x0000 DL = 0x01    0x12F2
        AX = 0x0001 DL = 0x00    0x12E2
        AX = 0x0001 DL = 0x01    0x12CA
        AX = 0x0002 DL = 0x00    0x12BC
        AX = 0x0002 DL = 0x01    0x12F2
        AX = 0x0003 DL = 0x00    0x12C3
        AX = 0x0003 DL = 0x01    0x12D1
        AX = 0x0004 DL = 0x00    0x12C3
        AX = 0x0004 DL = 0x01    0x12D8

--

    0ADC:0A9C: (CL=10h AH=00h, at this time CL == caller's DL and DS = DOS segment 60h)
        IF BYTE PTR DS:[0128] != 0 JMP AB5  (60:128)
        IF CL < 0x20 JMP AACh
        CALL 11B3h
    0ADC:0AAC: (CL < 0x20)
        BX = 0A00h
        CALL 0ACFh
        CALL NEAR BX (BX comes from subroutine ACFh)
        return
    0ADC:0AB5: (BYTE PTR DS:[0128] != 0) aka (60:128)
        (BYTE DS:[0128])++
        IF BYTE PTR DS:[0128] != 2 JMP 0ACAh
    0ADC:0AC0: (BYTE PTR DS:[0128] == 2)
        BX = 0A21h
        CALL 0ACFh
    0ADC:0ACA: (jmp here from BYTE PTR DS:[0128] != 2 compare under 0ADC:0AB5)
        WORD PTR DS:[0124] = BX (BX comes from subroutine ACFh) (60:124)
        CALL NEAR WORD PTR DS:[0124]
        return

--

    0ADC:11B3: (CL=10h AH=00h, at this time CL == caller's DL and DS = DOS segment 60h)
        IF BYTE PTR DS:[011C] < 0x50 JMP 11C7h ; (60:11C cursor X position)
        IF BYTE PTR DS:[0117] == 0 JMP 11C2h ; (60:117 line wrap flag)
    0ADC:11C7: (60:11C < 0x50)
        IF BYTE PTR DS:[008A] == 0 JMP 1201h ; (60:8A kanji / graph mode flag)
        IF BYTE PTR DS:[0115] != 0 JMP 11F3h ; (60:115 kanji upper byte storage flag)
        IF CL < 0x81 JMP 1201h
        IF CL < 0xA0 JMP 11E9h
        IF CL < 0xE0 JMP 1201h
        IF CL >= 0xFD JMP 1201h
        BYTE PTR DS:[0116] = CL (60:116 kanji upper byte)
        BYTE PTR DS:[0115] = 1 (60:115 kanji upper byte storage flag)
    0ADC:11E9:
        return
    0ADC:11F3: (60:115 kanji upper byte storage flag set)
        CH = DS:[0116] (60:116 kanji upper byte)
        CALL 1236h
        (TODO finish this later, at 0ADC:11FA)
    0ADC:1201:
        CH = 0
        CALL 1260h
        CALL 129Dh
        DI = BX
        AX = ((AX * 2) + DL) * 2
        BX = (AX + 0A7Ch)
        BX = WORD PTR CS:[BX]
        PUSH ES
        ES = WORD PTR DS:[0032] ; (60:32 appears to be Text VRAM segment A000h)
        AX = CX
        CALL NEAR BX
        POP ES
        BYTE PTR DS:[0115] = 0 (60:115 upper byte storage flag)
        BYTE PTR DS:[011C] += AL (60:11C cursor X coordinate)
        CALL 1535h
        return

--

    0ADC:12E2 (DS = DOS segment 60h, ES = Text VRAM segment A000h, AX = character code, DI = memory offset)
        WORD PTR ES:[DI] = AX ; write character code
        DI += 0x2000
        WORD PTR ES:[DI] = WORD PTR DS:[013C] (60:13C display attribute in extended attribute mode) ; write attribute code
        AL = 1 (this indicates to caller to move cursor X position 1 unit to the right)
        return

--

    0ADC:3126:
        PUSH DS
        DS = WORD PTR CS:[0030] = DOS segment 60h
        WORD PTR DS:[05E1] = caller DS
        Store caller AX, SS, SP, DX, BX into DS: [5DB], [5DD], [5DF], [5E3], [5E5]
        SS:SP = WORD PTR CS:[0030] (DOS segment 60h : offset 767h)
        CLD, STI
        PUSH ES, BX, CX, DX, SI, DI
        BYTE PTR DS:[00B4] = 1
    0ADC:3154:
        CL -= 9, JMP to 0ADC:3180 if carry (if caller CL < 9)
    0ADC:315E:
        IF CL < 0x0D, JMP TO 0ADC:3173 (if caller CL < (0x16 = 0x0D+9))
    0ADC:3163:
        IF CL < 0x77, JMP TO 0ADC:3180 (if caller CL < (0x80 = 0x77+9))
    0ADC:3168:
        IF CL > 0x79, JMP TO 0ADC:3180 (if caller CL > (0x82 = 0x79+9))
    0ADC:316D: (JMPed here if CL >= 0x80 && CL <= 0x82)
        CL = (CL - 0x77) + 0x0D
    0ADC:3173: (JMPed here if CL >= 0x09 && CL <= 0x15)
        SI = (CL * 2) + 0x3A5C
    0ADC:317D:
        CALL NEAR WORD PTR 0ADC:[SI]
    0ADC:3180:
        CALL NEAR 0ADC:4080
    0ADC:3183:
        POP DI, SI, DX, CX, BX, ES
        DS = WORD PTR CS:[0030] = DOS segment 60h
        Restore caller AX, SS, SP, DX, BX from [5DB], [5DD], [5DF], [5E3], [5E5]
        POP DS (restore caller DS)
        IRET

--

    0ADC:378E: (CL=10h entry point)
        BX = WORD PTR DS:[05DB] caller's AX value
        BX = (BX >> 8)   (BX = caller's AH value)
        ADDR = (BX * 4) + 3A7Ch
        AX = WORD PTR CS:[ADDR]   (0ADC:[ADDR])
        BX = WORD PTR CS:[ADDR+2]
        CALL NEAR AX
        return

--

    0ADC:37A7: (CL=10h AH=00h entry point)
        CL = DL
        CALL NEAR BX (BX is 0A9C, no other case)

--

    0ADC:3A5C table contents.
    Note the INT DCh code maps:
        CL = 0x09..0x15 to table index 0x00..0x0C
        CL = 0x80..0x82 to table index 0x0D..0x0F

    0ADC:00003A5C B8 3A 5B 35 A4 31 A5 31 DF 32 1B 36 44 37 8E 37
    0ADC:00003A6C F0 37 6E 38 F7 38 85 3B 52 3C 27 39 B5 39 10 3A

    CL = 0x09    0x3AB8
    CL = 0x0A    0x355B
    CL = 0x0B    0x31A4
    CL = 0x0C    0x31A5
    CL = 0x0D    0x32DF
    CL = 0x0E    0x361B
    CL = 0x0F    0x3744
    CL = 0x10    0x378E
    CL = 0x11    0x37F0
    CL = 0x12    0x386E
    CL = 0x13    0x38F7
    CL = 0x14    0x3B85
    CL = 0x15    0x3C52
    CL = 0x80    0x3927
    CL = 0x81    0x39B5
    CL = 0x82    0x3A10

--
    
    0ADC:00003A7C A7 37 9C 0A AC 37 00 00 C3 37 00 00 CC 37 FA 0A  .7...7...7...7..
    0ADC:00003A8C D7 37 90 0B D7 37 99 0B DA 37 52 0C DA 37 75 0C  .7...7...7R..7u.
    0ADC:00003A9C DA 37 9C 0C DA 37 C3 0C E1 37 2E 0D E1 37 72 0D  .7...7...7...7r.
    0ADC:00003AAC DA 37 4E 0E DA 37 72 0E E8 37 2D 0B 8B 16 E3 05  .7N..7r..7-.....
    0ADC:00003ABC A1 DB 05 3D 00 00 74 10 3D 01 00 74 20 3D 10 00  ...=..t.=..t =..
    0ADC:00003ACC 74 4E 3D 11 00 74 49 C3 8E 06 E1 05 8B FA 2E 8E  tN=..tI.........

    0ADC:3A7C Call table (address, param)

    Calling convention on entry: AX = subroutine address, BX = param
    
    AH = 0x00    0x37A7    0x0A9C  ; 0ADC:00003A7C
    AH = 0x01    0x37AC    0x0000
    AH = 0x02    0x37C3    0x0000
    AH = 0x03    0x37CC    0x0AFA
    AH = 0x04    0x37D7    0x0B90  ; 0ADC:00003A8C
    AH = 0x05    0x37D7    0x0BDA
    AH = 0x06    0x37DA    0x0B99
    AH = 0x07    0x37DA    0x0C75
    AH = 0x08    0x37DA    0x0C9C  ; 0ADC:00003A9C
    AH = 0x09    0x37DA    0x0CC3
    AH = 0x0A    0x37E1    0x0D2E
    AH = 0x0B    0x37E1    0x0D72
    AH = 0x0C    0x37DA    0x0E4E  ; 0ADC:00003AAC
    AH = 0x0D    0x37DA    0x0E72
    AH = 0x0E    0x37E8    0x0B2D
    AH = 0x0F    0x168B    0x05E3

--

    0ADC:4080:
        DS = DOS kernel segment 60h from 0ADC:0030
        BYTE PTR DS:[00B4] = 0
        (other cleanup, not yet traced)
        CALL 0060:3C6F
        RET

--

    0ADC:0030 WORD DOS kernel segment (60h)
    0ADC:3A5C array of WORD values, offsets of procedures for each value of CL.
    0ADC:3A7C array of WORD value pairs (address, parameter). NOTE: Lack of range checking!
