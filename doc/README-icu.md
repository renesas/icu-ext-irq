# ICU

## 1. Overview

This Software Building Block (SBB) demonstrates a typical use of the Renesas Flexible Software Package (FSP) ICU HAL external-interrupt APIs. The reference application initializes and enables the ICU external IRQ driver. When the board user push button generates the configured external interrupt, the interrupt callback sets a flag; the application then clears the flag and toggles the first on-board LED.

This document covers only these reference boards:

- FPB-RA0L1
- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

The source also provides UART console output. The reference application initializes the console, prints SBB/FSP information, reports button presses and LED state, and traps after selected initialization or I/O errors.

## 2. Use Case Demonstrated

The SBB demonstrates an interrupt-driven GPIO event using the ICU HAL:

1. Initialize the external IRQ driver.
2. Enable the configured external interrupt.
3. Detect a user push-button event in the external IRQ callback.
4. Set a software flag from the callback.
5. Process the flag in application context.
6. Read and toggle the first on-board LED.
7. Print status information to the serial terminal.

The core pattern is useful when an application needs to react to an external digital event without continuously polling the input pin.

## 3. Software Organization

### 3.1 Reusable SBB code

The reusable ICU wrapper is implemented by:

- `icu_ep.c`
- `icu_ep.h`

The core exposes initialization, enable, and deinitialization functions around the FSP external IRQ driver. The callback records the interrupt event by setting `g_sw_press` when the callback channel equals `USER_SW_IRQ_NUMBER`.

### 3.2 Demo/reference application code

The reference application is implemented by:

- `icu_example_app.c`
- `icu_example_app.h`

`icu_example_app_run()` is the single application entry point. It initializes the UART console, prints information, initializes/enables the ICU SBB, waits for `g_sw_press`, toggles the first board LED using IOPORT APIs, and prints the resulting LED state.

The UART console (`console/uart_console.h`), terminal messages, LED demonstration, and error-print/trap behavior are demonstration infrastructure rather than the minimal ICU wrapper itself.

### 3.3 Generated/project configuration

The SBB includes `hal_data.h` and uses FSP-generated objects such as `g_external_irq_ctrl`, `g_external_irq_cfg`, and `g_ioport_ctrl`. Their generation and detailed configuration are not described in the supplied `readme.txt`; therefore, preserve or recreate equivalent FSP configuration when porting the SBB.

## 4. Public Functions to Review First

### `icu_init()`

Opens the FSP ICU external IRQ instance by calling `R_ICU_ExternalIrqOpen()` with `g_external_irq_ctrl` and `g_external_irq_cfg`.

```c
fsp_err_t icu_init(void);
```

### `icu_enable()`

Enables the opened external IRQ instance through `R_ICU_ExternalIrqEnable()`.

```c
fsp_err_t icu_enable(void);
```

### `icu_deinit()`

Closes the ICU external IRQ instance with `R_ICU_ExternalIrqClose()` if the control object indicates that it is open.

```c
void icu_deinit(void);
```

### `icu_example_app_run()`

Reference-application entry point. Review this function to understand how the reusable ICU functions are integrated with UART logging, the push-button flag, and LED I/O.

```c
void icu_example_app_run(void);
```

### `irq_ep_callback()`

This is the user-defined external IRQ callback in `icu_ep.c`. When the callback channel matches `USER_SW_IRQ_NUMBER`, it sets `g_sw_press = true`. Although it is part of the interrupt integration, it is not declared in the supplied `icu_ep.h` public header.

## 5. Core Execution Flow

```text
icu_example_app_run()
    |
    +-- uart_console_init()
    +-- R_FSP_VersionGet()
    +-- icu_init()
    |     +-- R_ICU_ExternalIrqOpen()
    +-- icu_enable()
          +-- R_ICU_ExternalIrqEnable()

External push-button interrupt
    |
    +-- irq_ep_callback()
          +-- if channel == USER_SW_IRQ_NUMBER
                  g_sw_press = true

Application loop
    |
    +-- if g_sw_press
          +-- clear flag
          +-- R_IOPORT_PinRead()
          +-- invert LED state
          +-- R_IOPORT_PinWrite()
          +-- print LED state
```

## 6. Reference Board Configuration Assumptions

### FPB-RA0L1

- User input: **SW1 push button**.
- External IRQ channel expected by the source: **IRQ0** (`USER_SW_IRQ_NUMBER = 0x00`).
- Place jumper **J9 on pins 1-2** to use the on-board debug functionality.
- Close **E20** to use **P002 for LED1**.

### FPB-RA2E3

- User input: **S1 push button**.
- External IRQ channel expected by the source: **IRQ0** (`USER_SW_IRQ_NUMBER = 0x00`).
- Close **E25, E28, E46, and E47** to use the J-Link OB VCOM serial terminal connection.
- Connect a jumper wire to short **NMI P200 (J3:21)** to **P206 (J3:18)**.

### FPB-RA4E2

- User input: **S1 push button**.
- External IRQ channel expected by the source: **IRQ9** (`USER_SW_IRQ_NUMBER = 0x09`).
- No additional FPB-RA4E2-specific jumper or wiring requirement is stated in the supplied `readme.txt`.

### FPB-RA6E2

- User input: **S1 push button**.
- External IRQ channel expected by the source: **IRQ9** (`USER_SW_IRQ_NUMBER = 0x09`).
- No additional FPB-RA6E2-specific jumper or wiring requirement is stated in the supplied `readme.txt`.

### FPB-RA8E1

- User input: **S1 push button**.
- External IRQ channel expected by the source: **IRQ13** (`USER_SW_IRQ_NUMBER = 0x0D`).
- No additional FPB-RA8E1-specific jumper or wiring requirement is stated in the supplied `readme.txt`.

For every board, connect the board USB debug port to the host PC with the USB cable appropriate for that board for programming and debugging.

## 7. Serial Terminal Assumptions

The supplied reference documentation specifies the following terminal settings:

- Speed: **115200 baud**
- Data: **8 bit**
- Parity: **none**
- Stop bits: **1 bit**
- Flow control: **none**
- Port: J-Link On-Board provided serial port

For Tera Term, the supplied documentation also specifies:

- `Setup -> Terminal... -> New-line`: Receive = **CR**, Transmit = **CR**
- Enable `Setup -> Terminal... -> Local echo` if typed characters should be echoed.

## 8. What Must Change When Porting to Another Board or Project

The supplied sources show which integration points are board/project dependent, but the supplied `readme.txt` does not provide complete porting values for boards beyond the documented reference configurations.

Review and adapt the following when integrating the SBB:

### Board and pins

- External IRQ input and its FSP configuration (`g_external_irq_cfg`).
- `USER_SW_IRQ_NUMBER` so it matches the configured interrupt channel.
- Board push-button wiring or routing.
- LED selection and IOPORT configuration. The demo operates on `g_bsp_leds.p_leds[0]`.
- Board-specific jumpers, solder bridges, or external wiring described above.

### UART

The demo depends on `uart_console_init()`, `uart_console_flush()`, and `printf()` output. If the target project does not need a console, treat this as demo functionality and integrate the ICU wrapper independently. If console output is retained, make the target UART routing and board connections consistent with the project's console implementation.

### Clocks and timing

No ICU clock frequency, debounce interval, latency target, or timing constraint is specified in the supplied `readme.txt` or source files. Use the values required by the target FSP configuration and application rather than assuming undocumented settings from this reference.

### Memory

No RAM, ROM, heap, or stack requirement is specified in the supplied material. Measure and configure these for the target build if resource budgets are important.

### Task priorities / RTOS

The supplied application uses an infinite foreground loop and does not document RTOS tasks, priorities, queues, semaphores, or scheduling assumptions. If incorporated into an RTOS application, the tasking model and priority choices are application-specific.

### Middleware

No middleware requirement is documented beyond the included UART console abstraction and FSP/BSP driver interfaces used by the example.

## 9. Dependencies and Hardware Resources

The supplied code directly depends on:

- Renesas FSP types and version API.
- FSP ICU HAL external IRQ APIs:
  - `R_ICU_ExternalIrqOpen()`
  - `R_ICU_ExternalIrqEnable()`
  - `R_ICU_ExternalIrqClose()`
- FSP IOPORT APIs used by the demo:
  - `R_IOPORT_PinRead()`
  - `R_IOPORT_PinWrite()`
- FSP-generated/configured objects:
  - `g_external_irq_ctrl`
  - `g_external_irq_cfg`
  - `g_ioport_ctrl`
- BSP LED information through `g_bsp_leds`.
- The project UART console abstraction used by the reference application.
- One configured external IRQ resource associated with the board push button.
- One on-board LED used by the demo.
- USB debug/programming connection.

FPB-RA2E3 additionally requires the jumper-wire connection and solder-bridge configuration listed in the board section. FPB-RA0L1 additionally requires its documented debug jumper and LED solder bridge.

## 10. Demonstrated Behavior and Performance

The demonstrated functional behavior is:

- The ICU external IRQ instance is opened and enabled.
- Pressing the selected user push button triggers the configured external IRQ.
- The callback sets `g_sw_press` for the expected IRQ channel.
- The application detects the flag and toggles the first on-board LED.
- A terminal message reports the button press and resulting LED state.

The supplied material does **not** provide measured interrupt latency, maximum event rate, CPU utilization, memory footprint, debounce performance, power consumption, or other quantitative performance results. These should not be inferred from this reference project.

## 11. File Ownership and Modification Guidance

### Reusable SBB source

- `icu_ep.c`: reusable ICU wrapper and interrupt callback.
- `icu_ep.h`: reusable interface declarations, shared flag declaration, and board-to-IRQ-channel selection.

These are the primary files to review when adapting or extending the SBB behavior.

### Demo/reference application

- `icu_example_app.c`: demonstration integration, UART output, LED toggle logic, and demo error handling.
- `icu_example_app.h`: demonstration entry point and terminal banner/information strings.

Application-specific behavior can be replaced without changing the basic ICU wrapper contract.

### Generated or preset project content

`hal_data.h` and the referenced FSP instances/configuration are generated/project configuration dependencies. The supplied files do not describe their generation policy in detail. In an FSP project, make configuration changes through the project's intended FSP configuration flow rather than relying on undocumented manual edits to generated content.

### Documentation

`readme.txt` is the source documentation supplied with the reference example. This Markdown file consolidates the relevant information for the five boards listed in the scope.

## 12. Updating to a Newer SBB Release After Local Modifications

The supplied files do not define a formal upgrade, merge, or backward-compatibility procedure for adopting a newer SBB release. Therefore, local-source changes should not be assumed to merge automatically with a future release.

A practical source-control strategy is to keep product-specific changes separated from the reusable SBB whenever possible:

1. Keep board/application behavior in application or configuration layers where feasible.
2. Track intentional modifications to `icu_ep.c` and `icu_ep.h` in version control.
3. When adopting a newer SBB revision, compare the new reusable files with the version on which the local changes were based.
4. Reapply or merge only the required product-specific changes, then rebuild and retest the external IRQ and LED behavior on the target board.
5. Recheck FSP-generated configuration if the new release changes required instances, APIs, or configuration assumptions.

This upgrade strategy is integration guidance; it is not specified by the supplied `readme.txt`.

## 13. Customer Integration Checklist

Before reusing this SBB, verify:

- [ ] The target is one of the supported/reference boards or has an equivalent external IRQ configuration.
- [ ] The push-button signal is routed to the configured external IRQ input.
- [ ] `USER_SW_IRQ_NUMBER` matches the callback channel expected for the target board.
- [ ] The FSP external IRQ instance provides `g_external_irq_ctrl` and `g_external_irq_cfg` or the code has been adapted to the project's instance names.
- [ ] The LED used by the reference application is available through `g_bsp_leds.p_leds[0]`, or the demo code has been adapted.
- [ ] Required board jumpers/solder bridges/wires are configured.
- [ ] UART console routing is configured if the demo terminal output is retained.
- [ ] The terminal is configured for 115200, 8 data bits, no parity, 1 stop bit, and no flow control.
- [ ] Target-specific clock, memory, and RTOS requirements have been reviewed because the supplied documentation does not prescribe them.
- [ ] Product-specific changes are isolated and tracked to simplify later SBB updates.

## 14. Source Files Reviewed

- `readme.txt`
- `icu_ep.c`
- `icu_ep.h`
- `icu_example_app.c`
- `icu_example_app.h`
