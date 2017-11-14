Gas Model
=========

General considerations
----------------------

One change from EVM to IELE is that integers can be arbitrarily large.  Hence,
to be able to store and manipulate them, registers need to be arbitrarily large.
Therefore, the memory required for their storage might become non-neglectable;
for example, registers could effectively be used to encode sequential memory.

This motivates the following design decision:
* Track register sizes and treat register sizes as amounts of memory
  w.r.t. gas costs.

To keep close to the gas model of EVM, we will regard register memory as
stack memory, and allow 1024 * 32 bytes to be used free of charge for stack.

Since stack frames are freed upon returning from a call, memory costs for
registers will be computed w.r.t. the maximum size of the registers stack.

Moreover, since efficient allocation schemes (such as pooling) could be used to
represent register values, we can conceive a register as a structure containing
metadata about a value, including length, sign, and the location where that
value is stored.  If another value needs to be stored into the register, a
memory cell large enough to hold can be reserved, the existing location marked
as free, and the register structure updated accordingly.

### Operations with return register

The memory variation introduced by updating the register *r* with value *v*
```
  registerLoadDelta(r, v) = machineWordLength(v) - registerDataSize(r)
```

Another change brought upon by arbitrary length integers is that arithmetic
operations need to take into account the size of the operands.

### Expressions

For an arithmetic operation we distinguish two variations:
* the gas cost reflecting the complexity of the computation; and
* the variation of memory due to updating the result register.

* `ISZERO RREG WREG` requires just the inspection of one bit of `WREG`, so
  let's term its gas cost `bitCost`
* `NOT RREG WREG` requires all words of `WREG` to be processed, so its cost
  would be `registerSize(WREG) * wordCost`
* `ADD RREG WREG1 WREG2` requires pairwise addition of the registers' words
  so its cost would be `max(registerSize(WREG1), registerSize(WREG2)) * wordCost`
* `SUB RREG WREG1 WREG2` has the same complexity as addition
  so its cost would be `max(registerSize(WREG1), registerSize(WREG2)) * wordCost`
* `MUL RREG WREG1 WREG2` we approximate the cost to that of the Karatsuba
  algorithm, leading to an approximate cost of
  `wordCost * (3 * N ^ log(2)(3) - 2*N)`
    where `N = max(registerSize(WREG1), registerSize(WREG2)) * wordCost`
* `DIV RREX WREG1 WREG2` - if the value of `WREG2` is 0, then `bitCost`,
   otherwise, we approximate the cost to that of the basecase division method,
   that is, `wordCost * (N - M) * M`
    where `M = min(registerSize(WREG1), registerSize(WREG2))`
          `N = max(registerSize(WREG1), registerSize(WREG2))`
*

Definitions
-----------

* *machineWordSize* - size in bits of a machine word
  (e.g., 32 for 32-bit architectures, 64 for 64-bit architectures)

* *machineWordLength(v) = ceil(log(MachineWordSize)(v+1))* - the minimum number
  of machine words required to represent value *v*

* *registerMetadataSize* - size in machine words required for the structure
  holding metadata information for registers
  (e.g., size, sign, pointer to location)

* *registerSize(r)* - machine words allocated for the value of register *r*
  If *r* was not previously used, *registerSize(r) = registerMetadataSize* to
  reflect that, upon allocation, that amount of memory would be additionally
  required.
