# ROS driver for the u-blox ZED-F9P GNSS receiver

*Brief setup instructions by Jonas Beuchert, original readme [below](#ublox_driver).*



You need a C++11 compiler, ROS, Eigen 3.3.3, and Boost, but they should come pre-installed with ROS already.

Install **Google's glog library** for message output:

```shell
$ sudo apt-get install libgoogle-glog-dev
```

Clone [gnss_comm](https://github.com/HKUST-Aerial-Robotics/gnss_comm) for ROS message definitions as well as this repository:

```shell
$ cd /path/to/your_catkin_ws/src
$ git clone git@github.com:HKUST-Aerial-Robotics/gnss_comm.git
$ git clone git@github.com:ori-drs/ublox_driver.git
```

Depending on your desired configuration, you might need to adapt [the configuration file](./config/driver_config.yaml), but it should work out-of-the-box.

Check which port the device is connected to, e.g. `$ ls /dev | grep tty` and unplugging and plugging in again the device. Usually the port should be `dev/ttyACM0` (default in the configuration file) or `/dev/ttyACM1`. Make sure that the correct `input_serial_port` is set in [the configuration file](./config/driver_config.yaml).

In case the receiver has never been used with ROS before make sure that you followed the **receiver configuration** given below.

**Grant permission to the port**, either temporarily (has to be repeated after every restart) with:

```shell
$ sudo chmod 666 /dev/ttyACM0
```

or alternatively permanently by adding a user to the `dialout` group with:

```shell
$ sudo adduser $USER dialout
```

and logging out and in again.

Finally, **build** the driver using catkin tools:

```shell
$ cd /path/to/your_catkin_ws
$ catkin build ublox_driver
```

**Launch** the `roscore` in a terminal and the driver in another one:

```shell
$ source devel/setup.bash
$ roslaunch ublox_driver ublox_driver.launch
```

It might be worth it to check out [Section 5](#5-synchronize-system-time) of the original read-me below to synchronize the local system time with a global time reference (and, therefore, with the timestamps of the GNSS observations.)

## Troubleshooting

If the driver prints the errors `ubx rxmrawx week=0 error` or `open: No such file or directory`, a cause can be that the port in [the configuration file](./config/driver_config.yaml) is set incorrectly.

In case a permission-denied error occurs, then the driver might not have access rights to the port, be sure that you have granted the user permissions to read and write to the port.

If no messages appear despite there being view of the sky, you might have to manually enable the binary messages of the receiver by following the receiver configuration below. The code is written in such a way that it gets stuck in an infinite while loop inside the serial handler in case a non-binary message is received.

## Receiver configuration

You can use different software to change the configuration of the receiver firmware (e.g., the measurement frequency or which messages are broadcasted):
* [**u-center**](https://www.u-blox.com/en/product/u-center) provides a graphical user interface for Windows. As of writing, _v22.07_ is the most recent version that supports the ZED-F9P chip.
* [**ubxtool**](https://gpsd.gitlab.io/gpsd/ubxtool.html) is a Linux/MacOS command line tool that is part of [GPSd](https://gpsd.gitlab.io/gpsd/). To get the full functionality for the ZED-F9P, you need a recent version (which at the time of writing was not available when installing from `apt` on Ubuntu 20). I built _GPSd v25.0_ from source following [these](https://gpsd.gitlab.io/gpsd/building.html) instructions:

```shell
$ sudo apt install scons
$ wget http://download.savannah.gnu.org/releases/gpsd/gpsd-3.25.tar.gz
$ tar -xzf gpsd-3.25.tar.gz
$ cd gpsd-3.25/
$ scons
$ sudo scons udev-install
$ export PYTHONPATH=${PYTHONPATH}:/usr/local/lib/python3/dist-packages
```
* Of course, you can also just figure out the binary representation of your configuration commands in [the u-blox F9 interface description](https://content.u-blox.com/sites/default/files/documents/u-blox-F9-HPG-1.32_InterfaceDescription_UBX-22008968.pdf) and just write them straight to your serial port `/dev/ttyACM0`.

## Enabling and disabling messages

The receiver can broadcast two different types of messages: **human-readable NMEA messages** and **binary UBX messages**.

For finding out which port is being used and what messages are currently published, you might use `screen` (`$ sudo apt-get install screen`):

```shell
$ screen /dev/ttyACM0
```

In case the serial port is not used by anyone else this should output periodic messages.

The human-readable NMEA messages (if published) will look as follows:

```shell
$GNGLL,5145.63813,N,00115.64255,W,180955.00,A,A*6A
```

corresponding to a measurement corresponding to 51° 45.63813" N und 1° 15.64255" W.

The binary messages will be displayed as special character sequences.

**This driver requires only the binary messages to be activated in order to work.** If the NMEA meesages are enabled the code will get stuck in an infinite loop inside the serial handler. You need to manually enable the binary messages and turn off NMEA messages in case the `screen` command outputs any human-readable messages.
Specifically, you need to **enable at least one of the following messages** (depending on your use case):

* **`UBX-RXM-RAWX`** for raw GNSS measurements from individual satellites.
* **`UBX-RXM-SFRBX`** for satellite navigation data broadcasted by the satellites.
* **`UBX-NAV-PVT`** for pre-computed GNSS fixes.

Since some of these messages consume a considerable amount of bandwidth, I suggest that you enebale only those that you require for your use case.

You can find a description of all messages in [the u-blox F9 interface description](https://content.u-blox.com/sites/default/files/documents/u-blox-F9-HPG-1.32_InterfaceDescription_UBX-22008968.pdf).

To change the current settings, install _u-center_ or _GPSd/ubxtool_, as described [above](#receiver-configuration).

If you use _u-center_, start the graphical user interface, connect the receiver via the left-most button, open the _Messages View_ and enable/disable messages by right-clicking on them.
Next, head to the _Configuration View_, select _CFG_, then _Save current configuration_, then _Send_ to save these settings to the flash on-board the F9P module.

If you use `ubxtool`, then first make sure that you can read and write to the receiver with:

```shell
$ ubxtool -p MON-VER
UBX-MON-VER:
  swVersion EXT CORE 1.00 (f10c36)
  hwVersion 00190000
  extension ROM BASE 0x118B2060
  extension FWVER=HPG 1.13
  extension PROTVER=27.12
  extension MOD=ZED-F9P
  extension GPS;GLO;GAL;BDS
  extension SBAS;QZSS
WARNING:  protVer is 10.00, should be 27.12.  Hint: use option "-P 27.12"
```

Depending on the version of `ubxtool` that you are using you might have to give the data values as hexadecimal (e.g. `0x01`) or decimal (e.g. `1` or `$((0x01))`) values.

If you use ubxtool, then you can use some or all of the following commands to **enable `UBX-NAV-PVT`, `UBX-RXM-SFRBX` and/or `UBX-RXM-RAWX`** (remember that you may not need all of them for your use case) and **disable all NMEA messages**:
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0x01,0x07,1` to enable `UBX-NAV-PVT`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0x02,0x13,1` to enable `UBX-RXM-SFRBX`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0x02,0x15,1` to enable `UBX-RXM-RAWX`.
* `for i in 0x0a 0x45 0x44 0x09 0x00 0x01 0x43 0x42 0x0d 0x40 0x47 0x06 0x02 0x07 0x03 0x0b 0x04 0x41 0x0f 0x05 0x08; do ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf0,$i,0; done` to disable all standard NMEA messages.

Finally **save the configuration** (make sure to use the correct serial interface):

````shell
$ ubxtool -f /dev/ttyACM0 -p SAVE
````

Other useful commands include:
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0x01,0x07,0` to disable `UBX-NAV-PVT`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0x02,0x13,0` to disable `UBX-RXM-SFRBX`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0x02,0x15,0` to disable `UBX-RXM-RAWX`.
* `for i in 0x0a 0x45 0x44 0x09 0x00 0x01 0x43 0x42 0x0d 0x40 0x47 0x06 0x02 0x07 0x03 0x0b 0x04 0x41 0x0f 0x05 0x08; do ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf0,$i,1; done` to enable all standard NMEA messages.
* `for i in 0x00 0x01 0x0d 0x02 0x04 0x05 0x08; do ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf7,$i,1; done` to enable all secondary NMEA messages.
* `for i in 0x00 0x01 0x0d 0x02 0x04 0x05 0x08; do ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf7,$i,0; done` to disable all secondary NMEA messages.
* `for i in 0x41 0x00 0x40 0x03 0x04; do ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf1,$i,1; done` to enable all u-blox NMEA messages.
* `for i in 0x41 0x00 0x40 0x03 0x04; do ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf1,$i,0; done` to disable all u-blox NMEA messages.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf0,0x00,1` to enable only the NMEA fix output (GGA).
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,0xf0,0x00,0` to disable only the NMEA fix output (GGA).
* `ubxtool -f /dev/ttyACM0 -p CFG-RATE,100` to set the measurement interval to 100 ms (= the measurement rate to 10 Hz). The default is 100 ms, but you might want to choose something else.

## Differential GNSS (D-GNSS)

If you want to obtain differential fixes in real-time (with potentially cm-accuracy), then you need to feed data from a nearby base station to the GNSS receiver.

If the **receiver has Bluetooth** (like the C099-F9P), then you can do the following:

* Pair the receiver with your phone via Bluetooth. The C099-F9P should have a name of the form *BT_C099-F9P_XXXX*. The receiver does not appear as a Bluetooth device? For the C099-F9P, you might have to enable it first: Plug in the receiver via USB, open a serial terminal (baud rate: 460800; serial frame: 8 bits, 1 stop bit, no parity; flow control: none; local echo off/disabled) and run the command `/bt_visible/run`.
* Install an NTRIP client (for example, [Lefebure](https://play.google.com/store/apps/details?id=com.lefebure.ntripclient&hl=en_GB&gl=US), [YCServer](https://play.google.com/store/apps/details?id=com.youcors.ycserver&hl=en_GB&gl=US), or [Bluecover](https://play.google.com/store/apps/details?id=pt.bluecover.ntripusb&hl=en_GB&gl=US)) on your phone.
* In the receiver settings, select the Bluetooth option and connect to the GNSS receiver.
* In the NTRIP caster section, select a base station. For example, the one near Bicester has the IP address 3.23.52.207 ([rtk2go.com](rtk2go.com)), the port 2101, the mount point / stream name OXTS1, and no username and no password. It uses the NTRIP Rev 1 protocol. You can find other base stations [in this table](http://rtk2go.com:2101). For example, you could search for stations with country code GBR, which provide at least GPS, Glonass (GLO), Galileo (GAL), and BeiDou (BDS) data. However, some streams might be password protected. Another option is to use one of the NTRIP broadcatsers from [the BKG](https://igs.bkg.bund.de/ntrip/). Their streams are listed [here](https://igs.bkg.bund.de/ntrip/streams) and the host URL is [igs-ip.net](igs-ip.net). For example, the station [LICC00GBR](https://igs.bkg.bund.de/api/collections/stations/items/LICC00GBR) is in London. However, you need to [register](https://register.rtcm-ntrip.org/cgi-bin/registration.cgi) first, which is free. Then, enter your username and password in your NTRIP client.
* Run the client and stream the data from the base to the receiver.
* Check that the flag `diff_soln` in the fixes in the `receiver_pvt` topic is `True` after your started to stream the data.

If the **receiver does not have Bluetooth**, but it is connected to a NUC and the **NUC has internet**, then follow the instructions in the section *Obtain RTK Solution (Optional)* below. 

Summary of steps here.

Install RTKLIB's NTRIP client:
```shell
git clone https://github.com/tomojitakasu/RTKLIB.git
cd RTKLIB/
git checkout rtklib_2.4.3
cd app/consapp/str2str/gcc/
make
```

Connect to the NTRIP stream of the base station:

```shell
./str2str -in ntrip://${NTRIP_SITE}:${NTRIP_PORT}/${MOUNT_POINT} -out tcpsvr://:3503
```

For example, for the stream from the base close to Bicester:

```shell
./str2str -in ntrip://3.23.52.207:2101/OXTS1 -out tcpsvr://:3503
```

Set the `input_rtcm` option to `1` in [config/driver_config.yaml](config/driver_config.yaml).

Launch the ROS node as usual:

```
roslaunch ublox_driver ublox_driver.launch
```

Again, check that the flag `diff_soln` in the fixes in the `receiver_pvt` topic is `True`.

To obtain differential fixes **offline** with post-processing, you can do the following:

* Log the stream from a base station during your trial with an NTRIP client of your choice.
* Convert the log into the RINEX format, if necessary.
* Convert the GNSS observations in your rosbag into RINEX format using [this tool](https://github.com/HKUST-Aerial-Robotics/GVINS-Dataset/blob/b2d7b8c546342b7b25b51d7138ba61932d22131b/toolkit/src/bag2rinex.cpp).
* Install [RTKLIB](http://rtklib.com/) or [this well-maintained fork](https://github.com/rtklibexplorer/RTKLIB/releases).
* Get the satellite navigation data from [here](https://igs.bkg.bund.de/root_ftp/IGS/BRDC/). Choose the correct year and day-of-the year and the file named `BRDM00DLR_S_YYYYDDD0000_01D_MN.rnx.gz`. (Alternatively, get the satellite navigation data from the base station, too.)
* Optionally, get SP3, CLK, ERP, and ION files from [here](https://igs.bkg.bund.de/root_ftp/IGS/products/orbits/). Choose the correct [GPS week number](https://www.ngs.noaa.gov/CORS/Gpscal.shtml) and the correct day of the week (Sunday=0, Monday=1, ...) to identify the correct directory and file. Files starting with `igs` are better than files starting `igr`, which are better than the `igu` files. Using these files should improve performance since they are more accurate than the broadcasted data mentioned above.
* Use RTKLIB's RTKPOST program with all these files to obatain a differential solution. If you are uncertain which options to choose, then have a look at the section in [the RTKLIB manual](http://www.rtklib.com/prog/manual_2.4.2.pdf) that describes RTKPOST. Probably the `Kinematic` mode is a good starting point.

# Assisted GNSS (A-GNSS)

You do not want to use differential GNSS, but want to reduce your time-to-first-fix to a few seconds? Then this section is for you. All you need is an internet connection at time of configuration.

The basic idea is to supply the receiver with assistance data, like satellite navigation data or an initial position during start-up. The first step is to download up-to-date satellite navigation data from the internet. The second step is to flash it to the receiver.

There are several options where to source the satellite navigation data. One option is [u-blox' AssistNow Service](https://developer.thingstream.io/guides/location-services/assistnow-user-guide). Data from AssistNow Online is valid for a few hours, data from AssistNow Offline up to 5 weeks. To use the services, you need a _token_. If based in the ORI, you can try to use [my token](https://drive.google.com/file/d/1AvLIvJe5UZBuxWclstPNm3llsoAr9bX-/view?usp=sharing). Otherwise, you can [register for one](https://www.u-blox.com/en/assistnow-service-evaluation-token-request-form). Then just connect your receiver via USB and do the following to download data from AssistNow Online and send it to the receiver's serial port:

```shell
token="your_token"
latitude="51.751920"
longitude="-1.258180"
accuracy="3000"
curl "https://online-live1.services.u-blox.com/GetOnlineData.ashx?token=$token;gnss=gps,glo,gal,bds,qzss;datatype=eph,alm,aux,pos;lat=$latitude;lon=$longitude;pacc=$accuracy" -o /dev/ttyACM0
```

The _latitude_, _longitude_, and _accuracy_ parameters are optional. In this example, I use the Carfax Tower as the initial location (specified in decimal degrees) and an uncertainity of this initial location of 3000 meter. For more parameters, check out [the user guide](https://developer.thingstream.io/guides/location-services/assistnow-user-guide).

The download size should be several kilobytes. If it's just a few bytes, then the request did not work. You can check the receiver's internal database with `ubxtool -f /dev/ttyACM0 -c 0x01,0x34`. There should be lots of satellites after transferring the assistance data. If you wish to reset the database for any reason, run `ubxtool -f /dev/ttyACM0 -p COLDBOOT`.

You can download data from AssistNow Offline with

```shell
curl "https://offline-live1.services.u-blox.com/GetOfflineData.ashx?token=$token;gnss=gps,glo,bds,gal"
```

However, I have not been able to succesfully use the data on a receiver yet.

Another option to load assistance data to your receiver is to briefly connect it to your phone (via USB or Bluetooth using, e.g., [YCServer](https://play.google.com/store/apps/details?id=com.youcors.ycserver&hl=en_GB&gl=US), [Bluecover](https://play.google.com/store/apps/details?id=pt.bluecover.ntripusb&hl=en_GB&gl=US), or [Lefebure](https://play.google.com/store/apps/details?id=com.lefebure.ntripclient&hl=en_GB&gl=US)) and to tap into an online NTRIP server stream, as described in the D-GNSS section above.



# ublox_driver

**Authors/Maintainers:** CAO Shaozu (shaozu.cao AT gmail.com)

The *ublox_driver* provides essential functionalities for u-blox GNSS receivers. This package is originally designed for [u-blox ZED-F9P module](https://www.u-blox.com/en/product/zed-f9p-module) according to the specification [UBX-18010854](https://www.u-blox.com/en/docs/UBX-18010854), but should also be compatible to other 8-series or 9-series u-blox receivers as long as the interface is the same.

The following diagram shows all possible input and output options supported by *ublox_driver*.

![ublox_driver diagram!](/figures/ublox_driver_diagram.svg "ublox_driver_diagram")

## 1. Prerequisites

### 1.1 C++11 Compiler
This package requires some features of C++11.

### 1.2 ROS
This package is developed under [ROS Kinetic](http://wiki.ros.org/kinetic) environment.

### 1.3 Eigen
We use [Eigen 3.3.3](https://gitlab.com/libeigen/eigen/-/archive/3.3.3/eigen-3.3.3.zip) for matrix manipulation.

### 1.4 Boost
Our software utilizes [Boost](https://www.boost.org/) library for serial and socket manipulation. Using command `sudo apt-get install libboost-all-dev` to install *Boost*.

### 1.5 gnss_comm
This package also requires [gnss_comm](https://github.com/HKUST-Aerial-Robotics/gnss_comm) for ROS message definitions and some utility functions. Follow [those instructions](https://github.com/HKUST-Aerial-Robotics/gnss_comm#1-prerequisites) to build the *gnss_comm* package.

## 2. Build ublox_driver
Clone the repository to your catkin workspace (for example `~/catkin_ws/`):
```
cd ~/catkin_ws/src/
git clone https://github.com/HKUST-Aerial-Robotics/ublox_driver.git
```
Then build the package with:
```
cd ~/catkin_ws/
catkin_make
source ~/catkin_ws/devel/setup.bash
```
## 3. Run with your u-blox receiver
Our software can take the serial stream from the u-blox receiver as an input source. Before running the package, you need to configure your receiver using [u-center](https://www.u-blox.com/en/product/u-center) to output at least `UBX-RXM-RAWX`, `UBX-RXM-SFRBX` and `UBX-NAV-PVT` messages to a specific serial port (a sample config used in our system can be found at *config/ucenter_config_f9p_gvins.txt*). Then connecting your computer(Linux) with the receiver, make sure the serial port appears as a file in the `/dev/` directory. Then add your account to `dialout` group to obtain permission on serial r/w operation via (no need to substitute $USER):
```
sudo usermod -aG dialout $USER
```
Open *config/driver_config.yaml*, set `online` and `to_ros` to 1, and adjust `input_serial_port` and `serial_baud_rate` according to your setting. Run the package with:
```
roslaunch ublox_driver ublox_driver.launch
```

Open another terminal, echo the ROS message by:
```
rostopic echo /ublox_driver/receiver_lla
```
If everything goes smoothly, there should be some ROS messages coming out. Note that some ROS topics remain inactive until the receiver gets GNSS signals. 

Besides the ROS message, you can record the serial stream to a file(`to_file` option) or forward the stream to another serial port(`to_serial` option). All output options can be turned on simultaneously.

### Obtain RTK Solution (Optional)
If your receiver owns an internal RTK engine (for example, ZED-F9P), you can input RTCM messages from the GNSS base station to the receiver in order to obtain the cm-level accurate RTK localization result. Our software can forward the RTCM stream from a local socket to the receiver's serial port. Nowadays many GNSS stations distribute their RTCM streams via [NTRIP protocol](https://en.wikipedia.org/wiki/Networked_Transport_of_RTCM_via_Internet_Protocol) and you can easily fetch the NTRIP data and map it to a local socket via [RTKLIB](http://www.rtklib.com/). Following those commands to build RTKLIB and setup the RTCM stream:
```
git clone https://github.com/tomojitakasu/RTKLIB.git
cd RTKLIB/
git checkout rtklib_2.4.3
cd app/consapp/str2str/gcc/
make
./str2str -in ntrip://${NTRIP_SITE}:${NTRIP_PORT}/${MOUNT_POINT} -out tcpsvr://:3503
```
Then set the `input_rtcm` option to `1` in *config/driver_config.yaml* and launch the ros node with:
```
roslaunch ublox_driver ublox_driver.launch
```
If the field `carr_soln` in `/ublox_driver/receiver_pvt` message becomes `2`, the RTK is in fix status. If you find the location of the GNSS base station reported in the RTCM message is somehow biased, you can apply correction via the variable `rtk_correction_ecef` in the config file.

## 4. Playback Log Files

Our package can also take an log file as the input. The log file can be recorded via [u-center](https://www.u-blox.com/en/product/u-center), RTKLIB or our package itself. To playback the log file, you need to set `online` to `0` and point `ubx_filepath` to your log file in the config file. Then launch the ros node with:
```
roslaunch ublox_driver ublox_driver.launch
```
Similar to the online receiver manner, all three output options are also supported in the playback mode. Note that the playback speed is controlled by the `serial_baud_rate` variable in the config file.


## 5. Synchronize System Time
In addition to message parsing and delivery, *ublox_driver* can also synchronize the local system time to the global time without the need of internet connection. Note that such synchronization is only in a coarse level and the accuracy is not guaranteed. To perform such synchronization, you need to set `UTC_OFFSET` macro in `src/sync_system_time.cpp` according to your timezone. Recompile and launch the driver with:
```
roslaunch ublox_driver ublox_driver.launch
```
Then open another terminal and run commands:
```
sudo su
source /opt/ros/kinetic/setup.bash
source ${YOUR_CATKIN_WORKSPACE}/devel/setup.bash
rosrun ublox_driver sync_system_time
```
The system time will be synchronized to the global time when the receiver gets a valid PVT solution.

## 6. Todo List

- [ ] Config u-blox receiver when the driver gets launched
- [ ] Infer timezone from geodetic location when sync time
- [ ] Optimize time-sync function to improve precision

## 7. Acknowledgements
Many of the ephemeris parsing functions in our package are adapted from [RTKLIB](http://www.rtklib.com/). We use [mini-yaml](https://github.com/jimmiebergmann/mini-yaml) for config parsing.

## 8. License
The source code is released under [GPLv3](https://www.gnu.org/licenses/gpl-3.0.html) license.
