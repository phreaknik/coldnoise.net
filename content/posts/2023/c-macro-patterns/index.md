+++
date = '2023-03-07T14:35:45-05:00'
draft = false
title = 'Practical C Macro Patterns for Embedded Driver Development'
+++
C macros get a bad rap—often deservedly so. But when developing drivers for 
embedded systems, carefully crafted macros can eliminate entire classes of bugs 
while making your code more maintainable. In this post, I'll walk through 
battle-tested macro patterns from production driver code, using examples from 
[a driver I wrote](https://github.com/phreaknik/snsr288x) for the snsr288x 
family of devices.

One question I get often: _when do you choose macros over inline functions?_

In general, I prefer inline functions whenever possible, but there are two
situations where I commonly reach for macros:
1) **When the logic is simpler/cleaner with _generic_ parameters.** I don't
want to write multiple versions of a function to support different type
variations, if I can avoid it.
1) **When I want to colocate type-specific logic with type definitions in
header files.** Driver development is heavily centered around register
definitions and device-specific data encodings, so I find it helpful to place
related manipulation macros alongside the type definitions themselves.

## Beware Macro Expansion Bugs

Before diving into useful macro patterns, I want to emphasize the use of
parentheses to avoid operator precedence issues that sneak up during macro
expansion. Remember, macro expansion happens before any compilation steps, and
deeply nested macro calls can expand into very large logic combinations very
quickly.

For example:
```rust
// WRONG: Missing parenthesis around parameters
#define DOUBLE(a) 2 * a

// This assertion fails, because macro expansion changes the order of operations
assert(DOUBLE(3 + 2) == 10);
// Macro expansion:
//      DOUBLE(3 + 2)
//   => 2 * 3 + 2
//   => 6 + 2
//   => 8
// assert(8 == 10) fails
```

Wrapping macro parameters in parenthesis prevents this:
```rust
// Correct.
#define DOUBLE(a) (2 * (a))

// This assertion passes
assert(DOUBLE(3 + 2) == 10);
// Macro expansion:
//      DOUBLE(3 + 2)
//   => (2 * (3 + 2))
//   => (2 * 5)
//   => 10
// assert(10 == 10) passes
```

> **Axiom 1**: When in doubt, add more parentheses. The cost is zero at
> runtime, and they prevent an entire class of subtle bugs. Every macro
> parameter should be wrapped in parentheses at every use, and the entire macro
> expansion should be wrapped in parentheses.

There is, however, one scenario that can't be guarded against (easily), so just
beware of it: multiple use of an argument can lead to an error if the argument
is a self-modifying expression, such as the increment/decrement operators. For
example:
```rust
// WRONG: a is used twice
#define SQUARE(a) ((a) * (a))

// This assertion still fails, because the increment operation happens twice
int val = 2;
assert(SQUARE(val++) == 9);
// Macro expansion:
//      SQUARE(val++)
//   => ((val++) * (val++))
//   => ((3) * (val++)) // val now = 3
//   => ((3) * (4)) // 3++ = 4
//   => 12
// assert(12 == 9) fails
```

The problem is that the macro argument is used multiple times in the macro
definition. Some compilers provide tools to solve this, such as the `typeof`
extension in GCC:
```rust
// Correct in GNU/Clang only :/
#define SQUARE(a) ({ \
    typeof(a) _tmp = (a); \
    _tmp * _tmp; \
})
```

But I don't recommend compiler-specific solutions. I would use an inline
function instead of a macro, in this situation:
```rust
static inline int square(int a) {
    return a * a;
}
```

> **Axiom 2**: If you cannot avoid multiple uses of a macro argument, use an
> inline function instead.

## Useful Macro Patterns

Now, on to the macro patterns that have saved me the most headaches in
production...

### 1. Macros for Type Validation

One of the most valuable uses of macros is creating validation helpers for
enums and structs:
```rust
// Enum value range check
#define SNSR288X_ENUM_WITHIN_RANGE(prefix, val) \
  ((prefix##__MIN <= (val)) && (prefix##__MAX >= (val)))

// Check struct fields for null pointers
#define SNSR288X_CTX_IS_VALID(pctx) \
  ((NULL != (pctx)) && SNSR288X_PHY_IS_VALID(&(pctx)->phy) && \
   SNSR288X_DRIVER_CFG_IS_VALID(&(pctx)->drv_cfg))
```

These macros provide compile-time patterns that help ensure runtime safety.
Every enum type defines `__MIN` and `__MAX` sentinels, allowing generic range
validation. Struct validation macros can be composed hierarchically, checking
both null pointers and the validity of nested structures. You can be more
thorough than this, but I'm satisfied with checking for unexpected null
pointers.

This pattern catches invalid parameters at API boundaries:
```rust
// Configure the DAC and return the driver error type
snsr288x_error_t snsr288x_configure_dac(
    snsr288x_ctx_t* ctx,
    snsr288x_dac_sel_t dac_sel,
    const snsr288x_dac_cfg_t* dac_cfg)
{
  assert(SNSR288X_CTX_IS_VALID(ctx));
  assert(SNSR288X_DAC_SEL_IS_VALID(dac_sel));
  assert(SNSR288X_DAC_CFG_IS_VALID(dac_cfg));
  // ...
}
```

### 2. Register Field Manipulation

Hardware register programming is error-prone when done manually. I prefer
defining a set of get/set macros to encapsulate all of the bit manipulation
necessary to access registers and their fields. Then, using the datasheet as a
reference, I define register values as well as _MASK and _SHIFT values needed
by the get/set macros.

First, define get/set macros. These encapsulate the bit manipulation logic:
```rust
// Get the bits of the specified field within the specified register, from
// reg_val read from the device
#define SNSR288X_GET_FIELD(reg, field, reg_val) \
  (((reg_val) & SNSR288X_REG__##reg##__##field##__MASK) >> \
   (SNSR288X_REG__##reg##__##field##__SHFT))

// Set the bits of the specified field within the specified register.
// Optionally reg_val initializes the state of the register.
// field_val specifies the new integer value for that field
#define SNSR288X_SET_FIELD(reg, field, reg_val, field_val) \
  (((reg_val) & ~(SNSR288X_REG__##reg##__##field##__MASK)) | \
   (((field_val) << (SNSR288X_REG__##reg##__##field##__SHFT)) & \
    SNSR288X_REG__##reg##__##field##__MASK))
```

Then define register values (see
[snsr288x_regs.h](https://github.com/phreaknik/snsr288x/blob/master/inc/snsr288x_regs.h)) for
a thorough example. These may look something like:
```rust
#define SNSR288X_REG__INTERRUPT_ENABLE_1__ADDR (0x05u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__VDD_OOR_EN__MASK (0x02u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__VDD_OOR_EN__SHFT (1)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__INVALID_CFG_EN__MASK (0x04u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__INVALID_CFG_EN__SHFT (2)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__EIS_CAL_DONE_EN__MASK (0x08u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__EIS_CAL_DONE_EN__SHFT (3)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__AC_DATA_RDY_EN__MASK (0x10u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__AC_DATA_RDY_EN__SHFT (4)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__FIFO_DATA_RDY_EN__MASK (0x40u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__FIFO_DATA_RDY_EN__SHFT (6)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__A_FULL_EN__MASK (0x80u)
#define SNSR288X_REG__INTERRUPT_ENABLE_1__A_FULL_EN__SHFT (7)
```

Usage is self-documenting:
```rust
// ... first, read INTERRUPT_ENABLE_1 register.
// ... assume reg_val points to this value

// See if the VDD_OOR interrupt is enabled
bool is_vdd_oor_interrupt_enabled =
    SNSR288X_GET_FIELD(INTERRUPT_ENABLE_1, VDD_OOR_EN, *reg_val);

// Update the register to set the VDD_OOR interrupt enable bit
*reg_val = SNSR288X_SET_FIELD(
    INTERRUPT_ENABLE_1, VDD_OOR_EN, *reg_val, true);
```

The register and field names appear explicitly in the code, making it easy to
cross-reference with hardware documentation. The macros handle all the bit
manipulation automatically, reducing the chances of human error when
manipulating bits.

### 3. Compile-Time Calculations

Macros can compute values that would otherwise require magic numbers. One such
example is when defining the allowable range of values. I like to use a
range-size macro:
```rust
#define SNSR288X_ENUM_RANGE_SIZE(prefix) \
  (1 + (prefix##__MAX) - (prefix##__MIN))
```

For example, in the snsr288x ADC config, we define up to 4 channels. The
range-size macro can then be used to help validate channel selection later:
```rust
typedef enum
{
  SNSR288X_CHAN__1,
  SNSR288X_CHAN__2,
  SNSR288X_CHAN__3,
  SNSR288X_CHAN__4,

  SNSR288X_CHAN__MIN = SNSR288X_CHAN__1,
  SNSR288X_CHAN__MAX = SNSR288X_CHAN__4
} snsr288x_channel_t;
#define SNSR288X_CHAN_IS_VALID(channel)                                        \
  SNSR288X_ENUM_WITHIN_RANGE(SNSR288X_CHAN, channel)
#define SNSR288X_MAX_NUM_CHANNELS SNSR288X_ENUM_RANGE_SIZE(SNSR288X_CHAN)
```

And we can go so far as to create a macro to easily dynamically compute the
number of ADC channels from the `part_id` we read from the device:
```rust
// SENSOR2881 has 1 ADC channel
#define NUM_CHANNELS_IN_PART(part_id) \
  (((SNSR288X_ID__SENSOR2881 == (part_id)) * SENSOR2881_NUM_CHANNELS) + \
   ((SNSR288X_ID__SENSOR2882 == (part_id)) * SENSOR2882_NUM_CHANNELS) + \
   ((SNSR288X_ID__SENSOR2884 == (part_id)) * SENSOR2884_NUM_CHANNELS))
```

The `NUM_CHANNELS_IN_PART` macro is particularly interesting—it uses
multiplication with boolean expressions to select the correct value without
branching. This evaluates at compile time when possible, but thanks to compiler
optimizations, it still ends up being very efficient in the cases where it
needs to be evaluated at runtime.

### 4. Register Address Calculations

Hardware often has regular patterns in register layouts. Macros can exploit
this:
```rust
#define FIRST_REG_IN_CHANNEL(chan) \
  (((SNSR288X_CHAN__1 == (chan)) * \
    SNSR288X_REG__SENSOR_1_CONFIGURATION_1__ADDR) + \
   ((SNSR288X_CHAN__2 == (chan)) * \
    SNSR288X_REG__SENSOR_2_CONFIGURATION_1__ADDR) + \
   ((SNSR288X_CHAN__3 == (chan)) * \
    SNSR288X_REG__SENSOR_3_CONFIGURATION_1__ADDR) + \
   ((SNSR288X_CHAN__4 == (chan)) * \
    SNSR288X_REG__SENSOR_4_CONFIGURATION_1__ADDR))
```

This allows code to work generically across hardware instances:
```rust
// Write a contiguous span of registers for the given ADC channel, to ammortize
// the cost of the SPI communication overhead
const uint8_t base_addr = FIRST_REG_IN_CHANNEL(channel);
spi_write(ctx, base_addr, &ctx->scratch[0], 
          SNSR288X_CHANNEL_ADDR_RANGE_SIZE);
```

### 5. Bulk Register Operations

When dealing with register sequences, these macros reduce repetition:
```rust
#define REG_RANGE_SIZE(first_reg, last_reg) \
  (1 + SNSR288X_REG__##last_reg##__ADDR - \
   SNSR288X_REG__##first_reg##__ADDR)

#define READ_REG_RANGE(dest, first_reg, last_reg) \
  do { \
    spi_read(ctx, \
             SNSR288X_REG__##first_reg##__ADDR, \
             (dest), \
             REG_RANGE_SIZE(first_reg, last_reg)); \
  } while (0)

#define WRITE_REG_RANGE(dest, first_reg, last_reg) \
  INIT_REGS(dest, first_reg, REG_RANGE_SIZE(first_reg, last_reg))
```

This makes register block operations concise and readable:
```rust
READ_REG_RANGE(&data[0], STATUS_1, STATUS_5);
WRITE_REG_RANGE(&data[0], INTERRUPT_ENABLE_1, INTERRUPT_ENABLE_5);
```

### 6. Encapsulating Verbose Data Operations

This one is perhaps the most obvious use of macros: to encapsulate a bit of
verbose logic into something much less verbost. I felt this post would be
incomplete without such an example, though, so I decided to pull one of the
more interesting examples out of the snsr288x drivers.

In this example, `SNSR288X_GET_SAMPLE_TAG(psample)` reads a specially encoded
data sample from the sensor and decodes the sample to extract just its tag
value:
```rust
#define SNSR288X_FIFO_SAMPLE_IS_16BITS(psamp) \
  SNSR288X_ENUM_WITHIN_RANGE(SNSR288X_SAMPLE_TAG__U16, \
                             (psamp)->bytes[0] & 0x0Fu)

#define SNSR288X_GET_SAMPLE_TAG(psample) \
  (snsr288x_sample_tag_t)( \
    (SNSR288X_FIFO_SAMPLE_IS_16BITS(psample) * \
     ((psample)->bytes[0] & 0x0F)) + \
    (SNSR288X_FIFO_SAMPLE_IS_12BITS(psample) * \
     ((psample)->bytes[0] & 0x0F) << 4) + \
    (SNSR288X_FIFO_SAMPLE_IS_12BITS(psample) * \
     ((psample)->bytes[1] & 0xF0) >> 4))
```

### 7. Multi-Statement Macros: The `do { } while(0)` Idiom

When a macro needs to execute multiple statements, wrapping them in 
`do { } while(0)` ensures the macro behaves like a single statement in all 
contexts.

Without this wrapper, macros break in subtle ways:

```rust
// WRONG: Multiple statements without do-while wrapper
#define CLEAR_AND_SET(reg, val) \
  (reg) = 0; \
  (reg) = (val)

// This looks fine...
CLEAR_AND_SET(my_register, 0x42);

// But breaks with control flow!
if (reset)
  CLEAR_AND_SET(my_register, 0x42);

// Expands to:
if (reset) {
  my_register = 0;  // Only this is inside the if!
}
my_register = 0x42;  // This always executes!
```

The `do { } while(0)` wrapper creates a proper statement block:

```rust
// CORRECT: Wrapped in do-while
#define CLEAR_AND_SET(reg, val) \
  do { \
    (reg) = 0; \
    (reg) = (val); \
  } while (0)

// Now it works correctly in all contexts
if (reset)
  CLEAR_AND_SET(my_register, 0x42);

// Expands to:
if (reset) {
  do {
    my_register = 0;
    my_register = 0x42;
  } while (0);
}
```

Here's a real example from the driver code:

```rust
#define READ_REG_RANGE(dest, first_reg, last_reg) \
  do { \
    spi_read(ctx, \
             SNSR288X_REG__##first_reg##__ADDR, \
             (dest), \
             REG_RANGE_SIZE(first_reg, last_reg)); \
  } while (0)

// Can be used safely anywhere a statement is expected:
if (needs_status_update)
  READ_REG_RANGE(&ctx->scratch[0], STATUS_1, STATUS_5);

for (int i = 0; i < retry_count; i++)
  READ_REG_RANGE(&buffer[i], DATA_START, DATA_END);
```

The `do { } while(0)` idiom ensures your multi-statement macros:
- Work correctly in all control flow contexts (if/else, loops, switch)
- Require a trailing semicolon (matching normal C statement syntax)
- Don't accidentally consume the semicolon and break subsequent else clauses
- Act as a single statement for scoping purposes

## Best Practices

I've given a lot of examples, but to sum it up, here is a non-exhaustive list
of best practices I try to be aware of:
1. **Always parenthesize macro arguments** to prevent operator precedence
   issues
1. **Create validation macros for every user-facing type** to catch errors at
   API boundaries
1. **Leverage token pasting (`##`)** to create naming conventions that scale
   across register sets
1. **Design macros to fail at compile time** when possible, rather than
   producing runtime errors
1. **Use `do { } while(0)` for multi-statement macros** to ensure they work
   correctly in all contexts

## Conclusion

There's a fine balance to strike when writing C macros. When done well, they
can drastically improve the robustness and readability of a codebase. I find
this especially true in device drivers, whose logic is riddled with bespoke
register definitions and bit manipulations.

Of course, I would be remiss if I did not mention that most of this effort is
unnecessary in modern languages like Rust :)
