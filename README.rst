=========================
QEMU PPC Screamer (mac99)
=========================

This fork focuses on running classic Mac OS 9 on the New World PowerMac
``mac99`` machine with Screamer audio. It adds:

- Support for loading SheepShaver-style New World ROMs ("Mac OS ROM").
- Reliable reset vector at ``0xFFF00100`` for raw ROMs larger than 1 MiB.
- A switch to force PIO mode on MacIO IDE (avoids OS 9 install stalls).
- Default Apple-like HDD model string (improves Drive Setup compatibility).


Build (macOS Universal)
=======================

Option A: Use GitHub Actions (recommended)
-----------------------------------------

Push to this repo. The workflow builds x86_64 and arm64 binaries and
publishes a universal2 CLI binary (``qemu-system-ppc-universal``) under the
"Continuous" Release.


Option B: Local universal build
-------------------------------

You can build per-arch binaries and lipo them together.

Requirements (Homebrew):

.. code-block:: shell

  brew install meson ninja pkg-config glib pixman

Build arm64 binary (Apple Silicon):

.. code-block:: shell

  ./configure --target-list=ppc-softmmu --disable-werror --disable-fuse --disable-fuse-lseek
  ninja -C build qemu-system-ppc
  mv build/qemu-system-ppc qemu-system-ppc-arm64

Build x86_64 binary (Rosetta toolchain):

Install x86_64 Homebrew (in */usr/local*) and ensure its pkg-config/libs are on
PATH under Rosetta. Then:

.. code-block:: shell

  env CFLAGS="-arch x86_64" LDFLAGS="-arch x86_64" \
    ./configure --target-list=ppc-softmmu --disable-werror --disable-fuse --disable-fuse-lseek
  ninja -C build qemu-system-ppc
  mv build/qemu-system-ppc qemu-system-ppc-x86_64

Create universal2 binary:

.. code-block:: shell

  lipo -create qemu-system-ppc-x86_64 qemu-system-ppc-arm64 -output qemu-system-ppc-universal
  chmod +x qemu-system-ppc-universal


Install Mac OS 9 (SheepShaver ROM)
==================================

Inputs:

- New World ROM file (SheepShaver-style), e.g. ``ppc.rom`` (2–4 MiB).
- Mac OS 9.0.4 ISO, e.g. ``9.iso``.
- New qcow2 disk image, e.g. ``9.qcow2``.

Create disk image:

.. code-block:: shell

  qemu-img create -f qcow2 9.qcow2 4G

Boot installer (PIO mode recommended during install):

.. code-block:: shell

  ./qemu-system-ppc-universal \
    -M mac99,via=cuda -cpu 7400 -m 256 \
    -bios ppc.rom \
    -global macio-ide.use-dma=false \
    -drive file=9.qcow2,format=qcow2,media=disk \
    -cdrom 9.iso -boot d

Notes:

- ``-global macio-ide.use-dma=false`` forces PIO mode and avoids rare
  stalls at "Updating Apple Hard Disk Drivers" and early large writes.
- Screamer audio is enabled via MacIO; default macOS audio backend is used.
- Input via ADB (CUDA) works out of the box; you may add USB input devices
  if preferred (e.g. ``-device usb-kbd -device usb-mouse``).


Boot from the installed disk
============================

Once the installer finishes and you reboot, boot from the hard disk:

.. code-block:: shell

  ./qemu-system-ppc-universal \
    -M mac99,via=cuda -cpu 7400 -m 256 \
    -bios ppc.rom \
    -global macio-ide.use-dma=false \
    -drive file=9.qcow2,format=qcow2,media=disk \
    -boot c

You may try re-enabling IDE DMA later by removing the ``-global`` setting.


Troubleshooting
===============

- ROM mapping: this tree auto-detects raw (non-ELF) ROMs and maps them at the
  top of 32-bit memory, ensuring the reset vector at ``0xFFF00100`` works.
- Video: for the best compatibility with Mac OS 9 using an Apple ROM, consider
  a PCI GPU with a Mac NDRV ROM. The default VGA may work, but acceleration and
  driver support can vary.


License
=======

This repository is based on QEMU and is released under the terms of the GNU
General Public License, version 2. See LICENSE for details.
