# Overview
We want to allow VM-owners to explicitly set the video device type when needed.

## Motivation
Currently, the default video device type for AMD64 in KubeVirt is **VGA** or **Bochs** in some cases.
Using `vga` video type can limit the video functionality available to virtual machines.
For example, VGA restricts display resolution settings for Windows VMs and may not support modern workloads effectively.
However, it is necessary to maintain compatibility for older guests that depend on VGA (e.g., RHEL 5/6).
To address this, we propose introducing a new configuration field to allow users to set the video device type.

## Goals
The VM spec template resource should have the ability to set the video device type when needed.

## Non Goals
Change the defaulting logic

## Definition of Users
VM-owners who require specific configurations for VMIs in their environments.

## User Stories
- As a VM owner, I want to be able to set a better video device like `virtio` instead of the standard `vga`.

## Repos
- [KubeVirt](https://github.com/kubevirt/kubevirt)

# Design

## VM-Level Configuration Field

### Description
Add a `Video` struct under `spec.template.spec.domain.devices` in the VM template schema.
This struct will include a single field, `type`, to specify the desired video device type.

The current behavior relies on the `autoattachGraphicsDevice` field, where:
1. If `autoattachGraphicsDevice` is not specified or is set to `true`, the default logic determines the video device type (VGA or Bochs for AMD64).
2. If `autoattachGraphicsDevice` is set to `false`, no graphics or video devices are attached.

The proposed change ensures that users can explicitly set the `type` for the video device, and the logic will respect this configuration.
The architecture-specific logic will continue to manage defaults for unspecified configurations.

### Implementation Logic
The following logic will be added to the `addGraphicsDevice` function:
1. If `Video` is explicitly specified in the VM spec, the video type from the spec is used.
2. If `autoattachGraphicsDevice` is not specified or is `true`:
   - Use Bochs for EFI guests on AMD64.
   - Use VGA for BIOS guests on AMD64.
3. If `autoattachGraphicsDevice` is set to `false`, no graphics or video devices are attached.

same hierarchy for the other arches

**Pros:**
- Granular Control: Enables precise configuration for each VM, avoiding cluster-wide changes that could affect unrelated workloads.
- Default Logic: Ensures reasonable defaults based on architecture while allowing user overrides for legacy compatibility.
- Scalability: Provides flexibility for administrators and VM owners in mixed environments with diverse requirements.

**Cons:**
- API Complexity: Requires schema changes to the VM template API, introducing additional fields that need maintenance and testing.
- Administrative Effort: Per-VM configuration may lead to higher administrative overhead if many VMs require toggling.

## API Examples

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm
spec:
  template:
    spec:
      domain:
        devices:
          video:
            type: virtio
          disks:
```

## Scalability
Overhead of the virt-launcher/qemu may be impacted.

## Update/Rollback Compatibility
Currently, the virt-launcher arch-converter manages the default video type.
We only provide a field for the user to specify explicitly if they want something else like virtio, so backward compatibility isn't affected.

## Functional Testing Approach
* Create a VM with video.type set to `virtio`, Validate that the domain contains virtio as video device type, and expect launch successfully.

# Implementation Phases
1. **Phase 1: Introduce New Field to Allow Users to Explicitly Set Their Desired Video Type**
    - Add logic in the virt launcher to check the field's existence before setting the default video device (virtio/vga).
    - Write unit and functional tests for the new behavior.

2. **Phase 2 Documentation**
    - Update KubeVirt documentation to include examples of configuring the video device type.
