# FPGA-notes

My notes about specific FPGAs

## Lattice Semiconductor CrossLink-NX

[CrossLink-NX](https://www.latticesemi.com/CrossLinkNX) is a 28 nm FPGA with
MIPI DPHY capable differential I/O and MIPI DPHY hard macros.

### Eval Board

There is a [CrossLink-NX Evaluation Board](https://www.latticesemi.com/en/Products/DevelopmentBoardsAndKits/CrossLink-NXEvaluationBoard),
but be warned:

The boards currently in stock at Mouser have the -ES2 (engineering sample)
versions of LIFCL-40.  These chips require you to use Radiant Programmer
version 2.2.  Newer versions will not work!

### Programming

There are some difficulties with programming CrossLink-NX:

1. Avoid cascading CrossLink-NXs in a JTAG chain: I had no luck getting this
to work (Radiant Programmer would not discover the devices during scanning). 
But this is not a known problem to Lattice, so it's possible there is
something wrong with our custom board.

2.  I could not program devices that were already configured.  You need to
provide a falling edge on nProgram to force the FPGA to be unconfigured. 
Note that it needs to be an edge- you can not reliably do it by tying
nProgram low.  Another option, in case you have no access to nProgram, is to
prevent the FPGA from being configured by shorting MISO on the SPI-flash
device until the FPGA gives up trying to configure itself.

Older Lattice FPGAs would always show up in the scan chain, but you would
not be able to program the attached SPI-flash if the interface was used by
your FPGA design.  For those, it was enough to configure the FPGA with an
image which did not use the SPI-flash interface, and then proceed with the
SPI-flash background programming.  With Crosslink-NX, something else is
going on, maybe having to do with fuse bits.

### DDR I/O

There are DDR I/O cells, such as ODDRX1 and IDDRX1.  These cells have a
parameter called GSR, which is "ENABLED" by default.  This is bad since the
globel reset net may not be what you think it is in the chip (it's randomly
decided which reset net will use the global reset net).  Anyway, just set
GSR to "DISABLED".

### DPHY I/O

The DPHY hard macros are from a company called [MIXEL](https://mixel.com/). 
The Crosslink-NX DPHY hard macros are different from the ones used in
CrossLink.

You may use the DPHY hard macros for free (without paying for additional
Lattice IP), but it's difficult:

1. The raw hard macros can be found in IP catalog / Module / IO / MIPI_DPHY. 

2. I had difficulty when using macros from Radiant 3.1, the FPGA would not
configure.  Radiant 3.0 was OK.  Maybe something to do with delaying the
exit of configuration until PLLs are locked, or something like that.

3. The manual for these macros are here:

[FPGA-TN-02081](https://www.latticesemi.com/view_document?document_id=52781)

The manual is totally incomplete.  In addition, the signal names don't match
the IP catalog names or the MIXEL signal names.  Maybe this is due to
licensing issues with MIXEL or maybe they are encouraging you to use the
non-free Lattice IP.

On the MIPI receiver:

The generated Verilog IP wrapper does not give the divided MIPI clock.  But
it's available if you modify the wrappper: look for "int_clk": this is the
divided clock from the MIPI clock which will be running if the MIPI clock is
running.  The "byte_clk" only runs when the the MIPI data pins are in
high-speed mode.

On the MIPI transmitter:

The transmitted is always in discontinuous clock mode, no matter what the
setting says in the GUI.  I think this is a code mistake in the generated
wrapper code.

Look for .UCTXREQH(hs_tx_data_en_i) and change it to .UCTXREQH(hs_tx_en_i). 
This way, the MIPI clock will run when hs_tx_en_i is high (and not only when
you are transmitting data).

The manual is incomplete, but you figure out the signaling protocol through
simulation.  Here is a timing diagram to save you some time:

![CrossLink-NX DPHY Signals](doc/crosslinknx-dphy.png)
