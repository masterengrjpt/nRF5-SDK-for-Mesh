# nRF5 SDK for Mesh Bootloader

*The mesh bootloader for over-the-mesh Direct Firmware Updates*

The mesh bootloader enables mesh devices to update their application, SoftDevice or bootloader
across a mesh network. The bootloader is capable of relaying while receiving, allowing all
devices in the mesh network to receive the update at the same time.

The bootloader supports DFU transfers occurring while the application is running, significantly
reducing application downtime during updates.

## Key parameters

The bootloader is limited to 22 kilobytes flash and 768 bytes RAM. In addition to the 22 kilobytes
flash, the bootloader reserves the two last pages of the flash. This means that the total size of
the bootloader is 32 kilobytes for the nRF52-series and 24 kilobytes for the nRF51-series.
The actual size of the bootloader depends on whether it is compiled with the serial (UART)
transport in addition to toolchain and optimization settings.

## Compatibility

The mesh bootloader supports all devices in the nRF51-series as well as nRF52832.

## Requirements
See [The toolchain document](@ref md_doc_getting_started_how_to_toolchain) for the required tools
for building and using the bootloader. Pre-compiled variants of the bootloader are available in
the `bin/` folder. Building the bootloader is only supported with the CMake build system and can
be enabled with the `BUILD_BOOTLOADER` CMake option. Be aware that the bootloader must be built
with `CMAKE_BUILD_TYPE` equal to `MinSizeRel` for it to fit within its flash size requirement.

## Usage

To get started with the Mesh-DFU, see the
[DFU Quick start guide](@ref md_doc_getting_started_dfu_quick_start). It is recommended that all
new users go through this guide, and use the steps as a basis for their DFU procedure.

The bootloader supports both serial and over the air transfers. The intended
main usage scenario is to have a single device in a network connected to a host
over a serial connection, controlled via the mesh-version of [pc-nrfutil](https://github.com/NordicSemiconductor/pc-nrfutil). From this
device, all devices in the network may receive any firmware updates over the
mesh. All update packets are flooded across the mesh, and all devices in the
network are updated at the same time.

## DFU Signing

The mesh-bootloader allows for signed DFU transfers, using 256 bit ECDSA. Signing DFUs prevents
attackers from hijacking a device by flashing their own firmware to it. While not enforced, it
is strongly recommend to give your device a signing key before deploying a product, as this
allows for better security, without any significant overhead.

The DFU transfers are signed by a private key, generated by the Nordic Semiconductor pc-nrfutil
tool (see the quick start guide for details), and each device has a copy of the public key to
verify a signature added to the end of signed firmwares. Under no circumstance should the private
signing key be shared with a third party or be present in the device firmware. You may choose to
use a single signing key for each application, or use the same key for all applications.

## Side-by-side DFU

The mesh bootloader is capable of receiving and relaying DFU transfers while the application is
running. When a transfer has finished, the DFU module goes idle, and notifies the application of
the new firmware bank. The application may choose to inspect the firmware, alter it, or do some
additional validation, then flash it at any time. After flashing a banked firmware, the old
firmware is no longer available on the device, and any rollbacks have to be executed as a new
update.

The bootloader is also capable of operating without an application, as a recovery mechanism from
a faulty application.

### The shared DFU module

Since the DFU operation is the same whether the device is operating in application or bootloader
mode, the DFU module in the bootloader has been separated from the rest of it, and is made
callable from the application. This sharing is implemented in a similar manner to the Softdevice,
where the application is expected to reserve some RAM for it, without linking it.
Access to the DFU module goes through a single entry point, which is located by the application side
framework upon initialization.

## Limitations

A couple of limitations to the bootloader applies:

- To update the bootloader, some space has to be allocated in the application section for double
banking the firmware, as it cannot overwrite itself while it's running. The bootloader will use
the end of the application area to achieve this, meaning that any application data present in that
area will be invalidated. The application has to be retransfered afterwards to repair itself.

- The bootloader is not able to host a DFU transfer on its own, it depends on an external entity
initiating and controlling the transfer.

- The firmware goes across the mesh network unencrypted, so at no point should the firmware contain
any keys or secrets in plaintext.

- The bootloader can only validate incoming transfers after they receive every segment of the
firmware. Effectively, a malicious third party could emulate a valid transfer, causing target
devices to delete their current firmware to make room for the new one, only to discover that the
transfer was a hoax during the signature validation at the end. However, the firmware in the faked
transfer would never be executed, as long as the devices require a signed images.

- Currently, there is no way to change the device-info page during run-time.