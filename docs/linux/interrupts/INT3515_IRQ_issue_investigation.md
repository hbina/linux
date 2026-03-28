# Investigation Report: INT3515 IRQ Index 1 Not Found

## 1. Issue Identification
The issue was identified from kernel logs (`journalctl -k -b`) on a remote ASUS PN50 machine running Ubuntu with kernel 6.17.0-14-generic.

### Error Messages
```
Feb 16 22:24:06 pn50 kernel: Serial bus multi instantiate pseudo device driver INT3515:00: error -ENXIO: IRQ index 1 not found
Feb 16 22:24:06 pn50 kernel: Serial bus multi instantiate pseudo device driver INT3515:00: error -ENXIO: Error requesting irq at index 1
```

### Impact
The `serial-multi-instantiate` driver failed to initialize multiple TPS6598x USB Power Delivery controllers. Because the probe failed for the second instance (index 1), the entire driver aborted, potentially leaving all USB-C PD controllers on the system non-functional or without interrupt support.

---

## 2. Root Cause Analysis

### Driver Architecture
The driver `drivers/platform/x86/serial-multi-instantiate.c` is a "pseudo" driver. Its purpose is to handle ACPI nodes that describe multiple physical I2C or SPI devices behind a single ACPI hardware ID (HID). It "unpacks" these resources and manually instantiates individual I2C/SPI clients.

### INT3515 (TPS6598x) Handling
The `INT3515` device represents TI TPS6598x USB PD controllers. The driver's static configuration for this device was:
```c
static const struct smi_node int3515_data = {
	.instances = {
		{ "tps6598x", IRQ_RESOURCE_APIC, 0 },
		{ "tps6598x", IRQ_RESOURCE_APIC, 1 },
		{ "tps6598x", IRQ_RESOURCE_APIC, 2 },
		{ "tps6598x", IRQ_RESOURCE_APIC, 3 },
		{}
	},
	.bus_type = SMI_I2C,
};
```
This configuration assumes that the BIOS provides **four** distinct Interrupt resources, one for each I2C client.

### The Discrepancy
Many modern BIOS implementations (specifically on the ASUS PN50 and similar laptops) only provide a **single** interrupt for all controllers in the `INT3515` node, even if they provide multiple I2C address resources.

When the driver successfully instantiated the first controller (`tps6598x` at index 0), it proceeded to the second. It attempted to fetch IRQ index 1 using `platform_get_irq()`, which returned `-ENXIO` because no second interrupt exists in the ACPI table. The driver then treated this as a fatal error and failed the entire probe.

---

## 3. Historical Context
Investigation into the git history (`git log -p -S "INT3515"`) revealed:
*   **Commit 5e63b2ea3dfb**: The driver was renamed to `serial-multi-instantiate` to support SPI.
*   **Commit a3dd034a1707**: Initial introduction of `INT3515` support in 2018.
*   **Commit 2cce82579d09**: A significant comment was added (later moved/removed) noting that `INT3515` has unresolved interrupt issues, including "interrupt floods" and cases where the IRQ line is not connected at all. It noted that the mapping between I2C resources and Interrupt resources is often not 1-to-1.

---

## 4. Fix Implementation

### Strategy
The goal was to make the IRQ resources optional for `INT3515`. If an interrupt is missing for a specific instance, the driver should log a debug message and continue instantiating the remaining devices rather than failing.

### Changes in `drivers/platform/x86/serial-multi-instantiate.c`

#### 1. Generic IRQ Optionality
Improved `smi_get_irq()` to support the `IRQ_RESOURCE_OPT` flag for all interrupt types. Previously, this flag was only respected in the `IRQ_RESOURCE_AUTO` path.

```c
// New logic in smi_get_irq
if (ret < 0 && (inst->flags & IRQ_RESOURCE_OPT)) {
    dev_dbg(&pdev->dev, "No irq at index %d
", inst->irq_idx);
    return 0; // Return success with IRQ 0 (no IRQ)
}
```

#### 2. Updated INT3515 Definition
Changed `int3515_data` to use `IRQ_RESOURCE_AUTO` (to handle both APIC and GPIO transparently) and added the `IRQ_RESOURCE_OPT` flag to all instances.

```c
static const struct smi_node int3515_data = {
	.instances = {
		{ "tps6598x", IRQ_RESOURCE_AUTO | IRQ_RESOURCE_OPT, 0 },
		{ "tps6598x", IRQ_RESOURCE_AUTO | IRQ_RESOURCE_OPT, 1 },
		{ "tps6598x", IRQ_RESOURCE_AUTO | IRQ_RESOURCE_OPT, 2 },
		{ "tps6598x", IRQ_RESOURCE_AUTO | IRQ_RESOURCE_OPT, 3 },
		{}
	},
	.bus_type = SMI_I2C,
};
```

---

## 5. Verification & Validation

### Style Check
The changes were verified using `scripts/checkpatch.pl`:
```bash
git diff drivers/platform/x86/serial-multi-instantiate.c | ./scripts/checkpatch.pl --strict -
# Result: total: 0 errors, 0 warnings, 0 checks, 36 lines checked
```

### Configuration
The driver requires the following kernel configuration:
*   `CONFIG_ACPI=y`
*   `CONFIG_I2C=y`
*   `CONFIG_SERIAL_MULTI_INSTANTIATE=m` (or `y`)

### Compilation
Compilation was tested using:
```bash
make drivers/platform/x86/serial-multi-instantiate.o
```
*Note: A local environment issue with `objtool` and `gelf.h` was encountered but bypassed for the object-file-only validation.*

---

## 6. Future Considerations
If the `tps6598x` driver requires an interrupt to function (e.g., for PD contract negotiation or status updates), these controllers will now operate in a "polled" mode (if supported) or have degraded functionality. However, this is preferable to the entire driver failing to probe, as it allows controllers that *do* have interrupts to work, and others to at least be visible to the system.
