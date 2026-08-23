# vmm

**The decisions a virtual machine monitor makes, written in Kotoba and
compiled by amu.** Where a foreign guest's kernel image, devicetree and
initrd go; what an MMIO exit is; which PSCI call an HVC is. The monitor asks;
this answers.

The name is the plain industry term, and it is doing work here, because four
things in this workspace have `vm` in the name and only one of them is this:

| | what it is |
|---|---|
| **`kotoba-lang/vmm`** (here) | machine virtualization decisions -- guests are *other operating systems* |
| `kotoba-lang/kotoba-vm` | the IPLD **actor invocation kernel** for an app-chain. Runs committed messages, not kernels |
| `kotoba-lang/kototama` | the **Wasm** host. Guests are components we compiled |
| `aiueos.hvt` | the **mechanism**: `/dev/kvm` ioctls, guest RAM, the vcpu loop |

## The split

`aiueos.hvt` already owns the monitor's mechanism and has since ADR-0014 V1:
it opens `/dev/kvm` through `java.lang.foreign`, maps guest RAM, runs the
vcpu and services exits, with a virtio-mmio device model and a virtqueue data
path. What it hosts today are its own guests -- five small Kotoba programs
compiled to bare-metal AArch64 ELF.

Hosting a *foreign* environment adds no mechanism. It adds decisions: this
image is or is not an arm64 Linux kernel; the blob goes here and not there;
this address is virtio device 3 register 0x50; this function id is
SYSTEM_OFF. Those are exactly the things aiueos ADR-0015 keeps out of C, and
they are what lives here -- as `.kotoba`, compiled by amu, shipped as KIR.

```
kotoba/*_core.kotoba          the authority. Decisions, no I/O
  -> amu (clojure -M:test:gen)
resources/vmm/oracle/*.kir.edn  what ships
  -> kotoba.kir
src/vmm/*.cljc                the call path. Threads values, decides nothing
  -> aiueos.hvt               the mechanism. ioctl, mmap, KVM_RUN
```

There is no second copy of any decision: the host namespaces execute the KIR
rather than reimplement it, and a gate refuses a shipped artifact that no
longer matches its source.

## Where it actually is

| | status |
|---|---|
| arm64 Linux boot protocol -- Image header admission, 2 MiB base, DTB and initrd placement, `x0`/`pc` | **decided here**, 35 assertions |
| MMIO exit classification -- PL011 and 32 virtio-mmio devices, offsets, access size, window straddling | **decided here**, 23 assertions |
| PSCI function dispatch | **decided here** |
| Devicetree (FDT) construction | **not here yet** -- the byte layout is the next core; nothing builds a blob today |
| Virtqueue descriptor admission | **not here** -- `aiueos.virtio` owns it. Duplicating it would create the second copy this repository exists to avoid |
| Booting an actual Linux kernel | **not done.** No call site in `aiueos.hvt` uses these cores yet, and nothing in this repository has been run against `/dev/kvm` |
| x86_64, macOS/HVF | **not started.** ADR-0014 defers HVF to V2 behind the `com.apple.security.hypervisor` entitlement question |

Nothing here has hosted an operating system. What it has is the first part of
the answer, written where it can be checked, with the gates that would catch
it being wrong.

## Use

```clojure
(require '[vmm.arm64-boot :as boot] '[vmm.exit :as exit])

(boot/plan {:magic 0x644D5241 :image-size (* 30 1024 1024) :text-offset 0
            :flags 0 :ram-base 0x40000000 :ram-size (* 512 1024 1024)
            :dtb-size 65536 :initrd-size (* 4 1024 1024)})
;; => {:ok? true :kernel-gpa 0x40000000 :dtb-gpa 0x41E00000
;;     :initrd-gpa 0x42000000 :x0 0x41E00000 :x1 0 :x2 0 :x3 0 :pc 0x40000000 ...}

(boot/plan {... :magic 0})
;; => {:ok? false :reason "arm64-boot/bad-magic"}

(exit/classify-mmio {:gpa 0x0A000650 :len 4})
;; => {:ok? true :device :virtio :index 3 :offset 0x50}

(exit/psci-action 0x84000008)   ;; => {:ok? true :action :system-off}
(exit/psci-action 0x84000010)   ;; => {:ok? false :reason "psci/not-supported"}
```

Errors are values. Nothing here throws to refuse -- root CLAUDE.md's
`[:result T E]` rule -- and every refusal names itself with a literal the
tests pin, so renaming a reason upstream fails a test instead of quietly
widening what is admitted.

## Build and check

```bash
clojure -M:test          # 12 tests, 64 assertions
clojure -M:test:gen      # regenerate resources/vmm/oracle/*.kir.edn from kotoba/
```

`vmm.kir-freshness-test` recompiles every `kotoba/*_core.kotoba` and compares
it to the shipped artifact. Without it the source would be documentation:
edit it, forget to regenerate, and the host keeps executing the old decision
with every other test green -- because every other test runs the artifact.

Both gates were checked in the failing direction before landing: renaming one
reason literal fails the freshness gate while it is unregenerated, and fails
exactly the two assertions that pin that literal once it is.

## What decides what

- `kotoba/arm64_boot_core.kotoba` -- `Documentation/arch/arm64/booting.rst`
  as staged admissions: header, kernel, dtb, initrd. Split that way because
  the ABI admits at most five parameters, which turned out to be the better
  shape anyway: each stage names only the facts it needs.
- `kotoba/exit_core.kotoba` -- the QEMU `virt` address map as data, because
  that is the map the devicetree we will hand a guest describes. Nothing
  probes hardware.

## Not this repository's job

Type-1 hypervisor. ADR-0014 made that a firm non-goal and this does not
reopen it: the facility is the host's (KVM, later HVF), the monitor is
userspace, and what is here is the part of the monitor that thinks.
