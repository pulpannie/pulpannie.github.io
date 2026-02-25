---
layout: post
title: "notes on CPU Virtualization with VT-x"
date: 2026-02-09 10:00:00 +0900
categories: [hardware]
tags: [hardware, virtualization, OS]
---
These are my notes on *Hardware and Software Support for Virtualization*, Chapter 4, x86-64: CPU Virtualization with VT-x.

# The VT-x Architecture
The VT-x architecture aimed to not change the semantics of the individual instructions of the ISA. Therefore, it instead **extends** the architecture such that the architecturally visible state of the processor is *duplicated* into a new mode of execution: **root mode**. Hypervisors and host operating systems run in root mode, while a VM executes in non-root mode.
The processor is at any point in time either in root mode or non-root mode. The transitions are atomic, which is different from the conventional implementation of a context switch by an Operating System.


## Transitions between root and non-root modes


Upon transition from non-root mode to root mode (called #vmexit), the state of the virtual machine is stored in a structure in physical memory called the **Virtual Machine Control Structure (VMCS)**. The vmresume instruction transitions back from root mode to non-root mode and loads the VMCS state into the current processor state. The architecture does not define the layout or "correct" state of VMCS. It is up to the implementation of the hypervisor to vmread and vmwrite portions of the guest state into VMCS. (AMD, on the other hand, architecturally defines a virtual machine control block, VMCB).

Some of the causes for vmexit include:
- The guest attempts to execute a root-mode-privileged instruction
- hypercall via vmcall instructions
- exceptions (including regular page faults, or EPT violations which are a subset of page faults)
- external interrupts that occur while the CPU is executing in non-root mode

# KVM - A Hypervisor for VT-x
KVM(Kernel Virtual Machine) is an open-source Linux kernel module which implements a type-2 hypervisor. It was designed assuming the existence of hardware support for virtualization.

It relies on QEMU, an open-source complete machine simulator, to emulate I/O. Together with KVM, QEMU is responsible for the userspace implementation of all I/O front-end device emulation, while the Linux host is responsible for the I/O backend (via normal system calls), and the KVM kernel module is responsible to multiplex the CPU and MMU of the processor. 

More specifically, the KVM kernel module only handles the basic CPU and platform emulation issues. This includes, CPU emulation, memory management and MMU virtualization, interrupt virtualization, and some chipset emulation (APIC, IOAPIC, etc.). But it excludes all I/O device emulation.

In the simplest form, the KVM is designed with the following principles:
1. Configure the hardware appropriately
2. Let the VM execute directly on the hardware
3. Upon the first trap or interrupt, the hypervisor then regains control, and "just" emulates the trapping instruction according to the semantic

