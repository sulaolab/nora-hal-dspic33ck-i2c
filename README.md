# nora-hal-dspic33ck-i2c

**NORA-HAL** — *Native On-chip Resource Assistant*

Small, readable I2C HAL for Microchip dsPIC33CK devices, with a blocking **master** API
and an interrupt-driven, callback-based **slave** API — part of **NORA-HAL**, a HAL family
whose public API is namespaced `nora_*` / `NORA_*`.

This project is intended as a compact alternative to large generated driver code. The goal
is not to hide everything behind a framework, but to provide a simple driver that is easy
to read, test, modify, and adapt.

The master and slave roles live in separate headers so a program includes only what it
uses:

* `nora_i2c_master.h` — bus-master API (include for master use)
* `nora_i2c_slave.h` — slave/device API (include for slave use)
* `nora_i2c.h` — shared types and lifecycle, pulled in by both

A program may include either or both.

## Naming

The public API is `nora_*` / `NORA_*`. This repository has carried that namespace since
its first commit — there is no earlier `dspic33ck_*` public API here and therefore no
compatibility aliases to remove.

The chip name survives in exactly two places, both deliberate:

* **Implementation file names** carry a backend tag: `nora_i2c_dspic33ck_master.c` is the
  dsPIC33CK backend of the processor-neutral `nora_i2c_master.h`. A second processor would
  add its own `nora_i2c_<tag>_master.c` beside it, not a second header.
* **Backend-private identifiers** inside those files (register-layer macros and statics),
  which no caller sees. The two internal headers carry the tag for the same reason.

The tag is `_dspic33ck` — a different silicon family from dsPIC33AK (dsPIC33**C** vs
dsPIC33**A**), and never abbreviated to `_dspic33c`.

### Relationship to the dsPIC33AK HAL of the same name

[nora-hal-dspic33ak-i2c](https://github.com/sulaolab/nora-hal-dspic33ak-i2c) is the same
API shape for the dsPIC33AK family, and the master surface matches — the same blocking
calls, the same low-level primitives, the same pending-transaction model. The silicon is a
**different I2C module**, and one difference reaches the integrator:

| | dsPIC33CK (here) | dsPIC33AK |
|---|---|---|
| peripheral | the **classic 16-bit** module (`I2CxCONL` / `CONH` / `STAT` / `BRG`) | the 32-bit "new" module |
| slave interrupt | one dedicated `SI2Cx` interrupt that already fires on address match, on every data byte, and — with `PCIE` — on STOP | address / data / STOP aggregated into one event interrupt routed through an `INTC` register |
| **who owns the slave vector** | **this HAL defines `_SI2CxInterrupt` itself** (in the device layer); the delegate is also exported for an integration that wants to own the vector | the application owns it and forwards to `nora_i2c_slave_event_irq()` |
| repeated START / STOP completion | `I2CxCONL.RSEN` / `PEN` on the classic register layout | `RSEN`, plus a documented `STOPE`-before-`PEN` ordering workaround |

**That vector-ownership row is a real difference, not a wording difference**: a project that
defines its own `_SI2CxInterrupt` gets a duplicate-symbol error against this backend, whereas
against the published AK HAL it is required to define one. It is the classic peripheral's
single dedicated slave interrupt that makes the turnkey form possible here — no
interrupt-routing register is involved.

The dsPIC33AK and dsPIC33CK fleets are **not** symmetric, and nothing here should be read
as a claim that they are.

## Status

Validation target:

* Devices: dsPIC33CK64MC105 (EV88G73A Curiosity Nano), dsPIC33CK256MP508 (DM330030)
* Compiler: XC-DSC v3.31.01
* DFP: dsPIC33CK-MC_DFP 1.10.386 / dsPIC33CK-MP_DFP 1.15.423 or compatible

**How to read this.** These HALs are built for evaluation, FAE demos and architecture
experiments, so exhaustive per-function coverage was never the goal — there is no unit-test
suite, and the evidence is integration testing on real hardware. Three tiers, used across
the seven sibling repositories: **integration-verified** (it ran in the working system and something
observable would have broken otherwise), **hardware-observed, not a matrix** (it worked in
the configuration actually run), **compiled, not executed**.

**Integration-verified — master, I2C1 at 400 kHz against a WM8904 codec:**

* The codec's device ID reads **`0x8904`** with the codec wired, and bring-up **aborts** if
  it does not — no stage runs after an unconfirmed ID. Both halves are hardware evidence:
  the positive read, and the refusal.
* With **no codec attached** the probe reports a clean `ERR_NACK`, which is what
  distinguishes "the module works, nobody answered" from "the bus is broken".
* Blocking write, blocking read, and write-read with repeated START all run as the codec
  control path: the audio system reached `passthrough running` on five consecutive boots
  including a cold power cycle, and every configuration write is read back and verified
  before the next stage.
* The probe names *where* it probed (pins, rate) in its own report.

**Hardware-observed, not a matrix:**

* **Bus speed.** 400 kHz works on this fleet; **100 kHz does not** — a board/bus fact
  recorded upstream, not a defect in this HAL, and the reason 400 kHz is the stated rate.
  `bus_hz` is a **request, not a guarantee**: `nora_i2c_init()` and
  `nora_i2c_set_bus_speed()` convert it to the single `I2CxBRG` reload value, so achievable
  rates are quantized to `Fcy / (BRG + 1 + Fcy * TDELAY)` and a combination the divider
  cannot express is **clamped and still reported as `OK`** — only a zero frequency is
  refused. `TDELAY` is the module's fixed internal SDA/SCL delay, taken here as a 130 ns
  estimate, which is the term to check with a scope if a rate reads slightly off. (The AK module
  has a documented slow-speed hazard of its own — `STOPE` setting before `PEN` clears, which
  at 100 kHz let the next START be ignored, since diagnosed and fixed there. The classic CK
  peripheral has its own `PEN` timing and that analysis has not been repeated here, so do
  not assume either the bug or the fix transfers.)
* **Instances.** Only **I2C1** has been used on either board. The enum reaches three and
  presence is a device question (`nora_i2c_is_present()`).

**Compiled, not executed:**

* **The slave role.** It builds in both configurations, but **there is no caller anywhere
  outside `src/hal_i2c/` anywhere in the validating application** — no board, no app code. The register
  sequence and the callback contract have been written and compiled and have never answered
  a real master. `nora_i2c_slave.h` says the same where it matters.
* **dsPIC33CK256MP508**: that configuration is compile-only upstream, so its I2C paths
  compile and do not run.

## Design policy

This driver is intentionally small.

* The normal API is blocking and simple.
* The low-level API separates *issue* and *status-check* operations, so a user can replace
  the polling loops with interrupt flags, RTOS waits, or cooperative-scheduler waits.
* No XC-DSC / DFP bitfield structures are exposed in the public API.
* Device-specific register symbols and the `_MI2CxIF` / `_SI2CxIF` interrupt symbols are
  confined to the device mapping layer.
* Timeout handling is **opt-in**: it exists only if the caller supplies a millisecond
  tick. There is no hidden time source.
* No dynamic memory, no RTOS dependency.

## Files

```text
src/
  nora_i2c.h                      shared types + lifecycle + nora_i2c_status_str()
  nora_i2c_master.h               master config + blocking / low-level / irq API
  nora_i2c_slave.h                slave config (callbacks) + slave API
  nora_i2c_dspic33ck_master.c     master engine
  nora_i2c_dspic33ck_slave.c      slave interrupt engine
  nora_i2c_dspic33ck_common.c     shared primitives (instance validation, register
                                  resolution, BRG calculation, presence)
  nora_i2c_dspic33ck_internal.h   backend-private, shared by the three .c files
  nora_i2c_dspic33ck_device.c     device register mapping + the _SI2CxInterrupt vectors
  nora_i2c_dspic33ck_device.h
  nora_i2c_dspic33ck_reg.h        internal register/bit definitions
```

## Basic usage (master)

```c
#include "nora_i2c_master.h"

static uint32_t app_get_ms(void) { return app_millisecond_tick; }

void app_i2c_init(void)
{
    const nora_i2c_config_t cfg = {
        .fcy_hz             = 100000000u,   /* Fcy */
        .bus_hz             = 400000u,      /* the validated speed */
        .timeout_ms         = 10u,
        .get_ms             = app_get_ms,
        .pending_timeout_ms = 0u,
    };

    (void)nora_i2c_init(NORA_I2C_INST_1, &cfg);
}
```

If `get_ms` is `NULL`, timeout handling is disabled. If `timeout_ms` is `0`, timeout
handling is also disabled. If `pending_timeout_ms` is `0`, stale pending-transaction
recovery is disabled. **All three are silent, by design** — `timeout_ms = 10` is *inert*
while `get_ms` is `NULL`, and both upstream boards deliberately leave it `NULL`, so a
transaction that hangs hangs. If you want timeouts, supply the tick; the HAL will not
invent one.

`addr7` arguments in this HAL are always **right-justified 7-bit** addresses. Do not pass
the R/W bit, and do not pass the already-shifted 8-bit form a datasheet may also print:
WM8904 is `0x1A` here, not `0x34`. Anything above `0x7F` is refused with
`ERR_INVALID_ARG` — `0x34` would otherwise go out as `0x34 << 1 = 0x68`, a legal address
byte for a different device, and fail as a wrong target or a puzzling `ERR_NACK`.

An instance can hold only one role at a time. `nora_i2c_init()` on an instance that is live
as a slave returns `ERR_BUSY`, and `nora_i2c_slave_init()` on a live master does the same;
release the current role (`nora_i2c_deinit()` / `nora_i2c_slave_deinit()`) first.
Re-initializing the role an instance already holds stays allowed. Note also that
`nora_i2c_deinit()`, despite living in the shared header, releases the **master** role only.

```c
uint8_t tx[2] = { 0x01u, 0x02u };
(void)nora_i2c_write(NORA_I2C_INST_1, 0x1au, tx, sizeof tx);

uint8_t reg = 0x00u, rx[2];
(void)nora_i2c_write_read(NORA_I2C_INST_1, 0x1au, &reg, 1u, rx, sizeof rx);
```

`nora_i2c_write_read()` generates a repeated START between the write and read phases.
`nora_i2c_status_str()` turns any status into a short name (`"ERR_NACK"`), and it lives
with the enum rather than with a caller because every caller that reports a failure needs
the same table.

## Runtime bus speed change

```c
(void)nora_i2c_set_bus_speed(NORA_I2C_INST_1, 100000000u /* fcy_hz */, 400000u /* bus_hz */);
```

The instance must be initialized and idle. The peripheral is briefly turned off around the
BRG write and its previous ON state restored; the same BRG calculation as `nora_i2c_init()`
is used. Returns `ERR_INVALID_ARG` (invalid instance or a zero frequency),
`ERR_NOT_PRESENT`, `ERR_NOT_INITIALIZED`, `ERR_BUSY` (host state machine active or a
no-STOP transaction pending), or `OK`. It does **not** run stale-pending recovery —
recovery stays with the read-after-restart / stop / deinit paths.

## Pending transaction API

For register-read style transfers `nora_i2c_write_read()` is usually enough. When an
application needs explicit control of a repeated-START sequence:

```c
(void)nora_i2c_master_write_no_stop(NORA_I2C_INST_1, 0x1au, &reg, 1u);
(void)nora_i2c_master_read_after_restart(NORA_I2C_INST_1, 0x1au, rx, sizeof rx);
```

`..._write_no_stop()` leaves the bus active without issuing STOP; `..._read_after_restart()`
issues a repeated START, reads, then STOPs. To abandon a pending transaction without a read
phase, call `nora_i2c_master_stop()`.

If both `get_ms` and `pending_timeout_ms` are set, a later public call detects an expired
pending transaction, attempts STOP, clears the pending state and returns
`NORA_I2C_ERR_TIMEOUT`.

## I2C slave (device) mode

Include `nora_i2c_slave.h` to answer as a slave. 7-bit addressing only — 10-bit and
general call are not handled.

```c
#include "nora_i2c_slave.h"

static uint8_t reg_file[8];
static uint8_t rx_idx, tx_idx;

static void    on_addr(bool is_read) { if (is_read) tx_idx = 0; else rx_idx = 0; }
static void    on_rx(uint8_t b)      { if (rx_idx < sizeof reg_file) reg_file[rx_idx++] = b; }
static uint8_t on_tx(void)           { return (tx_idx < sizeof reg_file) ? reg_file[tx_idx++] : 0xFFu; }
static void    on_stop(void)         { /* transaction complete */ }

void app_i2c_slave_init(void)
{
    const nora_i2c_slave_config_t cfg = {
        .addr7         = 0x55u,
        .addr_mask     = 0u,      /* I2CxMSK low 7 bits; 0 = exact match */
        .clock_stretch = false,   /* STREN: hold SCL for slower callbacks */
        .on_addr_match = on_addr,
        .on_rx_byte    = on_rx,
        .on_tx_byte    = on_tx,
        .on_stop       = on_stop,
    };

    /* The slave line, not the master one -- see below. Before init(), which
     * enables it. */
    (void)nora_i2c_slave_set_interrupt_priority(NORA_I2C_INST_2, 4u);
    (void)nora_i2c_slave_init(NORA_I2C_INST_2, &cfg);
}
```

**The vector is already defined for you.** The device layer defines `_SI2CxInterrupt` for
each instance that exists on the target; each clears its flag and calls
`nora_i2c_slave_event_irq(inst)`. That call is exported, so an integration that wants to
own the vector — or a host-side unit test — can drive the same service routine directly.
Set the slave line's priority with **`nora_i2c_slave_set_interrupt_priority()`**, before
`nora_i2c_slave_init()` — `init()` enables the line, so raising the priority afterwards is a
race there is no reason to take. Note the pairing: `nora_i2c_set_interrupt_priority()`
programs the **master** (`MI2Cx`) line only. The classic peripheral has three separate
interrupts, so there is no single "I2C priority", and the slave setter refuses rather than
returning OK on a part with no `_SI2CxIP` symbol for that instance.

Callback contract:

* `on_addr_match(is_read)` — the master addressed this device. `is_read` is true when the
  master will read from us (we transmit via `on_tx_byte`), false when it will write to us.
* `on_rx_byte(b)` — one received byte (master write). `NULL` drops the byte.
* `on_tx_byte()` — return the next byte to transmit; `0xFF` is sent if this is `NULL`.
* `on_stop()` — the transaction ended.

Callbacks run in interrupt context, so keep them short. With `clock_stretch = true` the
slave holds SCL (`STREN` / `SCLREL`) to give them more time. `nora_i2c_slave_is_active()`
reports whether the instance has been configured as a slave.

Remember the Status section: **this role has never run on hardware.**

## Low-level primitive API

The blocking API is built from small primitives — `..._ll_start_issue()`,
`..._ll_restart_issue()`, `..._ll_stop_issue()`, `..._ll_write_byte_issue()`,
`..._ll_read_byte_issue()`, `..._ll_ack_issue()`, and their `_busy` / `_done` / `_ready` /
`_acked` companions, plus `..._ll_has_nack()` / `_has_error()` / `_has_collision()`.

They expose simple hardware-oriented issue/done checks; the blocking API adds the extra
sequencing needed for safe back-to-back transactions. If you build your own engine on the
primitives, that extra sequencing is yours to reproduce.

## Interrupt helpers

* `nora_i2c_set_interrupt_priority()` — the **master** (`MI2Cx`) priority.
* `nora_i2c_slave_set_interrupt_priority()` — the **slave** (`SI2Cx`) priority, declared in
  `nora_i2c_slave.h`. Both take 0..7; 0 leaves the source masked by CPU priority rules, and
  both return `NORA_I2C_ERR_UNSUPPORTED` rather than OK if the selected part has no such
  priority symbol for that instance.
* `nora_i2c_irq_enable()` / `_irq_disable()` / `_irq_clear()` — **declared for API parity
  with the dsPIC33AK HAL and currently returning `NORA_I2C_ERR_UNSUPPORTED` on this
  backend.** They are reserved for a future interrupt-helper layer; there is no
  interrupt-driven master transfer engine here. The backend-private
  `nora_i2c_slave_irq_*()` helpers in `nora_i2c_dspic33ck_device.h` are a different,
  silicon-specific thing and are not these.

## Notes

* This repository does not include Microchip DFP header files.
* Lifecycle and presence are shared: `nora_i2c_is_present()`, `nora_i2c_is_initialized()`,
  `nora_i2c_deinit()`. Presence is a device question, answered from the device mapping
  layer rather than from a part number.
* Sibling repositories for this family:
  [dma](https://github.com/sulaolab/nora-hal-dspic33ck-dma) ·
  [timer](https://github.com/sulaolab/nora-hal-dspic33ck-timer) ·
  [gpio](https://github.com/sulaolab/nora-hal-dspic33ck-gpio) ·
  [clock](https://github.com/sulaolab/nora-hal-dspic33ck-clock) ·
  [uart](https://github.com/sulaolab/nora-hal-dspic33ck-uart) ·
  [spi-i2s-tdm](https://github.com/sulaolab/nora-hal-dspic33ck-spi-i2s-tdm)

## License

MIT No Attribution License (MIT-0). See [LICENSE](LICENSE).

Attribution is appreciated but not required.
