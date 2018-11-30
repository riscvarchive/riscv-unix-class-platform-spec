# RISC-V Base SBI Working Group Charter

The RISC-V Base SBI working group aims to standardize a base SBI
specification that can be used to interface between S-mode software and
M-mode software.  This working group will define:

* A versioning scheme for the UNIX-class platform specification.
* A mechanism that allows supervisor-mode software to detect the
  version of the SBI that it has been provided.
* A mechanism that allows supervisor-mode software to detect the
  presence of SBI calls.
* An SBI extension that defines the currently-existing SBI calls.

The SBI has existed as a defacto specification for managed by the open
source software community for years, but having an official RISC-V
specification will be important in the future as we expand the RISC-V
UNIX-class platform specification to be suitable for more use cases.
Defining the base SBI in an extensible manner follows the 

I would like to suggest Palmer Dabbelt from SiFive as the chair of this
working group, and Atish Parta from Western Digital as the vice-chair.
