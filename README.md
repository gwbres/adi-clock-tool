# ADI-AD9546 

[![Python application](https://github.com/gwbres/adi-ad9546/actions/workflows/python-app.yml/badge.svg)](https://github.com/gwbres/adi-ad9546/actions/workflows/python-app.yml)
[![PyPI version](https://badge.fury.io/py/adi-ad9546.svg)](http://badge.fury.io/py/adi-ad9546)

Set of tools to interact & program AD9546/45 integrated circuits, by Analog Devices.

Use [these tools](https://github.com/gwbres/adi-ad9548)
to interact with AD9548/47 older chipsets.

These scripts are not Windows compatible.   
These scripts expect a `/dev/i2c-X` entry, they do not manage the device
through SPI at the moment.

## Install 

```shell
python setup.py install
```

## Dependencies

* python-smbus

Install requirements with

```shell
pip3 install -r requirements.txt
```

## API

* Each application comes with an `-h` help menu.  
Refer to help menu for specific information. 
* `flag` is a mandatory flag
* `--flag` is optionnal: action will not be performed if not requested

For complex flag values (basically involving white spaces), for example 
`ref-input --coupling`, don't forget to encapsulate with inverted commas:

```shell
ref-input.py \
    --ref a \ # simple, one word
    --coupling "AC 1.2V" # 'complex' but meaningful value
ref-input.py \
    --ref aa \ # simple, one word
    --coupling "internal pull-up" # 'complex' but meaningful value
```

Flag values are case sensitive and must be exactly matched.
It is not possible to pass a non supported / unknown flag value,
scripts will reject those with a runtime error.

## AD9545 / 46

These scripts are developped and tested with an AD9546 chip.   
AD9545 is pin & register compatible, so it should work.   
It is up to the user to restrain to restricted operations in that scenario.

## Utilities

* `calib.py`: calibrates core portions of the clock. Typically required
when booting or a new setup has just been loaded.
* `distrib.py`: controls clock distribution and output signals.
Includes signal paths and output pins muting operations.
* `irq.py`: IRQ clearing & masking operations 
* `misc.py`: miscellaneous operations
* `mx-pin.py`: Mx programmable I/O management 
* `pll.py`: APLLx and DPLLx cores management. Includes
free running + holdover manual forcing operation
* `power-down.py` : power saving and management utility
* `ref-input.py`: reference & input signals management
* `regmap.py`: load or dump a register map preset
* `regmap-diff.py`: loaded / dumped regmap differentiator (debug tool)
* `reset.py`: device reset operations
* `status.py` : status monitoring, includes IRQ flag reports and onboard temperature reading
* `sysclk.py` : sys clock control & management tool

See at the bottom of this page for typical configuration flows.

## Register map

`regmap.py` allows the user to quickly load an exported
register map from the official A&D graphical tool.
* Input/output is `json`
* `i2c` bus must be specified
* `i2c` slave address must be specified 
* `--quiet` to disable the stdout progress bar

```shell
regmap.py -h
# load a register map (on bus #0 @0x48)
regmap.py 0 0x48 --load test.json
```

Export current register map to open it in A&D graphical tools:
```shell
regmap.py --dump /tmp/output.json 0 0x48
```

### Register map `diff`

It is possible to use the `regmap-diff.py` tool
to differentiate (bitwise) an official A&D registermap (created with their GUI)
and a dumped one (`--dumped` with regmap.py).

```shell
# order is always: 
#  1) official (from A&D GUi) 
#  2) then dumped file
regmap-diff.py official_ad.json /tmp/output.json
```

This script is mainly used for debugging purposes.

It is equivalent to a `diff -q -Z official_ad.json /tmp/output.json`
focused on the "RegisterMap" field.
That command being impossible to use, because --dump
does not replicate 100% of the official A&D file content (too complex),
and is not focused on the "RegisterMap" field.

## Status script

`status.py` is a read only tool, to interact with the integrated chip.  
`i2c` bus number (integer number) and slave address (hex) must be specified.

Use the `help` menu to learn how to use this script:
```shell
status.py -h
```

Output format is `json` and is streamed to `stdout`.
Each `--flag` can be cumulated and increase the size of the status report:

```shell
# Grab general / high level info (bus=0, 0x4A):
status.py --info --serial --pll 0 0x4A

# General clock infos + ref-input status (bus=1, 0x48):
status.py --info --pll --sysclk --ref-input 1 0x48

# IRQ status register
status.py --irq 0 0x4A

# dump status to a file
status.py --info --serial --pll 0 0x4A > /tmp/status.json
```

### Status report filtering

Status report can be quite verbose.
It is possible to filter the status report by field indentifiers
and field values. Filters of the same kind, and different kinds
can be cummulated, meaning, it is possible to cummulate
several identifiers filter and value filters.

* `--filter-by-key`: filters result by keyword identifiers.
Identifiers are passed as comma separated strings.
Filters are cummulated (comma separated) and applied according to
descripted order. Identifiers must match exactly (case sensitive).

```shell
# Clock infos filter
status.py --info --filter-by-key vendor 0 0x48
```

It is possible to cummulate filters using comma separated description

```shell
# Retain two infos
status.py --info --filter-by-key chip-type,vendor 0 0x48

# Clock distribution status: focus on channel 0
status.py --distrib --filter-by-key ch0 0 0x48

# Retain only `a` and `b` paths among channel0
status.py --distrib --filter-by-key ch0,a,b 0 0x48
```

By default, if requested keyword is not found,
filter op is considered faulty and fulldata set is exposed.

```shell
# trying to filter general infos
status.py --info --filter-by-key something 0 0x48
```

It is possible to filter several status reports with
relevant keywords:

```shell
# Request clock distribution status report
# and ref-input status report
# -> restrict distribution status report to `ch0`  
# -> restrict pll status to `ch0`
# --> --ref-input status is untouched because it does not contain the `ch0` identifier
status.py --pll --distrib --ref-input filter-by-key ch0 0 0x48

# Same thing idea, but we apply a filter on seperate status reports
# `ch1` only applies to `distrib`, `slow` only applies to `ref-input`
status.py --distrib --ref-input --filter-by-key ch1,slow 0 0x48
```

* `filter-by-value`: it is possible to filter status reports
by matching values. Once again, only exactly matching keywords
are retained.

```shell
# Return `0x456` <=> vendor field
status.py 1 0x48 \
    --info \
    --filter-by-value 0x456

# Return only deasserted values
status.py 1 0x48 \
    --distrib \
    --filter-by-value disabled 

# Event better `deasserted` value filter
status.py 1 0x48 \
    --distrib \
    --filter-by-value disabled,false,inactive
```

It is possible to combine `key` and `value` filters:

```shell
# from CH0 return only deasserted values
status.py 1 0x48 \
    --distrib \
    --filter-by-value ch0 \
    --filter-by-value disabled,false,inactive
```

### Extract raw data from status report

By default everything is encapsulated in json.   
It is possible to use the `--unpack` option,
to either:

* unpack the data of interest as raw data.
This works if we applied enough filter to zoom in on a single/unique field

```shell
status.py 0 0x4A \
    --info --filter-by-key vendor # extract vendor info \
    --unpack # raw value
status.py 0 0x4A \
    --misc --filter-by-key temperature,value # extract t° reading \
    --unpack # raw value
# extract temperature alarm bit
status.py 0 0x4A \`
    --misc --filter-by-key temperature,alarm # extract t° alarm bit \
    --unpack # raw value
```

Such scenario (meaningful filters) is handy when trying to interprate 
raw data into another script. Here's an example on how to do this in python once again:

```shell
# call status.py from another python script;
import subprocess
args = ['
    status.py', 
    '--misc', '0', '0x4A',
    '--filter-by-key', 'temperature,alarm', # --> efficient filter
    '--unpack', # bool() direct cast 
]
# interprate filtered stdout content directly
ret = subprocess.run(args)
if ret.exitcode == 0: # OK
    # direct cast
    has_alarm = bool(ret.stdout.decode('utf-8')) 
```

* reduce the data structure to 1D. 
"Simplifies" the output structure, but we lose data
if two identical identifiers still coexist

```shell
status.py 0 0x4A \
    --misc --filter-by-key temperature,value # extract t° reading \
    --unpack # raw value
```

## Sys clock

`Sys` clock compensation is a new feature introduced in AD9546.
`sysclock.py` allows quick and easy access to these features.

To determine current `sysclock` related settings, use status.py with `--sysclock` option.

* `--freq`: to program input frequency [Hz]
* `--sel` : to select the input path (internal crystal or external XOA/B pins)
* `--div`: set integer division ratio on input frequency
* `--doubler`: enables input frequency doubler

## Calibration script

`calib.py` allows chipset (re)calibration.   

It is required to perform a calibration at boot time.  
It is required to perform an analog Pll (re)calibration anytime
we recover from a sys clock power down.

* Perform complete (re)calibration

```shell
calib.py --all 0 0x4A
```

* Perform only a sys clock (re)calibration
(1st step in application note)

```shell
calib.py --sysclk 0 0x4A
```

Monitor internal calibration process with

```shell
status.py 1 0x4A \
    -pll --sysclk --filter-by-key calibrating
status.py 1 0x4A \
    --sysclk --irq --filter-by-key calibration 
```

## Clock distribution

`distrib.py` is an important utility.   
It helps configure the clock path, control output signals
and their behavior.  

To determine the chipset current configuration related to clock distribution,
one should use the status script with `--distrib` option.

Control flags:

* `--channel` (optionnal) describes which targetted channel.
Defaults to `all`, meaning if `--channel` is not specified, both channels (CH0/CH1)
are assigned the same value.
This script only suppports a single `--channel` assignment.

* `--path` (optionnal) describes desired signal path. 
Defaults to `all` meaning, all paths are assigned the same value (if feasible).  
This script only suppports a single `--path` assignment at a time.  
Refer to help menu for list of accepted values.

* `--pin` (optionnal) describes desired pin, when controlling an output pin.
Defaults to `all` meaning, all pins (+ and -) are assigned the same value when feasible.  
Refer to help menu for list of accepted values.

Action flags: the script supports as many `action` flags as desired, see the list down below.

* `--mode` set OUTxy output pin as single ended or differential
* `--format` sets OUTxy current sink/source format
* `--current` sets OUTxy pin output current [mA], where x = channel

```shell
# set channel 0 as HCSL default format
distrib.py --format hcsl --channel 0

# set channel 1 as CML format
distrib.py --format hcsl --channel 1

# set channel 0+1 as HCSL default format
distrib.py --format hcsl

# set Q0A, Q0B as differntial output
distrib.py --mode diff --channel 0

# set Q1A, as single ended pin
distrib.py --mode se --channel 1 --pin a

# set Q0A Q0B to output 12.5 mA, default output current
distrib.py --current 12.5 --channel 0

# set Q1A to output 7.5 mA, minimal current
distrib.py --current 7.5 --channel 1 --pin a
```

* `--sync-all`: sends a SYNC order to all distribution dividers.
It is required to run a `sync-all` in case the current output behavior
is not set to `immediate`.

```shell
# send a SYNC all
# SYNC all is required depending on previous actions and current configuration
distrib.py --sync-all 0 0x48
```

* `--autosync` : control given channel so called "autosync" behavior.

```shell
# set both Pll CH0 & CH1 to "immediate" behavior
distrib.py --autosync immediate 0 0x48

# set both Pll CH0 to "immediate" behavior
distrib.py --autosync immediate --channel 0 0 0x48

#  and Pll CH1 to "manual" behavior
distrib.py --autosync manual --channel 1 0 0x48
```

In the previous example, CH1 is set to manual behavior.  
One must either perform a `sync-all` operation,
a `q-sync` operation on channel 1,
or an Mx-pin operation with dedicated script, to enable this output signal.

* `--q-sync` : initializes a Qxy Divider synchronization sequence manually. 
When x is the `channel` and `y` is desired path.
```shell
# triggers Q0A Q0B Q1A Q1B SYNC 
distrib.py --q-sync 0 0x48

# triggers Q0A Q0B SYNC 
distrib.py --q-sync --channel 0 0 0x48

# triggers Q0B Q1B SYNC because --channel `all` is implied 
distrib.py --q-sync --path b 0 0x48
```

* `--unmute` : controls QXY unmuting opmode,
where x is the `channel` and `y` desired path.

```shell
# Q0A Q0B + Q1A Q1B `immediate` unmuting 
distrib.py --unmute immediate 0 0x48

# Q0A Q1A `phase locked` unmuting 
distrib.py --unmute phase --path a 0 0x48

# Q0B Q1B `freq locked` unmuting 
distrib.py --unmute freq --path b 0 0x48

# Q0A + Q1B `immediate` unmuting 
distrib.py --unmute immediate --path a 0 0x48
distrib.py --unmute immediate --path b 0 0x48
```

* `--pwm-enable` and `--pwm-disable`: constrols PWM modulator
for OUTxy where x is the `channel` and `y` the desired path.

* `--divider` : control integer division ratio at Qxy stage

```shell
# Sets R=48 division ratio, 
# for Q0A,AA,B,BB,C,CC and Q1A,AA,B,BB 
# because --channel=`all` and --path=`all` is implied
distrib.py --divider 48 0 0x48

# Sets Q1A,AA,B,BB R=64 division ratio
# because --path=`all` is implied
distrib.py --divider 64 --channel 1 0 0x48

# Q0A & Q0B R=23 division ratio
# requires dual assignment, because --pin {a,b} is not feasible at once
distrib.py --divider 23 --channel 0 --pin a 0 0x48
distrib.py --divider 23 --channel 0 --pin b 0 0x48
```

* `--half-divider` : enables "half divider" feature @ QXY path

* `--phase-offset` applies instantaneous phase offset to desired
output path. Maximal value is 2\*D-1 where D is previous `--divider` ratio
for given channel + pin.

```shell
# Apply Q0A,AA,B,BB,C,CC + Q1A,AA,B,BB 
# TODO
```

* `--unmuting` : controls "unmuting" behavior, meaning,
output signal can be exposed automatically depending on clock state.

* `--mute` and `--unmute` to manually enable/disable an output pin


## Reset script

To quickly reset the device

* `--soft` : performs a soft reset
* `--sans` : same thing but maintains current registers value 
* `--watchdog` : resets internal watchdog timer
* `-h` for more infos

```shell
# Resets (factory default)
reset.py --soft 1 0x48
regmap.py --load settings.json 1 0x48 
reset.py --sans 1 0x48 # settings are maintained
```

## Ref input script

`ref-input.py` to control the reference input signal,
signal quality constraints, switching mechanisms 
and the general clock state.

* `--freq` set REFxy input frequency [Hz]
* `--coupling` control REFx input coupling
`lock` must be previously acquired.

* `freq-lock-thresh` : frequency locking mechanism constraint.
* `phase-lock-thresh` : phase locking mechanism constraint.
* `phase-step-thresh` : inst. phase step threshold 
* `phase-skew`: phase skew

## PLL script

`pll.py` to control both analog and digital internal PLL cores.  
`pll.py` also allows to set the clock to free run or holdover state.

* `--type`: to specify whether we are targetting an Analog PLL (APLLx) 
or a Digital PLL (APLLx). This field is only required
for operations where it is ambiguous (can be performed on both cores).   
`--type all` : performs desired operation on both APLLx and DPLLx cores.

* `--channel` : set `x` in DPLLx or APLLx targeted cores.   
`--channel all`: is the default behavior, targets both channel 0 and 1 
of the desired type.

* `--free-run`: forces clock to free run state, `--type` is disregarded 
because `digital` is implied. 
* `--holdover`: forces clock to holdover state, `--type` is disregarded 
because `digital` is implied. 

It is easier to always request a `free-run`, in the sense this
request cannot fail.


## Power down script

`power-down.py` perform and recover power down operations.   
Useful to power down non needed channels and internal cores. 

The `--all` flag addresses all internal cores.  
Otherwise, select internal units with related flag

* Power down device entirely
```shell
power-down.py 0 0x4A --all
```
* Recover a complete power down operation
```shell
power-down.py 0 0x4A --all --clear
```


## Power down script

`power-down.py` perform and recover power down operations.   
Useful to power down non needed channels and internal cores. 

The `--all` flag addresses all internal cores.  
Otherwise, select internal units with related flag

* Power down device entirely
```shell
power-down.py 0 0x4A --all
```
* Recover a complete power down operation
```shell
power-down.py 0 0x4A --all --clear
```

* Wake `A` reference up and put `AA,B,BB` references to sleep:
```shell
power-down.py 0 0x4A --refb --refbb --refaa 
power-down.py 0 0x4A --clear --refa 
```

## CCDPLL : Digitized Clocking Common Clock Synchronizer

CCDPLL status report:

```shell
status.py --ccdpll 1 0x48
```

CCDPLL must be configured and monitored for UTS & IUTS related operations

## User Time Stamping cores

UTS cores allow the user to timestamp input data against
a reference signal. UTS requires the CCDPPL that is
part of the Digitized clocking core to be configured.

UTS core status reports and current readings:

```shell
status.py --uts 1 0x48

# UTS0 + UTS Fifo status report
status.py --uts 1 0x048 --filter-by-key fifo,0 
```

It is useful to combine this status report to the digitized
clocking status report.

UTS Readings are either signed 24 bit or signed 48 bit values,
this python script should scale and interprate those value
correctly (double check that).

It is not clear at the moment which UTSx core is fed
to the UTS FIFO therefore which scaling should be used
to interprate the UTS FIFO Reading. At the moment,
is is hardcoded to Core #0 (1st one).

## Inverse UTS

Inverse UTS allows the user to synthesize signals 
based on a local reference and a given timestamp related
to that reference.

## IRQ events

`status.py --irq` allows reading the current asserted IRQ flags.  

Clear them with `irq.py`:

* `--all`: clear all flags
* `--pll`: clear all PLL (PLL0 + PLL1 + digital + analog) related events 
* `--pll0`: clear PLL0 (digital + analog) related events 
* `--pll1`: clear PLL1 (digital + analog) related events 
* `--other`: clear events that are not related to the pll subgroup
* `--sysclk`: clear all sysclock related events 
* `-h`: for other known flags

## Misc

`status.py --misc` returns (amongst other infos) the internal temperature sensor reading.  

* Get current reading :
```shell
status.py --misc 1 0x48
# Filter on field of interest like this
status.py --misc 1 0x48 --filter-by-key temperature,value --unpack
# Is temperature range currently exceeded
status.py --misc 1 0x48 --filter-by-key temperature,alarm --unpack
```

* Program a temperature range :

```shell
misc.py --temp-thres-low -10 # [°C]
misc.py --temp-thres-high 80 # [°C]
misc.py --temp-thres-low -30 --temp-thres-high 90
status.py --temp 0 0x48 # current reading [°C] 
```

Related warning events are then retrieved with the `irq.py` utility, refer to related section.

## Typical configuration flows

* load a profile preset, calibrate and get started

```shell
regmap.py --load profile.json --quiet 0 0x48
status.py --pll --distrib --filter-by-key ch0 0 0x48
calib.py --all 0 0x48
status.py --pll --distrib --filter-by-key ch0 0 0x48
```

* distrib operation: mute / unmute + powerdown (TODO)

* using integrated signal quality monitoring (TODO)
