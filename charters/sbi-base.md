
## Introduction:
This is a proposal to make SBI a flexible and extensible interface.
It is based on the foundation policy of RISC-V i.e. modularity and
openness. This proposal will only talk about a base version which contains bare
minimum functionalities and will be mandatory to implement. All other features
such as CPU/HART power management or vendor extensions will be defined in respective
individual SBI extension specification. This will also help to keep the open source
discussion focused as well.

The proposed design is inspired by Power State Coordination Interface (PSCI) from
ARM world. However, it adds only a few new mandatory SBI calls providing
version information and supported Functions, unlike PSCI where a significant
number of functions are mandatory. The version of the existing SBI will
be defined as a minimum version(0.1) which will always be backward compatible.
Similarly, any Linux kernel with a newer feature will fall back if an older
version of SBI does not support the updated capabilities. Both the operating
system and SEE can be implemented to be two way backward compatible.

## Statement of Principles
1. Every SBI version except the legacy should be backward compatible.
2. All the SBI function set except the Base are optional.
3. A new function set/type should be added only if it is absolutely required.
4. Every new SBI function set/type need to be approved by the RISC-V Unix Platform
   Specification working group before it can be added to the specification.
5. It should not include self-modifying code.
6. It should always be possible for a supervisor mode software to replace any
   SBI call with its own implementation.

## SBI Calling Conventions
Each SBI call adheres to a specific calling convention that defines how data is
provided as input to or read as output. Every SBI call uses ecall instructions to
make a request to the SEE environment. When executed in S-mode, it generates an
environment-call-from-S-mode exception and performs no other operation. The
required arguments are passed through registers a0-a2 and the SBI call type is
passed via register a7. Once the machine mode receives the trap, it identifies the
type of SBI call from a7 register and performs the required operation. Every SBI
function call can return the following structure.
```
struct sbiret {
  long value;
  long error;
};
```
According to the RISC-V ABI spec[4], both a0 & a1 registers can be used for function
return values. As the nomenclature suggests, the first member of sbiret "value"
contains the return value from the SBI function. The second member of sbiret "error"
is optional and can be used to indicate any global error state.

Any unsupported SBI call should return SBI_ERR_NOT_SUPPORTED.
## SBI Functions:

SBI function is an individual feature that SBI interface provides to supervisor
mode from machine mode. Each function ID is assigned as per below section.

#### SBI Function ID numbering scheme:
A Function Set is a group of SBI functions which collectively implement similar
kind of feature/functionality. The function ID numbering scheme need to be scalable
in the future. At the same time, function validation check should be as quick as possible.

SBI Function ID is u32 type.

```
31            24                        0
-----------------------------------------
               |                        |
-----------------------------------------
Function Set   | Function Type          |
```
Bit[31:24] =  Function Set

Bit[23:0]  =  Function Type within Function Set

Function Type is just an integer within the function set. Thus, there can be 2^24
function types within a function set.

Here are few Function Sets for SBI v0.2:

| Function Set         | Value    | Description                                                     |
| -------------------  |:------:  | :---------------------------------------------------------------|
| Legacy Functions     | 0x00     | Existing SBI Functions. Not mandatory.                          |
| Base Functions       | 0x00     | Base Functions mandatory for any SBI version.                   |
| HART PM Functions    | 0x01     | Hart UP/Down/Suspend Functions for per-Hart power management.   |
| System PM Functions  | 0x02     | System Shutdown/Reboot/Suspend for system-level power management|
| Reserved             | 0x4f-0x6f| Reserved for experimental extensions                            |  
| Vendor Functions     | 0xff     | Vendor specific Functions                                       |

N.B. There is a possibility that different vendors can choose to assign the same function
numbers for different functionality. That's why vendor specific strings in
Device Tree/ACPI or any other hardware description document can be used to verify
if a specific Function belongs to the intended vendor or not.

#### SBI Function List in both SBI v0.2 and v0.1

|Function Type              | Function Set      | ID(v0.2)     |ID (v0.1)  |
|---------------------------| ------------------|:------------:|:---------:|
| sbi_set_timer             | Legacy            | 0x00 000000  | 0         |
| sbi_console_putchar       | Legacy            | 0x00 000001  | 1         |
| sbi_console_getchar       | Legacy            | 0x00 000002  | 2         |
| sbi_clear_ipi             | Legacy            | 0x00 000003  | 3         |
| sbi_send_ipi              | Legacy            | 0x00 000004  | 4         |
| sbi_remote_fence_i        | Legacy            | 0x00 000005  | 5         |
| sbi_remote_sfence_vma     | Legacy            | 0x00 000006  | 6         |
| sbi_remote_sfence_vma_asid| Legacy            | 0x00 000007  | 7         |
| sbi_shutdown              | Legacy            | 0x00 000008  | 8         |
|---------------------------|-------------------|--------------|-----------|
| sbi_get_version           | Base              | 0x01 000001  | -         |
| sbi_is_function_set       | Base              | 0x01 000002  | -         |
| sbi_is_function_type      | Base              | 0x01 000003  | -         |
| sbi_get_vendor_id         | Base              | 0x01 000004  | -         |
| sbi_get_vendor_id         | Base              | 0x01 000005  | -         |
| sbi_get_mimp_id           | Base              | 0x01 000006  | -         |
| sbi_get_sbiimp_id         | Base              | 0x01 000007  | -         |
| sbi_set_timer             | Base              | 0x01 000008  | -         |
| sbi_console_putchar       | Base              | 0x01 000009  | -         |
| sbi_console_getchar       | Base              | 0x01 00000A  | -         |
| sbi_clear_ipi             | Base              | 0x01 00000B  | -         |
| sbi_send_ipi              | Base              | 0x01 00000C  | -         |
| sbi_remote_fence_i        | Base              | 0x01 00000D  | -         |
| sbi_remote_sfence_vma     | Base              | 0x01 00000E  | -         |
| sbi_remote_sfence_vma_asid| Base              | 0x01 00000F  | -         |
| sbi_shutdown              | Base              | 0x01 000010  | -         |

#### Function Description

This section describes every newly introduced(in v0.2) function in details.
Please refer to [1] for any v0.1 functions.

```
struct sbiret sbi_get_version(void):
```
Returns the current SBI version implemented by the firmware.
version: uint32: Bits[31:16] Major Version
             Bits[15:0] Minor Version

The existing SBI version can be 0.1. The proposed version will be at 0.2
A different major version may indicate possible incompatible functions.
A different minor version must be compatible with each other even if
they have a higher number of features.

```
struct sbiret sbi_is_function_set(u32 fset)
```
Given a function set, check if the function set is supported or not.

```
struct sbiret sbi_is_function_type(u32 ftype, u32 fset):
```
Given a function set and function type, check if the function type is valid under
the function set or not.

```
struct sbiret sbi_get_vendor_id(void):
```
Returns the vendor ID as per JEDEC manufacturer IDs[5] i.e. data stored in mvendorid
CSR.

```
struct sbiret sbi_get_mimp_id(void):
```
Returns the machine implementation ID i.e. data stored in mimpid CSR.

```
struct sbiret sbi_get_sbiimp_id(void):
```
Returns the SBI implementation ID. This ID is provided by the software implementing
SBI.

TODO:
1. Do we need to assign SBI implementation ID as well to avoid conflicts?
2. Do we need to reiterate the legacy SBI functions here again with updated function
signature?

## Return error code Table:
Here are the SBI return error codes defined.

| Error Type               | Value  |
| -------------------------|:------:|
|  SBI_ERR_SUCCESS         |  0     |
|  SBI_ERR_FAILURE         | -1     |
|  SBI_ERR_NOT_SUPPORTED   | -2     |
|  SBI_ERR_INVALID_PARAM   | -3     |
|  SBI_ERR_DENIED          | -4     |
|  SBI_ERR_INVALID_ADDRESS | -5     |


## Reference:

[1] http://infocenter.arm.com/help/topic/com.arm.doc.den0022d/Power_State_Coordination_Interface_PDD_v1_1_DEN0022D.pdf

[2] https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.md

[3] https://github.com/riscv/riscv-sbi-doc/pull/9

[4] https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md

[5] https://riscv.org/specifications/privileged-isa/
