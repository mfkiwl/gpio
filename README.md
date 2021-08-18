# GPIO Peripheral

This repository provides an simple GPIO peripheral with integrated register file
to control 64 GPIOs (by default). The peripheral performs two stage
synchronization of the inputs to resolve potential metastability. The outputs
can be driven in push-pull or open-drain mode. Each GPIO supports any
combination of rising-edge, falling-edge, low-level and high-level interrupts
with individual status registers to query the type of pending interrupts.

The interface to the peripheral is the lightweight [register_interface protocol](https://github.com/pulp-platform/register_interface ).
However, the repository contains convenience wrappers to attach AXI-lite or APB
buses for control. Each module in the repository contains an additional wrapper
at the bottom of the respective source files for the users that prefer
SystemVerilog interfaces over hierarchical structs.

# Ports
| **Signal Name**  | **Direction** | **Description**                                                                                                                                               |
| ---------------  | ------------- | ---------------                                                                                                                                               |
| `clk_i`          | *input*       | Primary input clock. The control interface is suposed to be synchronous to this clock.                                                                        |
| `rst_ni`         | *input*       | Asynchronous active-low reset                                                                                                                                 |
| `gpio_in`        | *input*       | GPIO input signals from IO Pads (Pad -> SoC) signal.                                                                                                          |
| `gpio_out`       | *output*      | GPIO output signals to IO Pads (SoC -> Pad) signal.                                                                                                           |
| `gpio_tx_en_o`   | *output*      | GPIO TX Buffer enable signal. This signal is supposed to control the output buffer enable of the IO Pad. 0 -> TX disabled (High-Z or Pull-low/high), 1 -> TX. |
| `gpio_in_sync_o` | *input*       | Synchronized GPIO input signals. This port provides the `gpio_in` signal synchronized to `clk_i`.                                                             |
| `interrupt_o`    | *output*      | Global interrupt line. The interrupt line is asserted for one `clk_i` whenever an unmasked interrupt on one of the GPIOs arrives.                             |
| `reg_req_i`      | *input*       | Control interface request side using register interface protocol.                                                                                             |
| `reg_rsp_o`      | *output*      | Control interface request side using register_interface protocol.                                                                                             |

# Register Map
The registers of this module are all defined in the `gpio_regs.hjson` file which
is used to auto-generate the actual SV register file using [lowRISCs reggen tool](https://docs.opentitan.org/doc/rm/register_tool/ ).

Here is a summary of the registers:
## `INFO` Register (offset 0x00, read-only)
  Contains read-only registers with the number of GPIOs this instance of the
  GPIO peripheral was parametrized for and an IP version number.
  
  | 31 - 20    | 19 - 10    | 9 - 0           |
  |------------|------------|-----------------|
  | *reserved* | IP_VERSION | Number of GPIOs |


## `CFG` Register (offset 0x04, rw)
  Controls the interrupt mode of the gpios. If 1, keep the interrupt line
  asserted until all interrupts for all GPIOs are cleared. If 0, generate one
  cycle wide pulses for every new interrupt.

  | 31 - 1     | 0                |
  |------------|------------------|
  | *reserved* | `interrupt_mode` |

## `GPIO_MODE<0-XX>`  Registers (offset 0x08+<reg_idx>*4, rw)
  The GPIO_MODE registers control the operating mode of the individual GPIOs.
  Each register controls 16 GPIOs. 

   | 31 - 30  | ... | 3 - 2   | 1 - 0   |
   |----------|-----|---------|---------|
   | `GPIO15` | ... | `GPIO1` | `GPIO0` |
  
   *Continues in next register*

  The values for `GPIO0_MODE` are:
  
  | Value | Description                                             |
  |-------|---------------------------------------------------------|
  | 0     | Configures GPIO as an input.                            |
  | 1     | Configures GPIO as a push-pull output.                  |
  | 2     | Configures the GPIO to be in open_drain (drive 1) mode. |
  | 3     | Configures the GPIO to be in open_drain (drive 0) mode. |

## `GPIO_EN<0-XX>` Registers (offset 0x80+<reg_idx>*4, rw)
  Each bit of these registers control the sampling of one GPIO. This register
  enables sampling of the inputs. If disables (0) the corresponding GPIO will
  not sample the inputs (saves power) and will not generate any interrupts.
  
   |       31 | ... |       1 |       0 |
   |----------+-----+---------+---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `GPIO_IN<0-XX>` Registers (offset 0x100+<reg_idx>*4, read-only)
  The bits of these registers contain the input values of the corresponding
  gpios. 

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `GPIO_OUT<0-XX>` Registers (offset 0x180+<reg_idx>*4, rw)
  Change the value of the corresponding GPIO. In `GPIO_MODE 1` writing 1 drives
  high, writing 0 drives low. In `GPIO_MODE 2` (drive 1) writing a 1 will drive
  the gpio to high while writing a 0 will put the gpio in high-z. In `GPIO_MODE
  3` writing 0 drives low and writing 1 puts the gpio into high-z.
  
   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

  *Continues in next register*

## `GPIO_SET<0-XX>` Registers (offset 0x200+<reg_idx>*4, rw)
  For each asserted bit in the register, set the corresponding bit in the
  GPIO_OUT register (masked set). This simplifys setting a single gpio without
  altering the state of the other ones.
  
   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `GPIO_CLEAR<0-XX>` Registers (offset 0x280+<reg_idx>*4, rw)
  For each asserted bit in the register, clear the corresponding bit in the
  GPIO_OUT register (masked clear). This simplifys clearing a single gpio without
  altering the state of the other ones.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `GPIO_CLEAR<0-XX>` Registers (offset 0x300+<reg_idx>*4, rw)
  For each asserted bit in the register, toggle the corresponding bit in the
  GPIO_OUT register (masked toggle). This simplifys toggling a single gpio without
  altering the state of the other ones.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_RISE_EN<0-xx>` Registers (offset 0x380+<reg_idx>*4, rw)
   Enable interrupts on rising edges for the corresponding GPIO. 
   
   The corresponding gpio needs to be enabled `GPIO_EN`. Once an interrupt
   condition is detected, the global interrupt line `interrupt_o` is asserted
   (according to the interrupt mode in `CFG`) and the corresponding bit in the
   interrupt status registers is set. To clear the interrupt, write a `1` to the
   corresponding bit in the status register.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_FALL_EN<0-xx>` Registers (offset 0x400+<reg_idx>*4, rw)
   Enable interrupts on falling edges for the corresponding GPIO. 
   
   The corresponding gpio needs to be enabled `GPIO_EN`. Once an interrupt
   condition is detected, the global interrupt line `interrupt_o` is asserted
   (according to the interrupt mode in `CFG`) and the corresponding bit in the
   interrupt status registers is set. To clear the interrupt, write a `1` to the
   corresponding bit in the status register.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_LVL_HIGH_EN<0-xx>` Registers (offset 0x480+<reg_idx>*4, rw)
   Enable logic-high level-sensitive interrupts for the corresponding GPIO
   (interrupt keeps getting triggered while GPIO is high).
   
   The corresponding gpio needs to be enabled `GPIO_EN`. Once an interrupt
   condition is detected, the global interrupt line `interrupt_o` is asserted
   (according to the interrupt mode in `CFG`) and the corresponding bit in the
   interrupt status registers is set. To clear the interrupt, write a `1` to the
   corresponding bit in the status register.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_LVL_LOW_EN<0-xx>` Registers (offset 0x500+<reg_idx>*4, rw)
   Enable logic-low level-sensitive interrupts for the corresponding GPIO
   (interrupt keeps getting triggered while GPIO is low).
   
   The corresponding gpio needs to be enabled `GPIO_EN`. Once an interrupt
   condition is detected, the global interrupt line `interrupt_o` is asserted
   (according to the interrupt mode in `CFG`) and the corresponding bit in the
   interrupt status registers is set. To clear the interrupt, write a `1` to the
   corresponding bit in the status register.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_STATUS<0-xx>` Registers (offset 0x580+<reg_idx>*4, rw)
   Each bit indicates if there are any pending interrupts on the corresponding
   GPIO. Writing a 1 to a specific bit clears **all** pending interrupts (rise,
   fall, low, high) for the corresponding GPIO.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_RISE_STATUS<0-xx>` Registers (offset 0x600+<reg_idx>*4, rw)
   Each bit indicates if there is a pending rising-edge interrupt on the corresponding
   GPIO. Writing a 1 to a specific bit clears the interrupt for the corresponding GPIO.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_FALL_STATUS<0-xx>` Registers (offset 0x680+<reg_idx>*4, rw)
   Each bit indicates if there is a pending falling-edge interrupt on the corresponding
   GPIO. Writing a 1 to a specific bit clears the interrupt for the corresponding GPIO.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_LVL_LOW_STATUS<0-xx>` Registers (offset 0x700+<reg_idx>*4, rw)
   Each bit indicates if there is a pending low-level sensitive interrupt on the corresponding
   GPIO. Writing a 1 to a specific bit clears the interrupt for the corresponding GPIO.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

## `INTRPT_LVL_HIGH_STATUS<0-xx>` Registers (offset 0x780+<reg_idx>*4, rw)
   Each bit indicates if there is a pending low-level sensitive interrupt on the corresponding
   GPIO. Writing a 1 to a specific bit clears the interrupt for the corresponding GPIO.

   | 31       | ... | 1       | 0       |
   |----------|-----|---------|---------|
   | `GPIO31` | ... | `GPIO1` | `GPIO0` |

   *Continues in next register*

# Changing Number of GPIOs
  Changing the number of GPIOs requires regeneration of the register file to
  include the right number of config registers. The `gpio.sv` will automatically
  adapt accordingly.
  
  The repo contains a Makefile that simplifies the process of invoking the reggen tool for this regeneration.
  E.g. the following comand will reconfigure the project for 48 GPIOs.
  
  ```
  make reconfigure GPIOS=48
  ```
  
