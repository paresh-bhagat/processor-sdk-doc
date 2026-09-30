.. _how-to-use-pipewire:

How to Use PipeWire
===================

Abstract
--------

PipeWire is a modern, low-latency multimedia framework that has become the standard for audio and video handling in Linux applications, providing graph-based processing with real-time capabilities using unified architecture. This application report demonstrates the enablement of PipeWire on Texas Instruments' Sitara family of devices, which feature dual/quad-core Arm Cortex-A53 processors. PipeWire’s multi-process architecture allows multiple applications to seamlessly share multimedia content without conflicts or resource contention.
The document guides audio/video product developers through building Yocto image, setting up PipeWire, providing setup, configurations, performance benchmarks to leverage the platform's capabilities for multimedia processing.
Target Applications:
    • Professional audio equipment
    • Smart speakers and soundbars
    • Video conferencing systems
    • Automotive infotainment
    • Industrial HMI with multimedia
    • IoT devices with audio/video capabilities

Table of Contents
-----------------

#. Introduction
#. Audio System Architecture
#. Build PipeWire Image via Yocto
#. Start and use PipeWire
#. Configuration Details
#. Performance Benchmarks
#. References

Introduction
------------

The Linux audio landscape has historically been fragmented, with different subsystems serving different needs:
    • ALSA for low-level hardware access
    • PulseAudio for desktop audio
    • JACK for professional audio
    • GStreamer for multimedia pipelines

This fragmentation creates challenges for developers who must support multiple APIs and manage complex routing between different audio subsystems.
PipeWire is a modern multimedia framework and graph-based processing architecture designed to revolutionize audio and video handling in Linux systems. It serves as a unified solution that consolidates the functionality of PulseAudio, JACK, and other multimedia frameworks into a single, efficient processing engine.

1.1 Key Highlights
^^^^^^^^^^^^^^^^^^

    • Unified Multimedia Framework: Single solution for professional audio (JACK), consumer audio (PulseAudio), and video processing.
    • Low Latency Performance.
    • Security Model: Built-in sandboxing support for containerized applications.
    • Multi-process architecture to let applications share multimedia content.
    • Real-time multimedia processing on audio and video.

1.2 Basic Concepts
^^^^^^^^^^^^^^^^^^

1.2.1 PipeWire Server
~~~~~~~~~~~~~~~~~~~~~
The server is the core daemon that manages a graph-based multimedia processing engine. It handles the creation and execution of the media graph where audio, video, or MIDI data flows between different processing components.

1.2.2 PipeWire Clients
~~~~~~~~~~~~~~~~~~~~~~
Clients are applications that connect to the PipeWire server to produce or consume media streams. They create nodes in the media graph to send or receive audio or video data.

1.2.3 Session Manager
~~~~~~~~~~~~~~~~~~~~~
PipeWire does not handle device routing or policy decisions by itself. These tasks are managed by a session manager, which monitors devices and automatically connects streams. In this document, WirePlumber is used as the session manager.

1.2.4 Nodes, Ports, and Links
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The PipeWire processing graph consists of nodes, ports, and links. A node represents a processing element, ports act as input or output interfaces, and links connect ports between nodes to allow media data to flow through the graph.

1.3 PipeWire Main Components
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    • A PipeWire Daemon that implements the IPC and graph processing.
    • An example PipeWire Session Manager that manages objects in the PipeWire Daemon.
    • A set of Programs to introspect and use the PipeWire Daemon.
    • A PipeWire Library to develop PipeWire applications and plugins (tutorial).
    • The SPA (Simple Plugin API) used by both the PipeWire Daemon and in the PipeWire Library.

Audio System Architecture
-------------------------

.. figure:: ../images/linux_audio_stack_with_pipewire.png
    :align: center
    :width: 600

Linux audio stack with PipeWire

    • In Linux audio system using, applications such as media players, browsers, VoIP clients, or tools like GStreamer send and receive audio through APIs like the native PipeWire API or compatibility layers for Jack, Pulseaudio or (ALSA).
    • These audio streams are handled by the PipeWire media server, which builds a processing graph to mix, route, and schedule audio from multiple applications.
    • A session manager such as WirePlumber applies policies like device selection and automatic routing.
    • PipeWire accesses actual audio devices using its SPA device plugins (for example the ALSA plugin), which communicate with the ALSA sound subsystem inside the Linux kernel.
    • The kernel drivers then control the physical hardware such as audio codecs, sound cards, or interfaces like I2S or HDMI, completing the path from application audio streams to the real audio output or input devices.

Build PipeWire Image via Yocto
------------------------------

This section provides the steps to build a flashable image with PipeWire support via Yocto. The Yocto Project is an open-source collaboration project that helps developers create custom Linux-based systems regardless of the hardware architecture. Our Processor Linux SDK build is based on the Arago project which provides a set of layers for Openembedded and the Yocto Project targeting TI platforms.

Steps to Run Yocto Builds on Host
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Prerequisites (One-time setup)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The recommended Linux distribution is Ubuntu 22.04. The following build host packages are required for Ubuntu. The following command will install the required tools on the Ubuntu Linux distribution.
For Ubuntu 22.04, please run the following:

.. code-block:: bash

    $ sudo apt-get update
    $ # Install packages required for builds
    $ sudo apt-get -f -y install \\
          git build-essential diffstat texinfo gawk chrpath socat doxygen \\
          dos2unix python3 bison flex libssl-dev u-boot-tools mono-devel \\
          mono-complete curl python3-distutils repo pseudo python3-sphinx \\
          g++-multilib libc6-dev-i386 jq git-lfs pigz zstd liblz4-tool \\
          cpio file lz4 debianutils iputils-ping python3-git python3-jinja2 \\
          python3-subunit locales libacl1 unzip gcc python3-pip python3-pexpect \\
          xz-utils wget
    $ sudo locale-gen en_US.UTF-8

By default, Ubuntu uses “dash” as the default shell for /bin/sh. You must reconfigure to use bash by running the following command:

.. code-block:: bash

    $ sudo dpkg-reconfigure dash

Clone the oe-layer Setup
~~~~~~~~~~~~~~~~~~~~~~~~
.. code-block:: bash

    $ cd $HOME
    $ git clone https://git.ti.com/git/arago-project/oe-layersetup.git tisdk
    $ cd tisdk
    $ ./oe-layertool-setup.sh -f configs/processor-sdk/processor-sdk-scarthgap-12.00.00.06-config.txt

Download and Apply PipeWire Patches
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Following patches need to be applied to enable PipeWire and WirePlumber support in the image. These patches needs to be applied to meta-arago, which provides Arago distribution configuration for TI products. There is one more patch needed in meta-tisdk, for devices having PulseAudio integration in the OOB demo. meta-tisdk is the Yocto layer for TI Foundational SDK for Sitara MPU and Jacinto devices.
The patches from the download site are described in Table.

.. table:: Yocto PipeWire patches
    :align: left
    :widths: 20 40 40

    ==========  =================================================================  ===========================================================
    Patch Number Patch                                                              Description
    ==========  =================================================================  ===========================================================
    1           0001-recipes-multimedia-Add-pipewire-configuration-files.patch     Add reference PipeWire configuration files for 8-channel and 2-channel audio.
    2           0002-recipes-multimedia-Add-wireplumber-audio-configurati.patch    Add WirePlumber configuration with audio defaults service.
    3           0003-recipes-core-arago-default-image-Add-pipewire-audio-.patch    Enable PipeWire audio stack in arago-default-image.
    4           0001-ti-apps-launcher-Remove-pulseaudio-service-dependenc.patch    Remove PulseAudio service from image.
    ==========  =================================================================  ===========================================================

Following are the steps to download and apply the patches.

.. code-block:: bash

    $ cd $HOME
    $ git clone https://github.com/TexasInstruments/Linux-MPU-Audio-Enablement.git -b main
    $ cd $HOME/tisdk/sources/meta-arago
    $ patch -p1 < $HOME/Linux-MPU-Audio-Enablement/PipeWire/patches/meta-arago/0001-recipes-multimedia-Add-pipewire-configuration-files.patch
    $ patch -p1 < $HOME/Linux-MPU-Audio-Enablement/PipeWire/patches/meta-arago/0002-recipes-multimedia-Add-wireplumber-audio-configurati.patch
    $ patch -p1 < $HOME/Linux-MPU-Audio-Enablement/PipeWire/patches/meta-arago/0003-recipes-core-arago-default-image-Add-pipewire-audio-.patch
    $ cd $HOME/tisdk/sources/meta-tisdk
    $ patch -p1 < $HOME/Linux-MPU-Audio-Enablement/PipeWire/patches/meta-tisdk/0001-ti-apps-launcher-Remove-pulseaudio-service-dependenc.patch

Build PipeWire Image
^^^^^^^^^^^^^^^^^^^^
The final command below will build the tisdk-default-image, which is the Processor SDK image with arago filesystem and PipeWire support enabled.
.. code-block:: bash

    $ cd $HOME/tisdk
    $ cd build
    $ . conf/setenv

    # For RT (Real Time) Linux build
    $ MACHINE=<machine> ARAGO_RT_ENABLE=1 bitbake -k tisdk-default-image

    # For Non-RT Linux Build
    $ MACHINE=<machine> bitbake -k tisdk-default-image

Whereas MACHINE could be one of the following values:

.. table:: MACHINE values
    :align: left
    :widths: 30 70

    ============  ==============================================================
    MACHINE       Supported EVMs
    ============  ==============================================================
    am62xx-evm    AM62x Starter Kit (SK) - GP, HS-FS, HS-SE
    am62xx-lp-evm AM62x LP Starter Kit (SK) - HS-FS, HS-SE
    am62dxx-evm   AUDIO-AM62D-EVM evaluation module (EVM)
    am62lxxx-evm  AM62L EVM - HS-FS
    am62xxsip-evm SK-AM62-SIP Starter Kit (SK) Evaluation module (EVM)
    am62pxx-evm   AM62Px EVM - HS-FS, HS-SE
    am62axx-evm   AM62A Starter Kit (SK)
    ============  ==============================================================

Newly built wic image will be generated in deploy-ti/images/<machine>/directory.

AUDIO-AM62D-EVM Setup
^^^^^^^^^^^^^^^^^^^^^
Let’s take example of AM62D2-EVM to understand how to configure and use PipeWire. The AUDIO-AM62D-EVM evaluation module (EVM) or AM62D2-EVM is a low-cost expandable platform designed for developers to prototype and evaluate multi-channel audio applications across various use cases.

Hardware
~~~~~~~~
    • AUDIO-AM62D-EVM
    • MicroSD Card (minimum 16GB)
    • USB Type-C power supply (20W)
    • USB-to-UART cable
    • Windows or Linux host PC for flashing and console access
    • Audio Output device (speaker, TRS compatible)
    • Audio Input device (microphone, TRS compatible)

Configure EVM Boot Mode
~~~~~~~~~~~~~~~~~~~~~~~
The figure below shows some important cable connections, ports and switches.
Take note of the location of the "BOOTMODE" switch for SD card boot mode.
Setup EVM SD card boot mode setting:
    • BOOTMODE [ 8 : 15 ] (SW3) = 0100 0000
    • BOOTMODE [ 0 : 7 ] (SW2)  = 1100 0010

.. figure:: ../images/sd_boot_mode.png
    :align: center
    :width: 400

    SD Boot Mode

.. figure:: ../images/audio-am62d-evm.png
    :align: center
    :width: 400

    AUDIO-AM62D-EVM

UART Console Setup
~~~~~~~~~~~~~~~~~~
    • Connect UART to USB cable to EVM.
    • Identify the UART COM port as enumerated on the host machine (Windows Device Manager  Ports (COM & LPT)).
    • If you do not see any USB serial ports listed in Device Manager under Ports (COM & LPT), then install the UART to USB driver from FTDI.
    • Terminal Settings
        ◦ Baud Rate: 115200
        ◦ 8-N-1
    • Open Terminal and wait for EVM boot.

Flash the SD Card Image
~~~~~~~~~~~~~~~~~~~~~~~
    • rootfs wic image generated in last section with PipeWire support will be in deploy-ti/am62dxx-evm/images by name: tisdk-default-image-am62dxx-evm.rootfs.wic.xz
    • Use Balena Etcher
        ◦ Insert a micro-SD card into the USB SD card reader and start Etcher.
        ◦ Select the wic.xz image
        ◦ Select SD Card
        ◦ Click “Flash”

Booting EVM with SD Card
~~~~~~~~~~~~~~~~~~~~~~~~
    • Make sure the boot mode pins on the board are for SD Card boot.
    • Insert the SD Card into the SD Card slot
    • Connect the host PC to the USB Micro-B interface for UART
    • Power on the board
Log in as “root” with no password

.. code-block:: none

    Trying to boot from MMC2
    Authentication passed
    Authentication passed
    Authentication passed
    Authentication passed
    Authentication passed
    Starting ATF on ARM64 core...
    ...
     _____                    _____           _         _
    |  _  |___ ___ ___ ___   |  _  |___ ___  |_|___ ___| |_
    |     |  _| .'| . | . |  |   __|  _| . | | | -_|  _|  _|
    |__|__|_| |__,|_  |___|  |__|  |_| |___|_| |___|___|_|
    Arago Project am62dxx-evm -
    Arago 2025.01 am62dxx-evm -
    am62dxx-evm login:

Start and use PipeWire
----------------------

Check service status
^^^^^^^^^^^^^^^^^^^^
.. code-block:: bash

    root@<machine>: systemctl status pipewire wireplumber
    root@<machine>: systemctl status wireplumber

Enable PipeWire and Wireplumber
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Enable service to start automatically at boot
.. code-block:: bash

    root@<machine>: systemctl enable pipewire
    root@<machine>: systemctl enable wireplumber

A new service is also added by patches to set default audio devices in WirePlumber to avoid manual setup. To enable it:
.. code-block:: bash

    root@<machine>: systemctl enable set-audio-defaults

Start PipeWire and WirePlumber
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Start PipeWire and WirePlumber if not started by default
.. code-block:: bash

    root@<machine>: systemctl start pipewire
    root@<machine>: systemctl start wireplumber
    root@<machine>: systemctl start set-audio-defaults

General PipeWire commands
^^^^^^^^^^^^^^^^^^^^^^^^^
List all objects currently in PipeWire server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. code-block:: bash

    root@<machine>: pw-cli list-objects
    id 0, type PipeWire:Interface:Core/4
                object.serial = "0"
                core.name = "pipewire-0"
        id 1, type PipeWire:Interface:Module/3
                object.serial = "1"
                module.name = "libpipewire-module-rt"
        id 2, type PipeWire:Interface:Module/3
                object.serial = "2"
                module.name = "libpipewire-module-protocol-native"
        id 3, type PipeWire:Interface:SecurityContext/3
                object.serial = "3"
        id 4, type PipeWire:Interface:Module/3
                object.serial = "4"
                module.name = "libpipewire-module-profiler"
        id 5, type PipeWire:Interface:Profiler/3
                object.serial = "5"

List only nodes
~~~~~~~~~~~~~~~

.. code-block:: bash

    root@<machine>: pw-cli list-objects Node
        id 29, type PipeWire:Interface:Node/3
                object.serial = "29"
                factory.id = "11"
                priority.driver = "200000"
                node.name = "Dummy-Driver"
        id 30, type PipeWire:Interface:Node/3
                object.serial = "30"
                factory.id = "11"
                priority.driver = "190000"
                node.name = "Freewheel-Driver"
        id 31, type PipeWire:Interface:Node/3
                object.serial = "31"
                factory.id = "19"
                node.description = "Audio Output"
                node.name = "alsa_audio_sink"
                media.class = "Audio/Sink"
        id 32, type PipeWire:Interface:Node/3
                object.serial = "32"
                factory.id = "19"
                node.description = "Audio Input"
                node.name = "alsa_audio_source"
                media.class = "Audio/Source"

Inspect specific object
~~~~~~~~~~~~~~~~~~~~~~~

Inspect specific object using command pw-cli info <object-id>.

.. code-block:: bash

    root@<machine>: pw-cli info 31
        id: 31
        permissions: rwxm-
        type: PipeWire:Interface:Node/3
        input ports: 0/0
        output ports: 8/129
        state: "suspended"
        properties:
            factory.name = "api.alsa.pcm.source"
            node.name = "alsa_audio_source"
            node.description = "Audio Input"
            media.class = "Audio/Source"
            api.alsa.period-size = "1024"
            node.driver = "true"
            api.alsa.disable-mmap = "false"
            api.alsa.disable-batch = "false"
            api.alsa.path = "hw:0,0"
            audio.rate = "48000"
            audio.channels = "8"

Play and Record Stereo Audio
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Play audio via pw-play or aplay directly.
.. code-block:: bash

    root@<machine>: pw-play --target=alsa_audio_sink /usr/share/sample_audio.wav
    root@<machine>: aplay -r 48000 -f S32_LE -c 2 /usr/share/sample_audio.wav

Record audio via PipeWire user tools or aplay directly.
.. code-block:: bash

    root@ <machine>: pw-record --target=alsa_audio_source temp.wav
    root@<machine>: arecord -r 48000 -f S32_LE -c 2 temp.wav

Configuration Details
---------------------

A typical PipeWire setup consists of:
    • PipeWire daemon
    • Session manager (usually WirePlumber)
    • Compatibility servers (PulseAudio, JACK)
    • Client configuration
Each of these has its own configuration file which uses SPA JSON format, which is a relaxed JSON syntax.

.. table:: Configuration files
    :align: left
    :widths: 30 70

    ================  =========================================================
    Files             Purpose
    ================  =========================================================
    pipewire.conf     Configures the PipeWire daemon
    client.conf       Configures PipeWire clients
    pipewire-pulse.conf PulseAudio compatibility server
    filter-chain.conf Audio processing filters
    ================  =========================================================

Instead of modifying the main file, PipeWire recommends using drop-in files to override only specific settings. Example in directory

.. code-block:: bash

    $ /etc/pipewire/pipewire.conf.d/

Let’s discuss reference configuration files added specifically for AM62D2-EVM in patches in the upcoming sections.

Sink and Source Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
There are two reference configurations files, sink and source added for AM62D2-EVM.

90-pipewire-sink.conf
~~~~~~~~~~~~~~~~~~~~~
.. code-block:: ini

    # PipeWire sink configuration for AM62D.

    context.objects = [
        {
            factory = adapter
            args = {
                factory.name              = api.alsa.pcm.sink
                node.name                 = "alsa_audio_sink"
                node.description       = "Audio Output"
                media.class                 = "Audio/Sink"
                api.alsa.period-size   = 1024
                api.alsa.headroom        = 0
                api.alsa.disable-mmap = false
                api.alsa.disable-batch  = false
                api.alsa.path                  = "hw:0,0"
                audio.rate                       = 48000
                audio.channels              = 8
                audio.position               = [ FL FR FC LFE RL RR SL SR ]
            }
        }
    ]

91-pipewire-source.conf
~~~~~~~~~~~~~~~~~~~~~
.. code-block:: ini

    # PipeWire source configuration for AM62D.

    context.objects = [
        {
            factory = adapter
            args = {
                factory.name                   = api.alsa.pcm.source
                node.name                      = "alsa_audio_source"
                node.description            = "Audio Input"
                media.class                      = "Audio/Source"
                api.alsa.period-size        = 1024
                api.alsa.headroom         = 0
                api.alsa.disable-mmap  = false
                api.alsa.disable-batch   = false
                api.alsa.path                   = "hw:0,0"
                audio.rate                       = 48000
                audio.channels              = 8
                audio.position               = [ FL FR FC LFE RL RR SL SR ]
            }
        }
    ]

Both files use identical structural patterns with only key functional differences:
    • **context.objects**
      Main configuration array defining objects created in PipeWire context. Each object in this array becomes a node in the PipeWire graph.
    • **factory = adapter**
      Specifies that this object should be created using the "adapter" factory. Adapters in PipeWire are used to bridge between different APIs (in this case, ALSA to PipeWire). Both configuration files use adapter factory to bridge ALSA hardware to PipeWire nodes.
    • **factory.name**
      Determines direction: output vs input
    • **node.name**
      Internal PipeWire node identifier. Creates alsa_audio_sink for playback and alsa_audio_source for capture.
    • **node.description**
      Human-readable name in audio apps.
    • **media.class**
      PipeWire media classification "Audio/Sink" for playback and "Audio/Source" for recording.
    • **api.alsa.path**
      For direct hardware access. Only PipeWire can access the audio hardware and ALSA applications must go through PipeWire.
    • **audio.channels**
      Configures both input and output for 8 channel audio in case of AM62D2-EVM and stereo audio in case of other devices.
    These configurations create two fundamental nodes in PipeWire's audio graph:
      - Sink Node: Terminal endpoint for audio playback chains
      - Source Node: Starting point for audio capture chains
      - Matched Pair: Enables full-duplex audio applications
    For more information, please refer Alsa Configuration.

WirePlumber Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^
Use the command below to list all the available audio sinks and sources.
.. code-block:: bash

    root@<machine>: wpctl status
    PipeWire 'pipewire-0' [1.6.0, root@am62dxx-evm, cookie:3333499771]
    └─ Clients:
          34. WirePlumber [1.6.0, root@am62dxx-evm, pid:9716]
          58. WirePlumber [export] [1.6.0, root@am62dxx-evm, pid:9716]
          94. wpctl [1.6.0, root@am62dxx-evm, pid:9753]

    Audio
    ├ Devices:
    │ 59. Built-in Audio [alsa]
    │
    ├ Sinks:
    │ * 31. Audio Output [vol: 1.00]
    │ 68. Built-in Audio Stereo [vol: 0.40]
    │
    ├ Sources:
    │ * 32. Audio Input [vol: 1.00]
    │ 69. Built-in Audio Stereo [vol: 1.00]
    │
    ├ Filters:
    │
    └─ Streams:

    Video
    ├ Devices:
    │
    ├ Sinks:
    │
    ├ Sources:
    │
    ├ Filters:
    │
    └─ Streams:

    Settings
    └─ Default Configured Devices:
         0. Audio/Sink alsa_audio_sink
         1. Audio/Source alsa_audio_source

Use wpctl inspect to display information about the specified object.
.. code-block:: bash

    root@<machine>: wpctl inspect 31

Defaults source could also be set manually by using the ID number of sinks and sources:
.. code-block:: bash

    root@<machine>: wpctl set-default 30
    root@<machine>: wpctl set-default 31

The WirePlumber changes in the patches do not include any configuration changes but introduces an automated audio device configuration system through two key files:
    • set-audio-defaults.sh
    • set-audio-defaults.service
The shell script (set-audio-defaults.sh) implements an initialization sequence that waits up to 30 seconds for WirePlumber to become ready, then uses PipeWire's command-line tools (pw-cli and wpctl) to locate the alsa_audio_sink and alsa_audio_source nodes defined in the PipeWire configuration files discussed earlier and explicitly set them as the system's default audio devices. This automation is necessary because while PipeWire creates audio nodes based on the configuration files, it doesn't automatically designate them as the default devices that applications will use. The accompanying systemd service (set-audio-defaults.service) ensures this script runs automatically after WirePlumber starts.

Performance Benchmarks
----------------------

Let’s run multiple concurrent audio applications on each server and measure average performance for PulseAudio and PipeWire.
Following is the test setup
    • Hardware - AM62D2-EVM
    • Kernel - 6.18
    • Yocto - master

Latency
^^^^^^^
This test measures playback latency for PulseAudio and PipeWire. For PulseAudio, latency was measured using the sink latency reported by pactl, which reflects the effective playback delay observed at the audio output.
.. code-block:: bash

    root@<machine>: pactl list sinks | grep Latency
       Latency: 1270744 usec, configured 1365333 usec

For PipeWire latency was estimated by summing the buffer durations (quantum) of the active stream and sink nodes obtained from pw-top, representing application and device buffering.
.. code-block:: bash

    root@<machine>: pw-top
    S   ID  QUANT   RATE    WAIT    BUSY   W/Q   B/Q  ERR FORMAT           NAME
    S   29      0      0    ---     ---   ---   ---     0                  Dummy-Driver
    S   30      0      0    ---     ---   ---   ---     0                  Freewheel-Driver
    R   31   2048  48000  19.1us 504.7us  0.00  0.01    0    S32LE 8 48000 alsa_audio_sink
    R   77   4800  48000  28.9us 125.1us  0.00  0.00    0    S16LE 8 48000  = pw-play
    S   32      0      0    ---     ---   ---   ---     0                  alsa_audio_source
    S   68      0      0    ---     ---   ---   ---     0                  alsa_output.platform-sound.stereo-fallback
    S   69      0      0    ---     ---   ---   ---     0                  alsa_input.platform-sound.stereo-fallback

.. table:: Default Latency
    :align: left
    :widths: 30 70

    ============  ==============
    Audio Server  Latency (ms)
    ============  ==============
    PulseAudio    ~1365 ms
    PipeWire      ~143 ms
    ============  ==============

These latency values can be further reduced by changing configuration for ex. quantum, clock rate, number of fragments, fragment size etc.

CPU and Memory Usage
^^^^^^^^^^^^^^^^^^^^
This test measure CPU utilization and memory usage when multiple audio streams simultaneously use the audio server at the same latency configuration. We will configure PulseAudio to work at latency closer to 143 msec by changing PULSE_LATENCY_MSEC so that configured latency value is closer to 143msec.
.. code-block:: bash

    root@<machine>: export PULSE_LATENCY_MSEC=325
    root@<machine>: pactl list sinks | grep Latency
       Latency: 141702 usec, configured 142500 usec

In PulseAudio, latency cannot be set to an exact value using PULSE_LATENCY_MSEC, as it is treated as a target rather than a strict configuration. The actual latency is determined by internal buffer fragmentation, hardware constraints, and scheduler behavior. As a result, the observed latency may differ significantly from the requested value and typically aligns with discrete buffer sizes supported by the underlying ALSA device. However, there are other methods to control latency by directly modifying variables that affect latency such as fragments, fragments size etc. Refer Pulseaudio modules documentation for more details.

.. table:: CPU Load (same latency)
    :align: left
    :widths: 30 70

    ============  ==============
    Audio Server  CPU Usage (average)
    ============  ==============
    PulseAudio    ~5%
    PipeWire      ~4%
    ============  ==============

.. table:: Memory usage (same latency)
    :align: left
    :widths: 30 70

    ============  ==================
    Audio Server  Memory Usage (average)
    ============  ==================
    PulseAudio    ~21600 KB
    PipeWire      ~66800 KB (Wireplumber- ~22990KB)
    ============  ==================

CPU and Memory Usage with Resampling
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This tests measures CPU overhead and memory usage introduced when the audio server performs sample rate conversion and configured at same latency. This test evaluates the efficiency of the resampling implementation. Both PulseAudio and PipeWire are configured to resample audio to 44.1 KHz (vs native 48 KHz).
For PulseAudio, resampling can be introduced by adding following commands:
.. code-block:: bash

    root@<machine>: echo “default-sample-rate = 44100” > /etc/pulse/daemon.conf
    root@<machine>: echo “alternate-sample-rate = 44100” > /etc/pulse/daemon.conf

For PipeWire, resampling to 44.1 KHz can be done by following commands:
.. code-block:: bash

    root@<machine>: pw-metadata -n settings 0 clock.force-rate 44100

.. table:: CPU Usage with Resampling
    :align: left
    :widths: 30 70

    ============  ==============
    Audio Server  CPU Usage (average)
    ============  ==============
    PulseAudio    ~23%
    PipeWire      ~10%
    ============  ==============

.. table: Memory Usage with Resampling
    :align: left
    :widths: 30 70

    ============  ==================
    Audio Server  Memory Usage (average)
    ============  ==================
    PulseAudio    ~21500 KB
    PipeWire      ~59900 KB (Wireplumber-23000)
    ============  ==================

References
----------

    • PipeWire documentation
    • Pulseaudio modules documentation
