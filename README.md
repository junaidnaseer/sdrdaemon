SDRdaemon
=========

<h1>SDRdaemon is going to the end of its life</h1>

Soon SDRangel https://github.com/f4exb/sdrangel project is going to take over by absorbing the same functionnality in the form of two new channel plugins. There are many advantages:

  - Better performance: early tests show at least a 2x improvement in CPU load (two times lower)
  - Integration of all devices supported by SDRangel
  - Possibility to run the Rx and Tx channels of the same device concurrently
  - Use the same REST API as SDRangel to control all parameters. Makes nanomsg dependency obsolete.
  - Even more configuration options provided by SDRangel
  
Consequently as soon as the new release (4.1) of SDRangel is out this project will be archived.

<h1>Introduction</h1>

**SDRdaemon** package can be used to:
  - send I/Q samples read from a SDR device over the network via UDP (Rx mode)
  - rceive I/Q samples from the network via UDP and send them to a SDR device (Tx mode) 

**sdrdaemonrx** is a basic software-defined radio receiver that just sends the I/Q samples over the network via UDP. It was developed on the base of NGSoftFM (also found in this Github repo: https://github.com/f4exb/ngsoftfm) and shares a lot of the code for the interface with the SDR hardware devices.

It conveys meta data in the data flow so that the receiving application is informed about parameters essential to render correctly the data coming next such as the sample rate, the number of bytes used for the samples, the number of effective sample bits, the center frequency... (See the "Data format" chapter for detals).

While running the program accepts configuration commands on a TCP port using nanomsg messages with a content in the same format as the configuration string given on the command line (See the "Running" chapter for details). This provides a dynamic control of the device or features of the application such as the decimation. A Python script is provided to send such messages.

In order to recover possible lost blocks it uses Cauchy MDS Block Erasure codec to encode data with redundancy, It can add a user defined number of redundant block so that if the nominal number of blocks is received (128 blocks) it can recover the lost blocks in any position.

Note that if you set the number of redundant blocks to 0 then no FEC is used.

**sdrdaemontx** does the same thing as sdrdaemonrx but the other way round. It takes blocks read from UDP possibly with redundant blocks and sends the I/Q data to a SDR transmitter.

Hardware supported:

Receivers:

  - **RTL-SDR** based (RTL2832-based) hardware is supported and uses the _librtlsdr_ library to interface with the RTL-SDR hardware.
  - **HackRF** One and variants are supported with _libhackrf_ library.
  - **Airspy** is supported with _libairspy_ library.
  - **BladeRF** is supported with _libbladerf_ library.

Transmitters:

  - **HackRF** One and variants are supported with _libhackrf_ library.

SDRdaemon programs can be used conveniently along with SDRangel (found in this Github repo: https://github.com/f4exb/sdrangel) as the client application. So in this remote type of configuration you will need both an angel and a daemon :-)

GNUradio is also supported with a specific `sdrdaemonsource`source block for Rx devices provided in the `gr-sdrdaemon` OOT module. The `sdrdaemonsink` sink block for Tx devices does not exist at the moment.

SDRdaemon package requires:

 - Linux
 - C++11
 - Boost includes only (library is not linked with SDRdaemon)
 - nanomsg
 - CM256cc library found in this Github repo: https://github.com/f4exb/cm256cc
 - RTL-SDR library (http://sdr.osmocom.org/trac/wiki/rtl-sdr) for RTL-SDR support
 - HackRF library (https://github.com/mossmann/hackrf/tree/master/host/libhackrf) for HackRF support
 - Airspy library (https://github.com/airspy/host/tree/master/libairspy) for Airspy support
 - BladeRF library (https://github.com/Nuand/bladeRF/tree/master/host) for BladeRF support
 - supported hardware
 - A computer or embedded device such as the Raspberry Pi 2 or 3 to which you connect the hardware.
 
&#9758; On Raspberry Pi 3 it is recommended to run in a native aarch64 system for better performance. This is particularly true on the Tx side with `sdrdaemontx`. For the moment only OpenSUSE does that: [link](https://en.opensuse.org/HCL:Raspberry_Pi3).

For the latest version, see https://github.com/f4exb/SDRdaemon

Branches:

  - _master_ is the "production" branch with the most stable release
  - _dev_ is the development branch that contains current developments that will be eventually released in the master branch
  - _fix_ contains fixes that cannot wait for the dev branch to go to production


<h1>Prerequisites</h1>

<h2>Base requirements</h2>

<h3>Ubuntu/Debian</h3>

  - `sudo apt-get install cmake pkg-config libusb-1.0-0-dev libasound2-dev libboost-all-dev libnanomsg-dev`

<h3>OpenSUSE</h3>

in particular if you run the aarch64 version with the RPi3. Performance in an aarch64 (ARM 64 bits amrv8) is considerably better than in an armv7 system. This is particularly true with `sdradaemontx`. Use gcc version 6 to get automatic vectorization.

  - `sudo zypper install cmake gcc6 gcc6-c++ libusb-1_0-devel boost-devel fftw3-devel`

Do once in the shell where you compile:

  - `export CC=/usr/bin/gcc-6`
  - `export CXX=/usr/bin/g++-6`

You will need to compile and install libnanomsg because it is not available as a package:

  - `git clone https://github.com/ZewoGraveyard/libnanomsg.git`
  - `cd libnanomsg`
  - `mkdir build; cd build`
  - `cmake -DCMAKE_INSTALL_PREFIX=/opt/install/libnanomsg ..`
  - `make -j4 install`

<h2>Forward Erasure Correction (FEC) support</h2>

You have to install [CM256cc](https://github.com/f4exb/cm256cc). You will then have to specify the include and library paths on the cmake command line. Say if you install cm256cc in `/opt/install/cm256cc` you will have to add `-DCM256CC_INCLUDE_DIR=/opt/install/cm256cc/include/cm256cc -DCM256CC_LIBRARIES=/opt/install/cm256cc/lib/libcm256cc.so` to the cmake commands.

`sdrdaemonrx` binary recognizes the configuration commmand `fecblk` to specify the number of FEC blocks. When `fecblk=0` is specified in the commands and hence no FEC blocks are added. 

Frames and blocks are numbered and even if no FEC blocks are added this can help in reconstructing frames with appropriate timings.

<h2>Airspy support</h2>

Airspy support must be installed for SDRdaemon to work with an Airspy device.

If you install from source (https://github.com/airspy/host/tree/master/libairspy) in your own installation path you have to specify the include path and library path. For example if you installed it in `/opt/install/libairspy` you have to add `-DLIBAIRSPY_LIBRARIES=/opt/install/libairspy/lib/libairspy.so -DLIBAIRSPY_INCLUDE_DIR=/opt/install/libairspy/include` to the cmake options.

To install the library from a Debian/Ubuntu installation just do:

  - `sudo apt-get install libairspy-dev`

<h2>BladeRF support</h2>

BladeRF support must be installed for SDRdaemon to work with a BladeRF device.

If you install from source (https://github.com/Nuand/bladeRF) in your own installation path you have to specify the include path and library path. For example if you installed it in `/opt/install/libbladerf` you have to add `-DLIBBLADERF_LIBRARIES=/opt/install/libbladeRF/lib/libbladeRF.so -DLIBBLADERF_INCLUDE_DIR=/opt/install/libbladeRF/include` to the cmake options.

To install the library from a Debian/Ubuntu installation just do:

  - `sudo apt-get install libbladerf-dev`

Note: for the BladeRF to work effectively on FM broadcast frequencies you have to fit it with the XB200 extension board.

<h2>HackRF support</h2>

HackRF support must be installed for SDRdaemon to work with a HackRF device.

If you install from source (https://github.com/mossmann/hackrf/tree/master/host/libhackrf) in your own installation path you have to specify the include path and library path. For example if you installed it in `/opt/install/libhackrf` you have to add `-DLIBHACKRF_LIBRARIES=/opt/install/libhackrf/lib/libhackrf.so -DLIBHACKRF_INCLUDE_DIR=/opt/install/libhackrf/include` to the cmake options.

To install the library from a Debian/Ubuntu installation just do:

  - `sudo apt-get install libhackrf-dev`

<h2>RTL-SDR support</h2>

The Osmocom RTL-SDR library must be installed before you can use SDRdaemon with a RTL-SDR device.
See http://sdr.osmocom.org/trac/wiki/rtl-sdr for more information.
SDRdaemon has been tested successfully with RTL-SDR 0.5.3. Normally your distribution should provide the appropriate librtlsdr package.
If you go with your own installation of librtlsdr you have to specify the include path and library path. For example if you installed it in `-DLIBRTLSDR_LIBRARIES=/opt/install/librtlsdr/lib/librtlsdr.so -DLIBRTLSDR_INCLUDE_DIR=/opt/install/librtlsdr/include` to the cmake options

To install the library from a Debian/Ubuntu installation just do:

  - `sudo apt-get install librtlsdr-dev`

<h2>nanomsg custom installation</h2>

If you build nanomsg from source obtained either by git clone or a released source package and install it in your own path (ex: `/opt/install/libnanomsg`) you will need to specify the include and library paths like this: `-DLIBNANOMSG_LIBRARIES=/opt/install/libnanomsg/lib/libnanomsg.so -DLIBNANOMSG_INCLUDE_DIR=/opt/install/libnanomsg/include`

<h1>Installing</h1>

To install SDRdaemon, download and unpack the source code and go to the
top level directory. Then do like this:

 - `mkdir build`
 - `cd build`
 - `cmake ..`

Compile and install

 - `make -j8` (for machines with 8 CPUs)
 - `make install`


<h1>Running</h1>

<h2>Rx examples</h2>

Typical commands:

  - RTL-SDR: `./sdrdaemonrx -t rtlsdr -I 192.168.1.3 -D 9090 -C 9091 -c txdelay=300,fecblk=8,freq=433970000,srate=1000000,ppmp=58,gain=40.2,decim=5,fcpos=2`
    - Use RTL-SDR device #0
    - Destination address for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - Using TCP port `9091` to listen to configuration commands (it is the default anyway)
    - Startup configuration:
      - Wait between UDP blocks: 300 microseconds
      - FEC: add 8 FEC blocks to the 128 blocks data frame resulting in a total of 136 blocks per frame.
      - Center frequency: _433.97 MHz_
      - Device sample rate: _1 MHz_
      - Local oscillator correction: _58 ppm_
      - RF gain: _40.2 dB_
      - Decimation: 2^_5_ = 32; thus stream sample rate is 31.25 kHz
      - Position of center frequency: _2_ is centered (decimation around the center)
  - Airspy: `./sdrdaemonrx -t airspy -I 192.168.1.3 -D 9090 -c txdelay=300,fecblk=8,freq=433970000,srate=10000000,ppmn=1.7,lgain=13,mgain=9,vgain=6,decim=5,fcpos=0`
    - Use Airspy device #0
    - Destination address for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - Using TCP port `9091` to listen to configuration commands (it is the default anyway)
    - Startup configuration:
      - Wait between UDP blocks: 300 microseconds
      - FEC: add 8 FEC blocks to the 128 blocks data frame resulting in a total of 136 blocks per frame.
      - Center frequency: _433.97 MHz_
      - Device sample rate: _10 MHz_
      - LO correction: _-1.7 ppm_
      - LNA gain: _13 dB_
      - Mixer gain: _9 dB_
      - VGA gain: _6 dB_
      - Decimation: 2^_5_ = 32; thus stream sample rate is 312.5 kHz
      - Position of center frequency: _0_ is infra-dyne (decimation around -fc/4)
  - HackRF: `./sdrdaemonrx -t hackrf -I 192.168.1.3 -D 9090 -c txdelay=300,fecblk=8,freq=433970000,srate=3200000,lgain=32,vgain=24,bwfilter=1.75,decim=3,fcpos=1`
    - Use HackRF device #0
    - Destination address for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - Using TCP port `9091` to listen to configuration commands (it is the default anyway)
    - Startup configuration:
      - Wait between UDP blocks: 300 microseconds
      - FEC: add 8 FEC blocks to the 128 blocks data frame resulting in a total of 136 blocks per frame.
      - Center frequency: _433.97 MHz_
      - Device sample rate: _3.2 MHz_
      - LNA gain: _32 dB_
      - VGA gain: _24 dB_
      - Decimation: 2^_3_ = 8; thus stream sample rate is 400 kHz
      - Position of center frequency: _1_ is supra-dyne (decimation around fc/4)
  - BladeRF: `./sdrdaemonrx -t bladerf -I 192.168.1.3 -D 9090 -c txdelay=300,fecblk=8,freq=433900000,srate=3200000,lgain=6,v1gain=6,v2gain=3,decim=3,bw=2500000,fcpos=1`
    - Use BladeRF device #0
    - Destination address for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - Using TCP port `9091` to listen to configuration commands (it is the default anyway)
    - Startup configuration:
      - Wait between UDP blocks: 300 microseconds
      - FEC: add 8 FEC blocks to the 128 blocks data frame resulting in a total of 136 blocks per frame.
      - Center frequency: _433.9 MHz_
      - Device sample rate: _3.2 MHz_
      - RF filter bandwidth: _2.5 MHz_
      - LNA gain: _6 dB_
      - VGA1 gain: _6 dB_
      - VGA2 gain: _3 dB_
      - Decimation: 2^_3_ = 8; thus stream sample rate is 400 kHz
      - Position of center frequency: _1_ is supra-dyne (decimation around fc/4)
  - Test signal source: `./sdrdaemonrx -t test -I 192.168.1.3 -D 9090 -c fecblk=8,power=40,decim=2,srate=500000,dfp=25000`
    - Destination address for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - FEC: add 8 FEC blocks to the 128 blocks data frame resulting in a total of 136 blocks per frame.
    - Carrier relative power is _-40 dB_
    - Base sample rate is _500 kHz_
    - Decimation is 2^_2_ = 4; thus stream sample rate is 125 kHz
    - Carrier frequency shift from the center is _25 kHz_

<h2>Tx examples</h2>

Typical commands:

  - HackRF: `./sdrdaemontx -t hackrf -I 192.168.1.3 -D 9090 -c freq=433970000,srate=3200000,vgain=24,bwfilter=1.75,interp=3`
    - Use HackRF device #0
    - Address of interface for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - Using TCP port `9091` to dialog with the remote for commands and status (it is the default anyway)
    - Startup configuration:
      - Center frequency: _433.97 MHz_
      - Device sample rate: _3.2 MHz_
      - VGA gain: _24 dB_
      - Interpolation: 2^_3_ = 8; thus stream sample rate is 400 kHz

  - File sink: `./sdrdaemontx -t file -I 192.168.1.3 -D 9090 -c freq=433970000,srate=64000,interp=1,file=test.sdriq`
    - Address of interface for the data is: `192.168.1.3`
    - Using UDP port `9090` for the data (it is the default anyway)
    - Using TCP port `9091` to dialog with the remote for commands and status (it is the default anyway)
    - Startup configuration:
      - Center frequency: _433.97 MHz_
      - Device sample rate: _128 kHz_
      - Interpolation: 2^_1_ = 2; thus stream sample rate is 64 kHz

<h2>All options</h2>

 - `-t devtype` is mandatory and must be either (depending on support libraries installed): 
    - `rtlsdr` for RTL-SDR devices 
    - `hackrf` for HackRF devices
    - `airspy` for Airspy
    - `bladerf` for BladeRF
    - `test` for test signal source (Rx only not hardware dependent)
    - `file` for file sink (Tx only not hardware dependent)
 - `-c config` Comma separated list of configuration options as key=value pairs or just key for switches. Depends on device type (see next paragraphs).
 - `-d devidx` Device index, 'list' to show device list (default 0)

<h2>Common configuration option for UDP transmission (sdrdaemonrx, sdrdaemon)</h2>

  - `txdelay=<int>` Rx only. Delay between the transmission of successive UDP blocks in microseconds. This may not result in the exact delay in microseconds as this is in fact the argument to `usleep` function. The system guarantees that at least this delay is respected and in many practical cases it is not possible to have a delay smaller than ~100 microseconds. You may adjust this number depending on the speed of your link. This prevents UDP congestion by mitigating competition between the process sending blocks as fast as possible and the IP link absorbing them. 

<h2>Common configuration option for Forward Erasure Correction (sdrdaemonrx)</h2>

  - `fecblk=<int>` Rx only. Value should be between 0 (no FEC) and 127. This is the number of FEC blocks added to the 128 I/Q data blocks sent per frame. See the "Data formats" chapter for details about the frame construction in the FEC case. In Tx mode the number of FEC blocks is given in the meta data of each frame.

<h2>Common configuration options for the decimation (sdrdaemonrx, sdrdaemon)</h2>

  - `decim=<int>` log2 of the decimation factor. Samples collected from the device are down-sampled by two to the power of this value. On 8 bit samples native systems (RTL-SDR and HackRF) for a value greater than 0 (thus an effective downsampling) the size of the samples is increased to 2x16 bits.
  - `fcpos=<int>` Relative position of the center frequency in the resulting decimation:
    - `0` is infra-dyne i.e. decimation is done around -fc/4 where fc is the device center frequency
    - `1` is supra-dyne i.e. decimation is done around fc/4
    - `2` is centered i.e. decimation is done around fc

<h2>Common configuration options for the interpolation (sdrdaemontx)</h2>

  - `interp=<int>` log2 of the interpolation factor. Samples received from the network are up sampled by two to the power of this value. Samples are recived as 2x16 bits and resized depending on the transmiting device. Interpolation is done always centered on the transmission frequency. There is no infra-dyne nor supra-dyne translation.

<h2>Device type specific configuration options</h2>

Note that these options can be used both as the initial configuration as the argument of the `-c` option and as the dynamic configuration sent on the UDP configuration port specified by the `-C` option.

<h3>RTL-SDR</h3>

  - `freq=<int>` Desired tune frequency in Hz. Accepted range from 10M to 2.2G. (default 100M: `100000000`)
  - `gain=<x>` (default `auto`)
    - `auto` Selects gain automatically
    - `list` Lists available gains and exit
    - `<float>` gain in dB. Possible gains in dB are: `0.0, 0.9, 1.4, 2.7, 3.7, 7.7, 8.7, 12.5, 14.4, 15.7, 16.6, 19.7, 20.7, 22.9, 25.4, 28.0, 29.7, 32.8, 33.8, 36.4, 37.2, 38.6, 40.2, 42.1, 43.4, 43.9, 44.5, 48.0, 49.6`
  - `srate=<int>` Device sample rate. valid values in the [225001, 300000], [900001, 3200000] ranges. (default `1000000`)
  - `ppmp=<int>` Argument is positive. Positive LO correction in ppm. LO is corrected by this value in ppm
  - `ppmn=<int>` Argument is positive. Negative LO correction in ppm. LO is corrected by minus this value in ppm. If `ppmp` is also specified `ppmp` takes precedence.
  - `agc=<int>` Turn on (1) or off (0) the device AGC (default 0: off)

<h3>HackRF</h3>

  - `freq=<float>` Desired tune frequency in Hz. Valid range from 1M to 6G. (default 100M: `100000000`)
  - `srate=<float>` Device sample rate (default `5000000`). Valid values from 1M to 20M. In fact rates lower than 10M are not specified in the datasheets of the ADC chip however a rate of `1000000` (1M) still works well with SDRdaemon.
  - `ppmp=<float>` Argument is positive. Positive LO correction in ppm. LO is corrected by this value in ppm
  - `ppmn=<float>` Argument is positive. Negative LO correction in ppm. LO is corrected by minus this value in ppm. If `ppmp` is also specified `ppmp` takes precedence.  
  - `lgain=<x>` (Rx only) LNA gain in dB. Valid values are: `0, 8, 16, 24, 32, 40, list`. `list` lists valid values and exits. (default `16`)
  - `vgain=<x>` VGA gain in dB. Valid values are: `0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36, 38, 40, 42, 44, 46, 48, 50, 52, 54, 56, 58, 60, 62, list`. `list` lists valid values and exits. (default `22`)
  - `bwfilter=<x>` RF (IF) filter bandwidth in MHz. Actual value is taken as the closest to the following values: `1.75, 2.5, 3.5, 5, 5.5, 6, 7,  8, 9, 10, 12, 14, 15, 20, 24, 28, list`. `list` lists valid values and exits. (default `2.5`)
  - `extamp=<int>` Turn on (1) or off (0) the extra amplifier (default 0: off)
  - `antbias=<int>` Turn on (1) or off (0) the antenna bias for remote LNA (default 0: off)
  - `pwidle=<float>` (Tx only) Value in negative dB of I/Q constant carrier power when idle (default 0: silent)

<h3>Airspy</h3>

  - `freq=<int>` Desired tune frequency in Hz. Valid range from 1M to 1.8G. (default 100M: `100000000`)
  - `srate=<int>` Device sample rate. `list` lists valid values and exits. (default `10000000`). Valid values depend on the Airspy firmware. Airspy firmware and library must support dynamic sample rate query.
  - `ppmp=<float>` Argument is positive. Positive LO correction in ppm. LO is corrected by this value in ppm
  - `ppmn=<float>` Argument is positive. Negative LO correction in ppm. LO is corrected by minus this value in ppm. If `ppmp` is also specified `ppmp` takes precedence.
  - `lgain=<x>` LNA gain in dB. Valid values are: `0, 1, 2, 3, 4, 5, 6, 7, 8 ,9 ,10, 11 12, 13, 14, list`. `list` lists valid values and exits. (default `8`)
  - `mgain=<x>` Mixer gain in dB. Valid values are: `0, 1, 2, 3, 4, 5, 6, 7, 8 ,9 ,10, 11 12, 13, 14, 15, list`. `list` lists valid values and exits. (default `8`)
  - `vgain=<x>` VGA gain in dB. Valid values are: `0, 1, 2, 3, 4, 5, 6, 7, 8 ,9 ,10, 11 12, 13, 14, 15, list`. `list` lists valid values and exits. (default `0`)  
  - `antbias=<int>` Turn on (1) or off (0) the antenna bias for remote LNA (default 0: off)
  - `lagc=<int>` Turn on (1) or off (0) the LNA AGC (default 0: off)
  - `magc=<int>` Turn on (1) or off (0) the mixer AGC (default 0: off)

<h3>BladeRF</h3>

  - `freq=<int>` Desired tune frequency in Hz. Valid range low boundary depends whether the XB200 extension board is fitted (default `300000000`).
    - XB200 fitted: 100kHz to 3,8 GHz
    - XB200 not fitted: 300 MHZ to 3.8 GHz.
  - `srate=<int>` Device sample rate in Hz. Valid range is 48kHZ to 40MHz. (default `1000000`).
  - `bw=<int>` IF filter bandwidth in Hz. `list` lists valid values and exits. (default `1500000`).
  - `lgain=<x>` LNA gain in dB. Valid values are: `0, 3, 6, list`. `list` lists valid values and exits. (default `3`)
  - `v1gain=<x>` VGA1 gain in dB. Valid values are: `5, 6, 7, 8 ,9 ,10, 11 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, list`. `list` lists valid values and exits. (default `20`)  
  - `v2gain=<x>` VGA2 gain in dB. Valid values are: `0, 3, 6, 9, 12, 15, 18, 21, 24, 27, 30, list`. `list` lists valid values and exits. (default `9`)  

<h3>Test (Rx only)</h3>

  - `freq=<int>` Desired center frequency in Hz sent in the meta data. Valid range 10 kHz to 10 GHz exclusive (default `435000000` i.e. 435 MHz).
  - `srate=<int>` Base sample rate in Hz. Valid range is 8kHZ to 10MHz. (default `5000000` i.e. 5 MS/s).
  - `power=<int>` Relative power of CW signaler in negative dB (i.e. 40 is -40 dB) (default `0`).
  - `dfp=<int>` Positive shift frequency of carrier from center frequency in Hz (default `100000` i.e. 100 kHz)
  - `dfn=<int>` Negative shift frequency of carrier from center frequency in Hz (default `100000` i.e. -100 kHz)
  - `blklen=<int>` Waveform buffer length in number of samples (default 64kS)

<h3>File sink (Tx only)</h3>

  - `freq=<int>` Desired center frequency in Hz sent in the meta data. Valid range 10 kHz to 10 GHz exclusive (default `435000000` i.e. 435 MHz).
  - `srate=<int>` Base sample rate in Hz. Valid range is 1MHZ to 6GHz. (default `48000` i.e. 48 kS/s).
  - `file=<string>` Name of the output file. (default `test.sdriq`).

<h2>Dynamic remote control</h2>

SDRdaemon listens on a TCP port (the configuration port) for incoming nanomsg messages consisting of a configuration string as described just above. You can use the utility `sdrdmnctl` in the bin directory of the installation directory (sits along `sdrdaemonrx` and other) to send such messages. It defaults to the localhost (`127.0.0.1`) and port `9091`. The configuration string is given as the `-c` option (same as for `sdrdaemon`). Example:

`/opt/install/sdrdaemon/bin/sdrdmnctl -I 192.168.1.3 -C 9999 -c frequency=433970000`

The complete list of options is:

  - `-I` IP address (or name defined by the DNS) of the machine hosting SDRdaemon (default `127.0.0.1`).
  - `-C` TCP port where SDRdaemon listens for configuration commands using nanomsg (default: `9091`).
  - `-c` message string. This is where you specify the configuration as a comma separated list of key=values (default: `freq=100000000`).
  - `-t` timeout in seconds. Timeout after which communication with SDRdaemon is abandoned (default: `2`).
  - `-h` online help

The nanomsg connection is specified as a paired connection (`NN_PAIR`). The connection can be managed by any program at the convenience of the user as long as the connection type is respected.

<h2>Running as a service</h2>

Have a look at the `service` subdirectory.

<h1>Data formats</h1>

<h2>Packaging</h2>

The I/Q data is sent in frames of 128 fixed size data blocks including a first block ("block zero") containing only meta data and a variable number of FEC blocks up to 127 FEC blocks. It is possible to use this scheme without FEC in which case no additional FEC blocks are present. All blocks have a fixed size of 512 bytes that represent the UDP payload size. The first 4 bytes are occupied by signalling data consisting of a 2 bytes frame count (wraps around at 65535), a 1 byte block count (0 to 127 (min) or 255 (max)) and a 1 byte filler. The rest is occupied by either the meta data (block zero), actual I/Q samples (127 samples per block resulting in 508 bytes) for data bytes or FEC data. The FEC is calculated on the 128 blocks of 508 bytes of meta data and I/Q samples.

Thus a complete frame contains 127 * 127 = 16129 samples.

<h2>Meta data block</h2>

The block of "meta" data consists of the following (values expressed in bytes):

<table>
    <tr>
        <th>Offset</th>
        <th>Length</th>
        <th>Type</th>
        <th>Content</th>
    </tr>
    <tr>
        <td>0</td>
        <td>4</td>
        <td>unsigned integer</td>
        <td>Center frequency of reception in kHz</td>
    </tr>
    <tr>
        <td>4</td>
        <td>4</td>
        <td>unsigned integer</td>
        <td>Stream sample rate (Samples/second)</td>
    </tr>
    <tr>
        <td>8</td>
        <td>1</td>
        <td>unsigned char</td>
        <td>number of bytes per sample. Practically 1 or 2</td>
    </tr>
    <tr>
        <td>9</td>
        <td>1</td>
        <td>unsigned char</td>
        <td>number of effective bits per sample. Practically 8 to 16</td>
    </tr>    
    <tr>
        <td>10</td>
        <td>1</td>
        <td>unsigned char</td>
        <td>number of (FEC protected) data blocks. Practically 128</td>
    </tr>    
    <tr>
        <td>11</td>
        <td>1</td>
        <td>unsigned char</td>
        <td>number of FEC blocks. Practically 0 to 127</td>
    </tr>
    <tr>
        <td>12</td>
        <td>4</td>
        <td>unsigned integer</td>
        <td>Seconds of Unix timestamp at the beginning of the sending processing</td>
    </tr>
    <tr>
        <td>16</td>
        <td>4</td>
        <td>unsigned integer</td>
        <td>Microseconds of Unix timestamp at the beginning of the sending processing</td>
    </tr>
    <tr>
        <td>20</td>
        <td>4</td>
        <td>unsigned integer</td>
        <td>CRC32 of the above (20 bytes)</td>
    </tr>
</table>

Total size is 24 bytes. The 484 (!) remaining bytes are reserved for future use. 

<h1>GNUradio supoort</h1>

The _gr-sdrdaemon_ module is provided in the _gr-sdrdaemon_ subdirectory. This subdirectory is a complete OOT module that can be built independently following GNUradio standards. Please refer to the documentation found in this directory for further information.

<h1>License</h1>

**SDRdaemon**, copyright (C) 2015-2017, Edouard Griffiths, F4EXB

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License along
with this program; if not, see http://www.gnu.org/licenses/gpl-2.0.html
