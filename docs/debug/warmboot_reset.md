Doubled sign of life
====================

Problem description
-------------------

Sometimes sign of life happens twice. It is caused by reset of platform during
warmboot (doing `rte_ctrl -pon` when the platform is in S5). Problem does
not occur when doing coldboot or reboot.

Initial ideas
-------------

[AGESA specification 44065 Rev. 3.04](https://support.amd.com/TechDocs/44065_Arch2008.pdf)
on page 38 and followings in the operational overview says:

> E — Main boot path. Proceed with full hardware initialization.
> Warm reset may be needed to instantiate new values into some registers.

Page 40, about AGESA software call entry points' duties:

```
AmdInitReset
    initialize heap ctl
    Primary ncHt link initialization
    SB initialization @ reset
    NB after HT

AmdInitEarly
    register load
    full HT initialization
    uCode patch load
    AP launch
    PwrMgmt Init
    NB post initialization
    Detect need for warm reset
```

It looks like moving SOL after call to `AmdInitEarly()` would fix the issue,
but then it gets printed after a relatively long period - user might think
that platform isn't booting, also it almost immediately disappears.

Building Debug version of AGESA
-------------------------------

Output of boot process with Debug version of AGESA (unrelated lines omitted
  for clarity):
```

coreboot-4.8-1064-g7321246a43-dirty Wed Jul 25 12:28:25 UTC 2018 romstage starting...
14-25-48Mhz Clock settings
(...)
PC Engines apu5
coreboot build 20182507
BIOS version 4.8-1064-g7321246a43-dirty
(...)
agesawrapper_amdinitreset() entry
CBFS: 'Master Header Locator' located CBFS at [200:7fffc0)
CBFS: Locating 'AGESA'
CBFS: Found @ offset 5ffdc0 size e6e52
CBFS: 'Master Header Locator' located CBFS at [200:7fffc0)
CBFS: Locating 'AGESA'
CBFS: Found @ offset 5ffdc0 size e6e52

AmdInitReset: Start


*** !!!AGESAcb_Agesa ***


[TP:C0]
    FCH Reset Data Block Allocation: [0x0], Ptr = 0x004001F0
Fch OEM config in INIT RESET Done

[TP:B1]


coreboot-4.8-1064-g7321246a43-dirty Wed Jul 25 12:28:25 UTC 2018 romstage starting...
14-25-48Mhz Clock settings
(...)
PC Engines apu5
coreboot build 20182507
BIOS version 4.8-1064-g7321246a43-dirty
(...)
agesawrapper_amdinitreset() entry
CBFS: 'Master Header Locator' located CBFS at [200:7fffc0)
CBFS: Locating 'AGESA'
CBFS: Found @ offset 5ffdc0 size e6e52
CBFS: 'Master Header Locator' located CBFS at [200:7fffc0)
CBFS: Locating 'AGESA'
CBFS: Found @ offset 5ffdc0 size e6e52

AmdInitReset: Start


*** !!!AGESAcb_Agesa ***


[TP:C0]
    FCH Reset Data Block Allocation: [0x0], Ptr = 0x004001F0
Fch OEM config in INIT RESET Done

[TP:B1]

AmdInitReset: Start


*** !!!AGESAcb_Agesa ***


[TP:C0]

AmdInitReset: End


[TP:C1]

[TP:C4]
(...)
```

It is clear that platrofm resets before `AmdInitEarly()`, somewhere in `AmdInitReset()`.
