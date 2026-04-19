# Exploration of Trusted Execution Environment (TEE)

**By Amith K S**

---

## What is TEE?

A trusted execution environment (TEE) is a secure area of a main processor. It helps the code and data loaded inside it be protected with respect to confidentiality and integrity. Data confidentiality prevents unauthorized entities from outside the TEE from reading data, while code integrity prevents code in the TEE from being replaced or modified by unauthorized entities.

## How It Works

Done by implementing unique, immutable, and confidential architectural security, which offers hardware-based memory encryption that isolates specific application code and data in memory. This allows user-level code to allocate private regions of memory, called enclaves, which are designed to be protected from processes running at higher privilege levels.

## Additional Details

- A TEE typically combines a hardware isolation mechanism with a secure operating system, protecting trusted applications from user-installed apps and the main operating system.
- While groups like GlobalPlatform and FIDO mandate hardware isolation for TEEs, others like EMVCo accept both hardware and software-based solutions.
- Only trusted applications running inside the TEE have full access to the device's main processor, memory, and peripherals, and they use cryptography to protect themselves from one another.
- To prevent software simulation, manufacturers embed unalterable private keys directly into the chip during manufacturing, often using one-time programmable memory like eFuses.
- The hardware checks a public key provided at runtime against a hash embedded in the chip; if they match, it verifies the digital signature of the trusted firmware to ensure only vendor-approved software accesses privileged features.
- Trusted applications prove their integrity to external servers using cryptographic protocols, a process that cannot be faked by virtual machines or simulators because it requires the physical keys baked into the hardware.
- Extracting hardware keys is prohibitively expensive and requires advanced equipment, as keys are usually unique to each specific chip and tampering often destroys them.
- In consumer electronics, manufacturers intentionally design TEEs to retain control over the hardware keys, allowing them to monetize the hardware, enforce Digital Rights Management (DRM), and restrict device use cases.

---

## Prominent TEE Implementations and Architectures

### 1. Cloud and Virtualization TEEs (Confidential Computing)

Designed to protect entire virtual machines or workloads in the cloud, even from the cloud provider themselves.

- **AMD:** Secure Encrypted Virtualization (SEV) and Secure Nested Paging (SNP)
- **Intel:** Trust Domain Extensions (TDX)
- **ARM:** Confidential Compute Architecture (CCA)
- **IBM:** Secure Execution

### 2. Application-Level Enclaves

Designed to protect a specific, small piece of code or an app running on a normal computer, isolating it from the rest of the operating system.

- **Intel:** Software Guard Extensions (SGX)
- **RISC-V:** Keystone (an open-source TEE framework)

### 3. Mobile and Embedded TEEs

Physically partition the processor into a "Secure World" and a "Normal World." This is what powers mobile payments and fingerprint sensors on smartphones.

- **ARM:** TrustZone

### 4. Dedicated Security Coprocessors

Actual separate, dedicated microprocessors built right into the motherboard or CPU die that run their own hidden operating systems to manage platform security.

- **AMD:** Platform Security Processor (PSP)
- **Intel:** Management Engine (ME)

---

## TEE vs Sandboxing

- A **Sandbox** protects the *system* from a malicious *application*.
- A **TEE** protects the *application* from a malicious *system*.

| Feature | Sandboxing | Trusted Execution Environment (TEE) |
|---|---|---|
| **Primary Goal** | Protect the **System (OS)** from a malicious application | Protect the **Application (or Data)** from a malicious system |
| **Implementation Level** | Software (Enforced by the Operating System or Hypervisor) | Hardware (Enforced directly by the CPU and silicon) |
| **Trust Model** | Trusts the OS entirely; assumes the application is dangerous or untrusted | "Zero Trust" for software; assumes the OS is compromised. Trusts only the physical hardware |
| **Isolation Mechanism** | Virtual/Logical (Software rules blocking access to files, ports, or memory) | Physical/Cryptographic (Hardware-encrypted memory regions and strict access control) |
| **Primary Weakness** | Fails completely if the OS kernel is compromised (e.g., if a hacker gains root access) | Resilient to OS compromise, but vulnerable to complex hardware-level exploits (e.g., side-channel attacks) |

---

# Exploration Report: ARM TrustZone, OP-TEE Architecture, and Simulator Deployment

**By Amith K S**

---

## ARM TrustZone

TrustZone is the name of the Security architecture in the Arm A-profile architecture. First introduced in Armv6K, TrustZone is also supported in Armv7-A and Armv8-A. TrustZone provides two execution environments with system-wide hardware enforced isolation between them.

![ARM TrustZone Architecture Diagram](arm_trustzone_architecture.png)

The **Normal World** runs a rich software stack. This software stack typically includes a large application set, a complex operating system like Linux, and possibly a hypervisor. Such software stacks are large and complex. While efforts can be made to secure them, the size of the attack surface means that they are more vulnerable to attack.

The **Trusted World** runs a smaller and simpler software stack, which is referred to as a Trusted Execution Environment (TEE). Typically, a TEE includes several Trusted services that are hosted by a lightweight kernel. The Trusted services provide functionality like key management. This software stack has a considerably smaller attack surface, which helps reduce vulnerability to attack.

## Security States

To change Security state, in either direction, execution must pass through EL3.

![Security States Transition Diagram (EL3)](security_states_el3.png)

## Virtual Address Space

### The Solution: Parallel Maps

TrustZone gives the processor completely separate, physically isolated maps depending on what state the processor is currently in.

- **NS.EL1:0x8000 (Non-Secure Map):** Your standard Linux OS is running. It asks to read address `0x8000`. The processor knows it is in the Non-Secure state, so it grabs the **Non-Secure Map**. The map says, "Put them in physical memory slot A."
- **S.EL1:0x8000 (Secure Map):** The processor flips into the Secure State to run OP-TEE. OP-TEE asks to read address `0x8000`. The processor grabs the **Secure Map**. Because this is a totally different map, it points to physical memory slot B.

## Physical Address Space

While in Non-secure state, virtual addresses always translate to Non-secure physical addresses. This means that software in Non-secure state can only see Non-secure resources, but can never see Secure resources.

![Non-Secure Physical Address Space Diagram](non_secure_physical_address_space.png)

While in Secure state, software can access both the Secure and Non-secure physical address spaces. The NS bit in the translation table entries controls which physical address space a block or page of virtual memory translates to.

![Secure Physical Address Space Diagram](secure_physical_address_space.png)

---

## OP-TEE

OP-TEE is a Trusted Execution Environment (TEE) designed as companion to a non-secure Linux kernel running on Arm Cortex-A cores using the TrustZone technology.

The non-secure OS is referred to as the Rich Execution Environment (REE) in TEE specifications. It is typically a Linux OS flavor as a GNU/Linux distribution or the AOSP.

OP-TEE is designed primarily to rely on the Arm TrustZone technology as the underlying hardware isolation mechanism. However, it has been structured to be compatible with any isolation technology suitable for the TEE concept and goals, such as running as a virtual machine or on a dedicated CPU.

---

## References

- https://en.wikipedia.org/wiki/Trusted_execution_environment
- https://confidentialcomputing.io/about

