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

Debug version of AGESA
----------------------

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

PMxC0 S5/Reset Status
---------------------

As the issue depends on state from which platform is booted I looked through power
management registers in [BKDG for AMD Family 16h Models 30h-3Fh Processors, 52740 Rev 3.06](https://support.amd.com/techdocs/52740_16h_models_30h-3fh_bkdg.pdf).
Registers of interest were those with `Cold reset` value, as they were remembered
through resets and transitions to/from S5, but not after full power cycle. The most
important one is PMxC0 S5/Reset Status:
> This register shows the source of previous reset.
This register is also defined in AGESA headers in coreboot repository:
`#define FCH_PMIOxC0_S5ResetStatus          0xFED803C0ul`

Reading content of PMxC0 right before printing sign of life after different ways
of (re)booting the platform resulted in the following values:

| Entering/leaving S5 | PMxC0 during SOL | Bits set                     |
|---------------------|------------------|------------------------------|
| rte_ctrl -rel *     | 0x00000800       | SlpS3ToLtdPwrGdEn            |
| poff/pon **         | 0x40200402       | SleepReset, FourSecondPwrBtn |
| thermal/pon **      | 0x40200401       | SleepReset, ThermalTrip      |
| reboot              | 0x40080400       | DoReset                      |
| rte_ctrl -reset     | 0x40010400       | UsrReset                     |
| halt/pon            | 0x40200400       | SleepReset                   |
|                     | 0x001003FF       | S5ResetStatus_All_Status     |

\*) not S5, included for completeness

\*\*) results in doubled SOL. Value in table corresponds to first iteration, then
platform is reset and during second boot PMxC0 has the same value as after `reboot`

Table is stripped from some common bits that don't seem to affect the problem:
- 0x40000000 - reserved
- 0x00000400 - PwrGdDwnBeforeSlp3 - [BKDG, p. 932](https://support.amd.com/techdocs/52740_16h_models_30h-3fh_bkdg.pdf):
> **PwrGdDwnBeforeSlpS3**. Read-write. Cold reset: 0. BIOS: 1.
> 1=Delay SLP_S3 by 64 μs and also qualify the FCH PwrGood with SLP_S3;
> This allows internal logic to put signals into correct states before
> turning off the S0 power.

Last row shows value of `FCH_PMIOxC0_S5ResetStatus_All_Status`, which is what
AGESA checks when deciding whether platform needs a reset or not.
