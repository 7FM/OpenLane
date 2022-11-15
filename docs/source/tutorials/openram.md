# Designing a chip with an OpenRAM (sky130)

## Overview

This guide covers the RTL-to-GDS flow using [OpenRAM](https://openram.org/) cells and many macro-related features from the OpenLane flow for full chip integration.

## Create a new design

Create a new design using the following command:

```
./flow.tcl -design test_sram_macro -init_design_config -add_to_designs
```

## Create the Verilog files

Create or add Verilog files. In this case create file `designs/test_sram_macro/src/test_sram_macro.v` with following content:

```{literalinclude} ../../../designs/test_sram_macro/src/test_sram_macro.v
:language: verilog
```

## Connect the layout files and abstracts

LEF files are abstract representations of hard macroblocks, such as OpenRAM macros.
These files contain a lightweight abstract representation of the cell.
LEF contains only metal layers and layers that can connect between cells (`met1`, `via2`, `nwell`, `pwell`, etc).

OpenLane configuration of LEF files is done using `EXTRA_LEFS`.
In this case the absolute path is used, if the PDK location is different then the path needs to be changed.

Next, configure the GDS files of the hard macro. OpenLane configuration of GDS is `EXTRA_GDS_FILES`.

:::{warning}
It is the user's responsibility to make sure that GDS matches LEF files.
:::

```json
"EXTRA_LEFS":      "/openlane/pdks/sky130B/libs.ref/sky130_sram_macros/lef/sky130_sram_1kbyte_1rw1r_32x256_8.lef",
"EXTRA_GDS_FILES": "/openlane/pdks/sky130B/libs.ref/sky130_sram_macros/gds/sky130_sram_1kbyte_1rw1r_32x256_8.gds",
```

:::{warning}
If you run the design without this configuration you will get the following error:

```
[INFO]: Running Initial Floorplanning (log: designs/test_sram_macro/runs/full_guide_nomacros/logs/floorplan/3-initial_fp.log)...
[ERROR]: Floorplanning failed
[ERROR]: module sky130_sram_1kbyte_1rw1r_32x256_8 not found in /openlane/designs/test_sram_macro/runs/full_guide_nomacros/tmp/merged.nom.lef
[ERROR]: Check whether EXTRA_LEFS is set appropriately
```
:::

## Connect the blackbox information and timing data

:::{note}
[Multiple scenario Liberty files are not supported yet](https://github.com/The-OpenROAD-Project/OpenLane/issues/1343).
:::

Liberty flow contains the timings and description of I/O of the macro.
It is used in Timing analysis and in synthesis.
The liberty file is supplied to flow using `EXTRA_LIBS` configuration. Add it to the configuration file:

```
"EXTRA_LIBS":      "/openlane/pdks/sky130B/libs.ref/sky130_sram_macros/lib/sky130_sram_1kbyte_1rw1r_32x256_8_TT_1p8V_25C.lib",
```

:::{warning}
If you skip this configuration you will get the following error:

```
ERROR: Module `\sky130_sram_1kbyte_1rw1r_32x256_8' referenced in module `\test_sram_macro_unwrapped' in cell `\sram1' is not part of the design.
child process exited abnormally
```
:::

## Power/Ground nets

Create the power/ground nets.
The first net in the list will be used for standard cell power connections.

```json
"VDD_NETS": "vccd1",
"GND_NETS": "vssd1",
```

If you need more power/ground nets add the nets to the list:

```json
"VDD_NETS": "vccd1 vccd2",
"GND_NETS": "vssd1 vssd2",
```

## Power/Ground PDN connections

Add the PDN connections between SRAM cells and the power/ground nets.

Syntax: `<instance_name> <vdd_net> <gnd_net> <vdd_pin> <gnd_pin>`.

More information is available in [configuration variables documentation](../reference/configuration.md).
Each macro hook is separated using a comma, for example:

```json
"FP_PDN_MACRO_HOOKS": "submodule.sram0 vccd1 vssd1 vccd1 vssd1, submodule.sram1 vccd1 vssd1 vccd1 vssd1",
```

The instance names need to be fetched from the synthesis netlist.
For this purpose run the design until the synthesis stage using the following command:

```
./flow.tcl -design test_sram_macro -tag synthesis_only -to synthesis -overwrite
```

Open the following file `designs/test_sram_macro/runs/synthesis_only/results/synthesis/test_sram_macro.v`.

```verilog
/* Generated by Yosys 0.12+45 (git sha1 UNKNOWN, gcc 8.3.1 -fPIC -Os) */

module test_sram_macro(rst_n, clk, cs, we, addr, write_allow, datain, dataout);
wire _000_;
wire _001_;
wire _002_;
...
sky130_sram_1kbyte_1rw1r_32x256_8 \submodule.sram0  (
    .addr0(addr),
    ...
    .wmask0(write_allow[3:0])
);
sky130_sram_1kbyte_1rw1r_32x256_8 \submodule.sram1  (
    .addr0(addr),
    ...
    .wmask0(write_allow[7:4])
);
```

:::{warning}
Note that there are two cells `sky130_sram_1kbyte_1rw1r_32x256_8` with instance names `\submodule.sram0`, `\submodule.sram1`.
Both contain backslashes according to Verilog syntax.
In `FP_PDN_MACRO_HOOKS` you need to use the name without the escape slash. Therefore it turns into `submodule.sram0`, `submodule.sram1`.
:::

```json
"FP_PDN_MACRO_HOOKS": "submodule.sram0 vccd1 vssd1 vccd1 vssd1, submodule.sram1 vccd1 vssd1 vccd1 vssd1",
```

`FP_PDN_MACRO_HOOKS` forces connection between these pins and power/ground nets.
If these configuration is missing then power/ground will not be connected.
Try removing the parameter and running:

```
./flow.tcl -design test_sram_macro -tag full_guide_pdn_macrohooks -overwrite
```

Open an interactive session:

```
./flow.tcl -design test_sram_macro -tag full_guide_pdn_macrohooks -interactive
package require openlane
set_def designs/test_sram_macro/runs/full_guide_pdn_macrohooks/results/final/def/test_sram_macro.def
or_gui

# empty new line to force the command to run
```

Notice that the PDN straps are not connected to ring of SRAM:

:::{figure} ../../_static/openram/pdn_macro_hooks_missing.png
:::

## Floorplanning

Run the flow until the floorplanning stage:

```
./flow.tcl -design test_sram_macro -tag floorplan -overwrite -to floorplan
```

You will get the following output:

```
[STEP 3]
[INFO]: Running Initial Floorplanning (log: designs/test_sram_macro/runs/floorplan/logs/floorplan/3-initial_fp.log)...
[INFO]: Extracting core dimensions...
[INFO]: Set CORE_WIDTH to 877.22, CORE_HEIGHT to 875.84.
```

To view the output of the floorplan stage, run the following command:

```
./flow.tcl -design test_sram_macro -tag floorplan -interactive
package require openlane
set_def designs/test_sram_macro/runs/floorplan/results/floorplan/test_sram_macro.def
or_gui

# empty new line to force the command to run
```

It will look like this:

:::{figure} ../../_static/openram/basic_fp.png
:::

Looking at the floorplan, it would be better if the macros were centered, so the buffers can be placed near I/O.
In order to achieve this, keep the area almost the same,
but resize the `DIE_AREA` to a rectangle that allows `100um` all around each macro for standard cells.
In the next step, the location of macroblocks will be selected.

Set the following floorplan parameters:

```json
"FP_SIZING": "absolute",
"DIE_AREA": "0 0 750 1250",
"PL_TARGET_DENSITY": 0.5,
```

`FP_SIZING` is set to `absolute` and it will tell the floorplan to use `DIE_AREA` as macroblock's size.
Next, set the `DIE_AREA`. This value is carefully constructed.

:::{warning}
If `DIE_AREA` is set a to big value then you are going to have routing/placement/timing issues.
On the other hand, setting the value too low will cause placement and routing congestion issues.
:::

`PL_TARGET_DENSITY` is set to 0.5 to reflect the target final density of 50%.

I/O placement can be defined using the process described in the [Hardening Macros guide](../usage/hardening_macros.md).

## Macrocell placement

In this section, we will use manual macro placement to do the placement.
Automatic macro placement does not account for I/O pin placement causing the SRAMs to be placed at the edge of the DIE
which results in sub-optimal I/O power usage due to long nets.

The size of the cells can be taken from the LEF file `pdks/sky130B/libs.ref/sky130_sram_macros/lef/sky130_sram_1kbyte_1rw1r_32x256_8.lef`.
While it is not required to know the size of the cell,
it is useful for the purpose of making sure that the subcomponents do not overlap.

For example:

```
UNITS
DATABASE MICRONS 1000 ;
END UNITS
MACRO sky130_sram_1kbyte_1rw1r_32x256_8
CLASS BLOCK ;
SIZE 479.78 BY 397.5 ;
SYMMETRY X Y R90 ;
```

To specify the cell placement create the file `designs/test_sram_macro/macro_placement.cfg`:

```{literalinclude} ../../../designs/test_sram_macro/macro_placement.cfg
```

The syntax is `<instance name> <x> <y> <direction>`. Each new line contains configuration for one macro block

The instance name needs to be taken directly from the synthesis netlist without an escape symbol at the beginning.

Next, modify the `config.json` to reference this file.

```json
"MACRO_PLACEMENT_CFG": "dir::macro_placement.cfg",
```

```
./flow.tcl -design test_sram_macro -tag floorplan_v2 -overwrite -to floorplan
```

To view the output of the floorplan stage, run the following command:

```
./flow.tcl -design test_sram_macro -tag floorplan_v2 -interactive
package require openlane
set_def designs/test_sram_macro/runs/floorplan_v2/results/floorplan/test_sram_macro.def
or_gui

# empty new line to force the command to run
```

It will look like this:

:::{figure} ../../_static/openram/floorplan_v2.png
:::

## Common problems to avoid

### DRCs inside SRAM macros

:::{warning}
SRAM cells in sky130 have a special set of DRC rules.
OpenRAM uses these optimized SRAM cells but the current DRC deck is missing these rules, causing false issues.

Currently, use the workaround presented below.
:::

The sky130 uses optical proximity to reduce the size of the SRAM transistors.
The SRAM blocks in sky130 generated by OpenRAM use different DRC ruleset to accommodate for this size reduction.
Therefore when running the Magic VLSI it is expected to have many DRC violations.

The `MAGIC_DRC_USE_GDS` can be set to false, forcing the Magic VLSI to run DRC on DEF/LEF instead of GDS.
However, you will still get `This layer can't abut or partially overlap between subcells` DRCs.
These DRCs are not critical.

```json
"MAGIC_DRC_USE_GDS": false
```

If you open `designs/test_sram_macro/runs/full_guide/reports/signoff/drc.rpt`
you can see following error: `This layer can't abut or partially overlap between subcells`.

This error is caused by the issue in Open_PDKs. It is a warning, therefore it can be ignored.

For this example, we can just disable the DRC stopping the flow.
After the run is done, check the DRC report manually and make sure the issues are not critical.

```json
"QUIT_ON_MAGIC_DRC": false
```

:::{warning}
If you do not manually check the DRC,
then in the submission process to the foundry DRC errors will be checked anyway,
but the turnaround is too long to rely on this.
:::

### DRC because of PDN being too close to the met4 inside SRAM

The selected placement can cause DRC.
In this guide, the selected location of Y=700 causes met4 of the SRAM to be too close to power straps.
To mitigate this, the SRAM instance was moved down.

### Setup violations

During the run it was clear that the clock period of `10.0` was too low:

```
[WARNING]: There are max slew violations in the design at the typical corner. Please refer to 'designs/test_sram_macro/runs/full_guide_libs_3/reports/signoff/32-rcx_sta.slew.rpt'.
[WARNING]: There are max fanout violations in the design at the typical corner. Please refer to 'designs/test_sram_macro/runs/full_guide_libs_3/reports/signoff/32-rcx_sta.slew.rpt'.
[WARNING]: There are max capacitance violations in the design at the typical corner. Please refer to 'designs/test_sram_macro/runs/full_guide_libs_3/reports/signoff/32-rcx_sta.slew.rpt'.
[INFO]: There are no hold violations in the design at the typical corner.
[ERROR]: There are setup violations in the design at the typical corner. Please refer to 'designs/test_sram_macro/runs/full_guide_libs_3/reports/signoff/32-rcx_sta.max.rpt'.
```

As a solution the period was increased to `25.0`:

```
"CLOCK_PERIOD": 25.0,
```

### JSON syntax error regarding the comma

The last field of the object in JSON must not have any commas, otherwise, you will have a syntax issue:

```
[INFO]: Using configuration in 'designs/test_sram_macro/config.json'...
[ERROR]: Traceback (most recent call last):
File "/openlane/scripts/config/to_tcl.py", line 351, in <module>
    cli()
File "/usr/local/lib/python3.6/site-packages/click/core.py", line 1128, in __call__
    return self.main(*args, **kwargs)
File "/usr/local/lib/python3.6/site-packages/click/core.py", line 1053, in main
    rv = self.invoke(ctx)
File "/usr/local/lib/python3.6/site-packages/click/core.py", line 1659, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
File "/usr/local/lib/python3.6/site-packages/click/core.py", line 1395, in invoke
    return ctx.invoke(self.callback, **ctx.params)
File "/usr/local/lib/python3.6/site-packages/click/core.py", line 754, in invoke
    return __callback(*args, **kwargs)
File "/openlane/scripts/config/to_tcl.py", line 337, in config_json_to_tcl
    config_dict = json.loads(config_json_str)
File "/usr/lib64/python3.6/json/__init__.py", line 354, in loads
    return _default_decoder.decode(s)
File "/usr/lib64/python3.6/json/decoder.py", line 339, in decode
    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
File "/usr/lib64/python3.6/json/decoder.py", line 355, in raw_decode
    obj, end = self.scan_once(s, idx)
json.decoder.JSONDecodeError: Expecting property name enclosed in double quotes: line 27 column 1 (char 901)
```

The right way:

```
{
    ...
    "QUIT_ON_MAGIC_DRC": false
}
```

The wrong way:

```
{
    ...
    "QUIT_ON_MAGIC_DRC": false,
}
```

### Optional: Memory footprint

While running the flow it may use a significant amount of memory.
You can temporarily disable KLayout XOR check to reduce the memory footprint while experimenting.
But for the final GDS submission make sure that the XOR check is enabled.

```json
"RUN_KLAYOUT_XOR": false,
```

## Running the flow

Final `config.json` looks like this:

```{literalinclude} ../../../designs/test_sram_macro/config.json
:language: json
```

Finally, harden the macroblock by running the following command:

```
./flow.tcl -design test_sram_macro -tag full_guide -overwrite
```

To view the output of the floorplan stage, run the following command:

```
./flow.tcl -design test_sram_macro -tag full_guide -interactive
package require openlane
set_def designs/test_sram_macro/runs/full_guide/results/final/def/test_sram_macro.def
or_gui

# empty new line to force the command to run
```

It will look like this:

:::{figure} ../../_static/openram/final.png
:::

Reports can be found in `designs/test_sram_macro/runs/full_guide/reports`.

:::{note}
In the future, OpenDB will be used instead of DEF/LEF flow.
:::