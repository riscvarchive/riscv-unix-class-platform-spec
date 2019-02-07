
## Introduction:
This is a proposal to make SBI a flexible and extensible interface.
It is based on the foundation principles of RISC-V and aims to be modular,
extensible and open. This proposal only discusses SBI base specification which
defines mandatory functionalities required by all implementations. Additional
features such as CPU power management or vendor specific functionalities are not
described in this document. All extensions to this base SBI specifications will
be described in separate SBI extension specification.

The design of SBI base features presented in this document is inspired by the
Power State Coordination Interface (PSCI)[1] widely used for ARM based platforms.
However, it adds only a few new mandatory SBI calls providing
version information and supported functions, unlike PSCI where a significant
number of functions are mandatory. The next two section defines the most important
aspects of this specification i.e. a versioning and function id scheme. All the
future extensions will be be based on this scheme.

## SBI Versioning scheme
SBI specification version is a 32bits long unsigned integer.

```
31                    16                   0
--------------------------------------------
Major Version         |  Minor Version     |
```

The existing SBI version will be at version **v0.1**. It will called legacy version
from here onwards in this document. The version of this proposed specification
will be **v0.2**. A different major version may indicate possible incompatible
functions. A different minor version must be compatible with each other even if
they have a higher number of features.

## SBI Function ID numbering scheme:
SBI function ID is a combination of function set number and function type number.
A function type is an individual function and a function set is a group of SBI
functions types which collectively implement similar kind of feature/functionality.
The function ID numbering scheme need to be scalable and validation check should be
as quick as possible.

SBI Function ID is a 32bits long unsigned integer.

```
31           20          16                 0
---------------------------------------------
Function Set |  Reserved |   Function Type  |

```
Bit[31:16] =  Function Set

Bit[12:15] =  Reserved for future use

Bit[11:0]  =  Function Type within Function Set

Both function set and function type are unsigned integers. Thus, there can be 2^16
function types within a function set. The following table describes the generic
function set numbers and reserved number ranges for future use.

| Function Set | Value         | Description                                   |
| ------------ | ------------- | --------------------------------------------- |
| Legacy       | 0x000         | Existing SBI Functions. Not mandatory.        |
| Base         | 0x001         | Base Functions mandatory for any SBI version. |
| HART PM      | 0x002         | Hart power management.                        |
| System PM    | 0x003         | System-level power management                 |
| Reserved     | 0x010-0x7ff   | Reserved for experimental extensions          |
| Vendor       | 0x800-0xfff   | Vendor specific Functions                     |

Anybody who intend to add new SBI functionalities and unsure of the any existing
function set it belongs to, may choose any function set ID from *Reserved* region.
In case of any conflict with other SBI function set, RISC-V foundation reserves
the right to resolve the conflict by assigning different IDs to both parties.
Please keep in mind that, *Reserved* region should only be used for SBI functionalities
that may become standard in future. Any vendor specific functionalities should
fall under *Vendor* function set.

## Statement of Principles
1. Every new SBI version with minor version upgrade must be be backward compatible
   with previous versions except *Legacy* version.
2. All the SBI function set except the *Base* are optional.
3. A new function set/type should be added only if it is absolutely required.
4. Every new SBI function set/type need to be approved by the *RISC-V Unix Platform
   Specification* working group before it can be added to the specification.
5. SBI implementation should not include any self-modifying code.
6. SBI implementation should always be possible for a supervisor mode software to
   replace any SBI function with its own implementation.

## SBI Calling Conventions
Each SBI call adheres to a specific calling convention that defines how data is
provided as input or output. The *ecall* instruction is used to make a request to
the Supervisor Execution Environment(SEE). When executed in S-mode, it generates
an environment-call-from-S-mode exception and performs no other operation. The
required arguments are passed through registers a0-a2 and the SBI call type is
passed via register a7. Once the machine mode receives the trap, it identifies
the type of SBI call from a7 register and performs the specified operation.

Every SBI function call may return the following structure.
```
struct sbiret {
  long value;
  long error;
};
```
Both a0 & a1 registers can be used for function return values [3]. The first
member of sbiret "value" contains the return value or error from the SBI function.
The second member of sbiret "error" is optional and can be used to indicate any
other errors that SBI implementation may want to return.

Any unsupported SBI call should return SBI_ERR_NOT_SUPPORTED.

## SBI Functions:
SBI function is an individual feature that the SBI interface provides to supervisor
mode from machine mode. Each function ID is assigned as per below section.

### SBI Function List in both SBI v0.2 and v0.1

|Function Type              | Function Set      | ID(v0.2)     |ID (v0.1)  |
|---------------------------| ------------------|:------------:|:---------:|
| sbi_set_timer             | Legacy            | 0x0000 0000  | 0         |
| sbi_console_putchar       | Legacy            | 0x0000 0001  | 1         |
| sbi_console_getchar       | Legacy            | 0x0000 0002  | 2         |
| sbi_clear_ipi             | Legacy            | 0x0000 0003  | 3         |
| sbi_send_ipi              | Legacy            | 0x0000 0004  | 4         |
| sbi_remote_fence_i        | Legacy            | 0x0000 0005  | 5         |
| sbi_remote_sfence_vma     | Legacy            | 0x0000 0006  | 6         |
| sbi_remote_sfence_vma_asid| Legacy            | 0x0000 0007  | 7         |
| sbi_shutdown              | Legacy            | 0x0000 0008  | 8         |
|---------------------------|-------------------|--------------|-----------|
| sbi_get_spec_version      | Base              | 0x0010 0001  | -         |
| sbi_set_sbiimp_version    | Base              | 0x0010 0002  | -
| sbi_is_function_set       | Base              | 0x0010 0003  | -         |
| sbi_is_function_type      | Base              | 0x0010 0003  | -         |
| sbi_get_vendor_id         | Base              | 0x0010 0004  | -         |
| sbi_get_mimp_id           | Base              | 0x0010 0005  | -         |
| sbi_get_sbiimp_id         | Base              | 0x0010 0006  | -         |
|---------------------------|-------------------|--------------|-----------|
| sbi_set_timer             | Exp-1             | 0x0100 0000  | -         |
| sbi_console_putchar       | Exp-1             | 0x0100 0001  | -         |
| sbi_console_getchar       | Exp-1             | 0x0100 0002  | -         |
| sbi_clear_ipi             | Exp-1             | 0x0100 0003  | -         |
| sbi_send_ipi              | Exp-1             | 0x0100 0004  | -         |
| sbi_remote_fence_i        | Exp-1             | 0x0100 0005  | -         |
| sbi_remote_sfence_vma     | Exp-1             | 0x0100 0006  | -         |
| sbi_remote_sfence_vma_asid| Exp-1             | 0x0100 0007  | -         |

There are duplicate function types between *Legacy* and Exp-1 because some of them
may have a new function signature which makes it a different function type from
the legacy one. Moreover, some of them may fall under their own function type in
future or get added to the *Base* if they still considered to be mandatory for any
SBI version.

### Function Description

This section describes every newly introduced(in v0.2) function in details.
Please refer to [2] for any legacy(v0.1) functions.

```
struct sbiret sbi_get_version(void):
```
Returns the current SBI specification version implemented by the firmware.

```
struct sbiret sbi_set_sbiimp_version(void):
```
Returns the current version of the SBI implementation. This will be provided by the
software implementing SBI specification.

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
Returns the vendor ID as per JEDEC manufacturer IDs[4] i.e. data stored in mvendorid
CSR. This should be used to verify any vendor function types.

```
struct sbiret sbi_get_mimp_id(void):
```
Returns the machine implementation ID i.e. data stored in mimpid CSR.

```
struct sbiret sbi_get_sbiimp_id(void):
```
Returns the SBI implementation ID. The process to assign these IDs is not decided
yet.

## Return error code Table:
Here are the SBI return error codes defined.

| Error Type               | Value  |
| -------------------------|:------:|
|  SBI_SUCCESS             |  0     |
|  SBI_ERR_FAILURE         | -1     |
|  SBI_ERR_NOT_SUPPORTED   | -2     |
|  SBI_ERR_INVALID_PARAM   | -3     |
|  SBI_ERR_DENIED          | -4     |
|  SBI_ERR_INVALID_ADDRESS | -5     |


## Reference:

[1] http://infocenter.arm.com/help/topic/com.arm.doc.den0022d/Power_State_Coordination_Interface_PDD_v1_1_DEN0022D.pdf

[2] https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.md

[3] https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md

[4] https://riscv.org/specifications/privileged-isa/
