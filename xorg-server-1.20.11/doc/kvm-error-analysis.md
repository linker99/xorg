# KVM Internal Error Analysis

## Error Summary

This document analyzes the following KVM internal error and provides solutions.

```
KVM internal error. Suberror: 3
extra data[0]: 800000fd
extra data[1]: 31
```

## Register State at Error

| Register | Value              | Interpretation                        |
|----------|--------------------|-----------------------------------------|
| RAX      | 00000000ffffffed   | Return value -19 (ENODEV)              |
| RBX      | ffffffff81b25860   | Kernel pointer                         |
| RCX      | 0100000000000000   | MSR number (invalid/reserved)          |
| RDX      | 0000000000000000   | Zero                                   |
| RSI      | 0000000000000000   | Zero                                   |
| RDI      | 0000000000000046   | 70 decimal                             |
| RBP      | ffff88017014fea8   | Stack base pointer                     |
| RSP      | ffff88017014fea8   | Stack pointer                          |
| RIP      | ffffffff816ad716   | Kernel code (native_safe_halt area)    |
| RFL      | 00000286           | Flags: [--S--P-]                        |
| CPL      | 0                  | Ring 0 (kernel mode)                   |

## Error Interpretation

### Suberror Code 3: KVM_INTERNAL_ERROR_EMULATION

This error indicates that the KVM hypervisor could not emulate a privileged CPU instruction. This typically occurs when:

1. The guest tries to execute an instruction that requires special handling by the hypervisor
2. The hypervisor doesn't support or cannot emulate that specific instruction
3. There's a mismatch between guest expectations and host capabilities

### Extra Data Analysis

- **extra data[0]: 0x800000fd**
  - Bit 31 (0x80000000) = The failing instruction was recognized as valid
  - Lower bits (0xfd = -3 signed) = Error code related to permission/access issues

- **extra data[1]: 0x31**
  - Exit qualification or additional error information (49 decimal)

## Instruction Analysis

### Machine Code at RIP

```
Code=48 00 00 00 89 c2 0f 30 e9 66 ff ff ff 90 55 48 89 e5 fb f4 <5d> c3 ...
```

The `<5d>` marker indicates the instruction pointer position. Decoding the instructions:

| Bytes     | Instruction       | Description                           |
|-----------|-------------------|---------------------------------------|
| 89 c2     | mov edx, eax      | Move EAX to EDX                       |
| 0f 30     | wrmsr             | Write to Model-Specific Register      |
| e9 66...  | jmp rel32         | Relative jump                         |
| 90        | nop               | No operation                          |
| 55        | push rbp          | Save base pointer                     |
| 48 89 e5  | mov rbp, rsp      | Set up stack frame                    |
| fb        | sti               | Enable interrupts                     |
| f4        | hlt               | Halt until interrupt                  |
| **5d**    | **pop rbp**       | **<- Instruction pointer here**       |
| c3        | ret               | Return from function                  |

### Identified Function: native_safe_halt()

The instruction sequence `sti; hlt; pop rbp; ret` is the Linux kernel's `native_safe_halt()` function, used for CPU idle states:

```c
static inline void native_safe_halt(void)
{
    asm volatile("sti; hlt" : : : "memory");
}
```

## Root Cause

The error is caused by one of these privileged instructions failing to be emulated by KVM:

### 1. WRMSR (Write to Model-Specific Register) - Most Likely Cause

- Opcode: `0f 30`
- The RCX register contains `0x0100000000000000`, which specifies the MSR number
- This MSR value appears to be invalid or unsupported
- KVM could not emulate the write to this MSR

### 2. HLT (Halt) Instruction

- Opcode: `f4`
- Part of the CPU idle code
- Usually works fine under KVM, but may fail with specific configurations

### 3. STI (Enable Interrupts) Instruction

- Opcode: `fb`
- Should be transparent to KVM in most cases

## Solutions

### Solution 1: Configure KVM to Ignore Unknown MSRs

Add the `ignore_msrs=1` parameter to the KVM module:

```bash
# Temporary (until reboot):
sudo modprobe -r kvm_intel  # or kvm_amd for AMD CPUs
sudo modprobe kvm ignore_msrs=1
sudo modprobe kvm_intel     # or kvm_amd

# Permanent:
echo "options kvm ignore_msrs=1" | sudo tee /etc/modprobe.d/kvm.conf
```

**Note:** This will log warnings instead of crashing, but the guest may have reduced functionality.

### Solution 2: Modify Guest Kernel Boot Parameters

Add these parameters to the guest kernel command line (in GRUB):

```
idle=poll          # Use polling instead of HLT for idle
idle=nomwait       # Disable MWAIT instruction for idle
no_timer_check     # Skip timer setup checks
```

Edit `/etc/default/grub` in the guest:
```
GRUB_CMDLINE_LINUX="idle=poll no_timer_check"
```

Then run:
```bash
sudo update-grub
sudo reboot
```

### Solution 3: Update QEMU CPU Configuration

When starting the VM, use better CPU options:

```bash
# Pass through host CPU features:
qemu-system-x86_64 -cpu host ...

# Or with specific paravirtualized features:
qemu-system-x86_64 -cpu host,+kvm_pv_unhalt,+kvm_pv_eoi ...
```

### Solution 4: Update Software Versions

1. **Update QEMU/KVM** on the host:
   ```bash
   # Ubuntu/Debian
   sudo apt update && sudo apt upgrade qemu-kvm
   
   # RHEL/CentOS/Fedora
   sudo dnf update qemu-kvm   # or 'yum' on older systems
   ```

2. **Update Guest Kernel**:
   Older kernels may attempt to write to deprecated or unsupported MSRs. Update to a newer kernel version.

### Solution 5: Enable Nested Virtualization (if needed)

If running a hypervisor inside the VM:

```bash
# Check if enabled:
cat /sys/module/kvm_intel/parameters/nested  # Intel
cat /sys/module/kvm_amd/parameters/nested    # AMD

# Enable nested virtualization:
echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm.conf
# Then reload the module
```

### Solution 6: Check CPU Feature Flags

Ensure the guest sees the correct CPU features:

```bash
# In QEMU, explicitly enable required features:
qemu-system-x86_64 -cpu host,+vmx,+svm ...
```

## Recommended Approach

For most cases, try the solutions in this order:

1. **First:** Try `idle=poll` kernel parameter in the guest (quickest, no host changes)
2. **Second:** Add `ignore_msrs=1` to KVM module (if first doesn't work)
3. **Third:** Update QEMU/KVM and guest kernel to latest versions
4. **Fourth:** Review and adjust QEMU CPU configuration

## References

- [KVM Documentation](https://www.kernel.org/doc/html/latest/virt/kvm/)
- [QEMU Documentation](https://www.qemu.org/documentation/)
- [Intel SDM Volume 4 - Model-Specific Registers](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)
- Linux Kernel source: `arch/x86/kernel/process.c` (idle functions)
