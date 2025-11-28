# KVM Internal Error Analysis

This document analyzes KVM internal errors (Suberror 3: KVM_INTERNAL_ERROR_EMULATION) and provides solutions.

---

# Case 1: Linux Guest KVM Error Analysis

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

## Recommended Approach for Linux Guests

For most cases, try the solutions in this order:

1. **First:** Try `idle=poll` kernel parameter in the guest (quickest, no host changes)
2. **Second:** Add `ignore_msrs=1` to KVM module (if first doesn't work)
3. **Third:** Update QEMU/KVM and guest kernel to latest versions
4. **Fourth:** Review and adjust QEMU CPU configuration

---

# Case 2: Windows Guest KVM Error Analysis

## Error Summary

```
KVM internal error. Suberror: 3
extra data[0]: 0x000000008000002f
extra data[1]: 0x0000000080000001
extra data[2]: 0x0000000080000d82
extra data[3]: 0x0000000080000038
```

## Register State at Error

| Register | Value              | Interpretation                        |
|----------|--------------------|-----------------------------------------|
| RAX      | 0000000000000000   | Zero                                   |
| RBX      | ffffd00173a81180   | Kernel pointer (Windows KPCR area)     |
| RCX      | 0000000000000000   | Zero                                   |
| RDX      | 0000000000000000   | Zero                                   |
| RSI      | 0000000000000000   | Zero                                   |
| RDI      | 0000000000000046   | 70 decimal                             |
| RBP      | ffffd00173aae010   | Stack base pointer                     |
| RSP      | ffffd00173aadf88   | Stack pointer                          |
| RIP      | fffff804015a2      | Windows kernel code (truncated in log) |
| RFL      | 00000096           | Flags: [--S-AP-]                        |
| CPL      | 0                  | Ring 0 (kernel mode)                   |
| CR2      | 0000000000000030   | Page fault address (null ptr area)     |

## Operating System Detection

Based on the segment registers, this is a **Windows kernel**:

| Segment | Value | Description                              |
|---------|-------|------------------------------------------|
| CS      | 0010  | 64-bit kernel code segment (DPL=0)       |
| SS      | 0018  | Kernel stack segment (DPL=0)             |
| DS/ES   | 002b  | User mode data segment (DPL=3)           |
| FS      | 0053  | Windows TEB pointer (base=7fe87000)      |
| GS      | 002b  | Kernel KPCR/PRCB (base=ffffd00173a81000) |

## Extra Data Analysis

- **extra data[0]: 0x8000002f**
  - Bit 31 set = instruction recognized as valid
  - Low byte 0x2f = 47 = **EPT violation** (VM Exit Reason)

- **extra data[1]: 0x80000001**
  - EPT violation qualification flags

- **extra data[2]: 0x80000d82**
  - EPT violation related data

- **extra data[3]: 0x80000038**
  - Additional exit context

## Instruction Analysis

### Machine Code at RIP

```
Code=48 8b 40 48 48 85 c0 74 e5 48 ff e0 90 90 90 90 90 90 90 90 <48> 83 ec 28 ...
```

The `<48>` marker indicates the instruction pointer position.

### Decoded Instructions

**Before RIP (preceding code):**

| Bytes        | Instruction          | Description                    |
|--------------|----------------------|--------------------------------|
| 48 8b 40 48  | mov rax, [rax+0x48]  | Load pointer from structure    |
| 48 85 c0     | test rax, rax        | Check if null                  |
| 74 e5        | jz short -0x1b       | Jump if zero                   |
| 48 ff e0     | jmp rax              | Indirect jump through RAX      |
| 90 90 90...  | nop (padding)        | Alignment padding              |

**At RIP (current instruction):**

| Bytes        | Instruction          | Description                    |
|--------------|----------------------|--------------------------------|
| **48 83 ec 28** | **sub rsp, 0x28** | **Allocate 40 bytes on stack** |
| 48 85 c9     | test rcx, rcx        | Check if RCX is null           |
| 75 0b        | jnz short +0xb       | Jump if not zero               |
| 48 83 c4 28  | add rsp, 0x28        | Deallocate stack               |
| 48 ff 25 ... | jmp qword [rip+...]  | Indirect jump via RIP          |

### Identified Pattern

This is a **Windows kernel function prologue**:
- `sub rsp, 0x28` allocates 40 bytes (0x28 hex = 40 decimal): 32 bytes shadow space + 8 bytes alignment
- This is the entry point of a kernel function

## Root Cause

The instruction `sub rsp, 0x28` (stack allocation) is a normal instruction that should not fail.

### Primary Cause: EPT Violation

The extra data suggests an **Extended Page Tables (EPT) violation**:

1. **Memory Mapping Issue**: The stack page or code page at RIP is not properly mapped in EPT
2. **CR2=0x30**: Indicates a nearby null pointer dereference or invalid memory access
3. **Windows-specific**: Windows kernel may be accessing unmapped virtualization-related memory

### Contributing Factors

1. **Hyper-V Conflicts**: Windows may be trying to use Hyper-V features that conflict with KVM
2. **VBS/HVCI**: Virtualization-Based Security or Hypervisor-enforced Code Integrity
3. **Memory Configuration**: EPT not properly configured for all guest memory regions

## Solutions for Windows Guest

### Solution 1: Enable Hyper-V Enlightenments in QEMU

```bash
qemu-system-x86_64 \
  -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,hv_crash,hv_reset,hv_vpindex,hv_runtime,hv_synic,hv_stimer \
  ...
```

These enlightenments help Windows run better under KVM.

### Solution 2: Disable Conflicting Windows Features

In the Windows guest, disable these features:

```cmd
REM Disable Hyper-V
bcdedit /set hypervisorlaunchtype off

REM Disable Virtualization Based Security (VBS)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f

REM Disable Credential Guard
reg add "HKLM\SYSTEM\CurrentControlSet\Control\LSA" /v LsaCfgFlags /t REG_DWORD /d 0 /f

REM Reboot required
shutdown /r /t 0
```

### Solution 3: Use Proper CPU Model

```bash
# Use host CPU passthrough
qemu-system-x86_64 -cpu host ...

# Or use a specific CPU model known to work well
qemu-system-x86_64 -cpu Skylake-Client-v3 ...
```

### Solution 4: Configure KVM Module

```bash
# Ignore unknown MSRs (same as Linux guest)
echo "options kvm ignore_msrs=1" | sudo tee /etc/modprobe.d/kvm.conf

# Reload KVM module
sudo modprobe -r kvm_intel && sudo modprobe kvm_intel
```

### Solution 5: Check Memory Configuration

```bash
# Use a proper memory backend
qemu-system-x86_64 \
  -m 4G \
  -object memory-backend-memfd,id=mem,size=4G,share=on \
  -numa node,memdev=mem \
  ...
```

### Solution 6: Update QEMU/KVM

Ensure you're running the latest versions:

```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade qemu-system-x86 libvirt-daemon-system

# RHEL/CentOS/Fedora
sudo dnf update qemu-kvm libvirt
```

## Recommended Approach for Windows Guests

1. **First:** Disable Hyper-V and VBS in the Windows guest
2. **Second:** Add Hyper-V enlightenments to QEMU command line
3. **Third:** Configure `ignore_msrs=1` in KVM module
4. **Fourth:** Ensure using `-cpu host` or appropriate CPU model
5. **Fifth:** Update all virtualization software to latest versions

---

## References

- [KVM Documentation](https://www.kernel.org/doc/html/latest/virt/kvm/)
- [QEMU Documentation](https://www.qemu.org/documentation/)
- [Intel SDM Volume 4 - Model-Specific Registers](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)
- Linux Kernel source: `arch/x86/kernel/process.c` (idle functions)
- [Windows Hyper-V Enlightenments in QEMU](https://www.qemu.org/docs/master/system/i386/hyperv.html)
