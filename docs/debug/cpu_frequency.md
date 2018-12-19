# Debug notes for CPU frequency issues

[GitHub issue](https://github.com/pcengines/coreboot/issues/196)

Most of information is taken from [BKDG](https://www.amd.com/system/files/TechDocs/52740_16h_Models_30h-3Fh_BKDG.pdf),
unless noted otherwise.

## Reproducing the issue

Steps described in linked GitHub issue were used.

For some reason, our installation of pfSense doesn't include `cpuctl` module,
so `turbostat` didn't work at all. According to its source code, the `Bzy_MHz`
field shows real CPU frequency. It is counted as ratio of clock ticks of CPU
divided by clock ticks of P0 as a reference over some period of time, and scaled
appropriately. It is thus a good measurement of real CPU frequency.

Another approach was to indirectly compare CPU frequencies by comparing bogo ops
from `stress-ng`. This method can give different results depending on system
load or interrupts, but can be used as an approximation. In our tests, bogo ops
after the issue occurred were slightly lower than those reported on GitHub
(1250-1350 vs. 1407), however they were definitely lower than when the issue
did not happen (~1950). This was mostly consistent with what was described on
GitHub when it comes to triggering this issue.

However, after a couple of days we couldn't reproduce this anymore, no matter
which platform or firmware version was used. Every test showed high bogo ops
value (1900-2050), with no difference after a couple of days of runtime and
different ways of starting the system (cold boot, warm boot, reset, reboot from
pfSense and another operating system).

## P-states

AMD processors have 2 sets of P-state numeration: software and hardware. They
both start with P0 being highest-performance accessible, but hardware P0 isn't
the same as software P0. On the software side additional boost states (Pb0, Pb1)
are used. All boosted P-states are always higher performance than non-boosted
P-states. Hardware P0 is software Pb0. Rest of states is mapped 1-to-1, with the
same names corresponding to different states, which can be confusing. Number of
boost states is written in `D18F4x15C[NumBoostStates]`, in case of apu it is 2.
**BIOS should not provide ACPI `_PSS` entries for boosted P-states.**

Boost states cannot be requested directly, some conditions that must be met:

* P0 (software) requested
* boost enabled (i.e. boost states exist and not disabled in `MSRC001_0015[CpbDis]`,
  this bit is clear in our case so boost isn't disabled here)
* actual demand on CPU
* no upper limit set (`D18F4x13C[SmuPstateLimit]`, set as 0 so no limit)
* additional hardware limits (these values vary, most of them is set by AGESA):
  * temperature (temperature limit is set in `D18F3x64` as 105 &deg;C)
  * TDP - whole CPU (set as a dynamic in `D18F3x1FC`, there is a dead link to
    section 2.5.11.8 in BKDG)
  * power consumption - individual cores (power required by core is also
    calculated or hard-coded in AGESA, but a P-state would be disabled if it
    were to require more power than a board can supply)

CPU can temporarily go above the TDP for one core, given that enough of other
cores are halted or waiting on IO operation. This is configured in `D18F4x16C`.

Lower limits can be set in `D18F3x64[HtcPstateLimit]`, `D18F3x68[SwPstateLimit]`
and `D18F3xDC[HwPstateLimit]` - all of these use hardware numeration. Also in
BKDG among other limitations "APML" (Advanced Platform Management Link?) was
mentioned with a dead link to `D18F3xC4[PstateLimit]`.

There is a maximum of 8 states, but only 5 are used in apu:

| Software | Hardware | Frequency | Register
|----------|----------|-----------|--------------
| Pb0      | P0       | 1.4 GHz   | MSRC001_0064
| Pb1      | P1       | 1.2 GHz   | MSRC001_0065
| P0       | P2       | 1.0 GHz   | MSRC001_0066
| P1       | P3       | 800 MHz   | MSRC001_0067
| P2       | P4       | 600 MHz   | MSRC001_0068
| n/a      | n/a      |           | MSRC001_0069
| n/a      | n/a      |           | MSRC001_006A
| n/a      | n/a      |           | MSRC001_006B

Problem is that platform doesn't go back to 1.0 GHz, so it isn't probably
connected to boost. However boost states were not observed despite no visible
limits (except for dynamic TDP, not sufficiently described in BKDG). This could
be due to the nature of checking for P-states - perhaps a function that reads
registers uses IO operations which don't require computing power as much.

Other MSRs directly connected to P-states:

* MSRC001_0061 - current P-state limit, P-state max value - read only.
* MSRC001_0062 - P-state control, write to this register requests a change.
Actual change might not happen if it exceeds any of set limits or if another
core on the same voltage/frequency domain uses different state.
* MSRC001_0063 - current P-state. Uses software numbering. May not be accurate
after warm reset, if it happened during state change (might be connected to
reboot issues).
* MSRC001_0071 - COFVID status. This register has fields describing real value
of P-state, current P-state limit (2 in this case, so no boost is allowed),
startup P-state, maximum frequency of CPU and NB, current frequency and voltage.
All P-states here use hardware numbering.

#### Values obtained from registers

Values of the mentioned registers were read using BITS, `lspci -xxxx` (Debian)
and `pciconf` (pfSense). `pciconf` doesn't allow to read registers above 0x100
(extended PCI configuration space) so `D18F4` was not checked there. Also MSRs
were only read and written with BITS - accessing them from OS would require a
kernel module and could mess/get messed by power management of OS.

Results seems to be consistent across different platforms (except for small
voltage differences), OSes and warm/cold boots.

| Register     | Value              | Decoded
|--------------|-------------------:|-------------------------
| D18F3x64     |         0x426a0025 | HtcPstateLimit = 4 (low, HW)<br>HtcHystLimit = 2<br>HtcTmpLimit = 0x6a = 105 &deg;C<br>HtcActSts = 1 *processor entered HTC since reset*<br>HtcAct = 0 *processor is not in HTC state currently*<br>HtcEn = 1
| D18F3x68     |         0x40000000 | SwPstateLimit = 4 (low, HW)<br>SwPstateLimitEn = 0
| D18F3xC4     |         0x00000000 | *mentioned but not described in BKDG*
| D18F3xDC     |         0x68786400 | NbsynPtrAdjPstate = 2<br>NbsynPtrAdjLo = 5<br>CacheFlushOnHaltTmr = 0xf<br>NbsynPtrAdj = 6<br>HwPstateMaxVal = 4 (low, HW)
| D18F4x110    |         0x000c4014 | MinResTmr = 0x62 = 98<br>CSampleTimer = 0x14 = 20 *(~10 ms)*
| D18F4x13C    |         0x00000001 | SmuPstateLimitEn = 1<br>SmuPstateLimit = 0 (high, HW)
| D18F4x15C    |         0x00000189 | BoostLock = 0<br>CstatePowerEn = 1<br>ApmMasterEn = 1<br>NumBoostStates = 2<br>BoostSrc = 1 *use of Pb0 and Pb1 enabled*
| D18F4x16C    |         0x000024bc | CstateCores = 1 *whether CstateCnt describes cores or compute units, 1 = cores*<br>CstateCnt = 2 *how many cores/CUs need to be in CC6 for boosting others*<br>CstateBoost = 2 (HW) *core needs to be in this **P**-state before being boosted*<br>ApmTdpLimitSts = 1<br>ApmTdpLimitIntEn = 1<br>TdpLimitDis = 1
| D18F5x84     |         0x0e0ef003 | DdrMaxRateEnf = 0xe<br>DdrMaxRate = 0xe<br>DctEn = 0xf<br>CmpCap = 3 *(4 cores)*
| D18F5xE0     |         0x0000xxx1 | RunAvgRange = 1 *(interval = 40ms)*<br>*bits marked with x are reserved, they change between reads*
| MSRC001_0061 | 0x0000000000000020 | CurPstateLimit = 0 (high, SW)<br>PstateMaxVal = 2 (low, SW)
| MSRC001_0062 | 0x0000000000000000 | PstateCmd = 0 *see notes below*
| MSRC001_0063 | 0x0000000000000000 | CurPstate = 0 *see notes below*
| MSRC001_0064 | 0x8000025f0000b84c | PstateEn = 1<br>IddDiv = 2<br>IddValue = 0x5f<br>NbPstate = 0<br>CpuVid = 0x5c<br>CpuDid = 1 *(divide by 2)*<br>CpuFid = 0xc *(COF = 1400 MHz)*
| MSRC001_0065 | 0x8000024d0000c848 | PstateEn = 1<br>IddDiv = 2<br>IddValue = 0x4d<br>NbPstate = 0<br>CpuVid = 0x64<br>CpuDid = 1 *(divide by 2)*<br>CpuFid = 8 *(COF = 1200 MHz)*
| MSRC001_0066 | 0x800002700000d844 | PstateEn = 1<br>IddDiv = 2<br>IddValue = 0x70<br>NbPstate = 0<br>CpuVid = 0x6c<br>CpuDid = 1 *(divide by 2)*<br>CpuFid = 4 *(COF = 1000 MHz)*
| MSRC001_0067 | 0x8000025f0040e040 | PstateEn = 1<br>IddDiv = 2<br>IddValue = 0x5f<br>NbPstate = 1<br>CpuVid = 0x70<br>CpuDid = 1 *(divide by 2)*<br>CpuFid = 0 *(COF = 800 MHz)*
| MSRC001_0068 | 0x8000024b0040ece0 | PstateEn = 1<br>IddDiv = 2<br>IddValue = 0x4b<br>NbPstate = 1<br>CpuVid = 0x76<br>CpuDid = 3 *(divide by 8)*<br>CpuFid = 0x20 *(COF = 600 MHz)*
| MSRC001_0069<br>MSRC001_006A<br>MSRC001_006B | 0x000000000041fe00 | PstateEn = 0
| MSRC001_0071 | 0x3a1c00027442d844 | MaxNbCof = 7 *(700 MHz)*<br>CurPstateLimit = 2 (high, HW)<br>MaxCpuCof = 0xe = 14 *(1400 MHz)*<br>StartupPstate = 2 (HW) *cold reset, may be different after reboot/reset*<br>CurNbVid = 0x74<br>NbPstateDis = 0<br>CurPstate = 2 (HW) *see notes below*<br>CurCpuVid = 0x6c<br>CurCpuDid = 1<br>CurCpuFid = 4

In the table SW means software numbering, HW - hardware. High limit means
an upper limit on frequency (performance), lowest P-state number. COF (current
operating frequency) is calculated as `100 * ((CpuFid + 0x10) / (2^CpuDid))`.
Transitions between P-states were working as expected. To change P-state to the
lower frequency following steps were taken:

1. Write to MSRC001_0062 with requested, higher P-state number (SW).
2. Read from MSRC001_0062 should return written state, as long as it is within
limits.
3. Read from MSRC001_0063 returns 0, MSRC001_0071 returns previous value - no
P-state transition occurred because of frequency and voltage domains.
4. Write to MSRC001_0062 of other cores.
5. After all cores have requested change the actual P-state transition takes
place. Reads from MSRC001_0063 and MSRC001_0071 return expected values.

Changing P-state to higher performance results in immediate change, as frequency
and voltage domains are tailored to the most demanding core.

Actual frequency and/or voltage can be different than in the state pointed by
CurPstate in MSRC001_0063 after a warm reset that occurred during plane
transition. In this case current value can be that before or after transition.
Firmware is required to transition the processor to valid COF and VID settings.
This can be the source of some reboot problems, but it is hard to test in a
reliable way - reset is asynchronous event that have to take place in very short
amount of time during transition.

## Voltage regulator

CPU communicates with voltage regulator using SVI2 (Serial VID Interface 2.0).
Typically, after all frequency and voltage dependencies described above are
resolved, transition to higher performance follows a sequence:

1. SVI2 command is send to the voltage regulator, `D18F5x12C[Svi2CmdBusy]`
   is set.
2. Voltage regulator sets the requested voltage, waits for it to stabilize and
   clears `Svi2CmdBusy`.
3. After `Svi2CmdBusy` is clear frequency change is performed.

Transition towards lower performance starts with frequency change and doesn't
have to wait for it to finish before changing voltage. Because of these
restrictions it is impossible for CPU to work with high frequency and too low
voltage for that frequency.

Multiple requests can be grouped:

> * If multiple commands are issued that affect the P-state of a domain prior
>   to when the processor initiates the change of the P-state of that domain,
>   then the processor operates on the last one issued.
> * Once a P-state change starts, the P-state state machine (PSSM) continues
>   through completion unless interrupted by a PWROK deassertion. If multiple
>   P-state changes are requested concurrently, the PSSM may group the
>   associated VID changes separately from the associated COF changes.

Setting `D18F5x12C[WaitVidCompDis]` changes behaviour of transition towards
higher performance - instead of waiting for voltage regulator to report end of
transition next request can be made after a defined time period
(`D18F3xD8[VSRampSlamTime]`).

## Findings from logs obtained from community

Registers values from community revealed that CPU had voltage of (software) P0,
but its frequency was stuck at a value of P2. Voltage change to P0 was requested
but not reported as done by voltage regulator for some reason.

It is possible that under certain circumstances an infinite loop occurs during
boot - CPU is not requested to transition to P0 as its voltage already is equal
to that of P0, so AGESA waits for frequency change to catch up, which doesn't
happen. An explicit request to transition to P0 during boot could help with
[the other issue](https://github.com/pcengines/apu2-documentation/issues/64).

It is unclear why some platforms are influenced and others are not. Maybe
different batch of voltage regulator (chip U7 on the bottom side of board)
was used?

#### ACPI tables

All required objects for P-states are present, even the optional `_PPC`. Only
difference between actual tables and recommendations from BKDG is entry
`DUTY_WIDTH` in Fixed ACPI Description Table (FADT) - AMD recommends value of 0,
while real entry has 3. Description of this entry from [ACPI specification](http://www.acpi.info/DOWNLOADS/ACPIspec-2-0a.pdf):

>The bit width of the processor’s duty cycle setting value in the P_CNT
>register. Each processor’s duty cycle setting allows the software to select
>a nominal processor frequency below its absolute frequency as defined by:
>
>    THTL_EN = 1
>
>    BF*DC/(2^DUTY_WIDTH)
>
>    Where:
>
>BF – Base frequency
>
>DC – Duty cycle setting
>
>When THTL_EN is 0, the processor runs at its absolute BF.
>A DUTY_WIDTH value of 0 indicates that processor duty cycle is not
>supported and the processor continuously runs at its base frequency.

## C-states

Only theoretical research on registers was done from C0, and everything seems to
be correct. Practical research (i.e. transitions between states) is impossible,
because when CPU is in lower C-state it doesn't process instructions and thus
cannot report any of its register values.

As for ACPI tables, we couldn't find `_CRS` object. `P_LVL2_LAT` showed a
value 0f 0x65 = 101, which disables C2 state according to [specification](http://www.acpi.info/DOWNLOADS/ACPIspec-2-0a.pdf).
First recorded value of this field actually showed 0x64 = 100, that is as
specified in BKDG, but all subsequent logs show 0x65. This could either be an
error with serial connection or this value did change, which could possibly
explain sudden problems with reproducing. A look at the source code however
shows that it is set as 0x65, against instructions from BKDG. If this value was
indeed different it had to be set by some other agent (payload, OS, AGESA).

#### Fix

Setting `D18F5x12C[WaitVidCompDis]` resolved both this issue as well as problems
with reboot. This bit is protected by `D18F2x1B4[SmuCfgLock]` after system or a
payload starts, but is cleared during init.
