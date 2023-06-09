# ROS driver for the u-blox ZED-F9P GNSS receiver

*Brief setup instructions by Jonas Beuchert, original readme [below](#ublox_driver).*

You need a C++11 compiler, ROS, Eigen 3.3.3, and Boost, but they might be installed already.

Install Google's glog library for message output:

```shell
sudo apt-get install libgoogle-glog-dev
```

Clone [gnss_comm](https://github.com/HKUST-Aerial-Robotics/gnss_comm) for ROS message definitions:

```shell
cd spot_git
git clone git@github.com:HKUST-Aerial-Robotics/gnss_comm.git
```

Clone this repository:

```shell
git clone git@github.com:ori-drs/ublox_driver.git
```

Depending on you desired configuration, you might need to change [the configuration file](/config/driver_config.yaml), but it should work out-of-the-box. Sometimes, you need to change the port. Usually, it is `dev/ttyACM0` (default) or `/dev/ttyACM1`.

Create symbolic links from you catkin workspace to the cloned repositories

```shell
cd ../spot_ws/src
ln -s ~/spot_git/gnss_comm/ .
ln -s ~/spot_git/ublox_driver/ .
```

Build the driver using catkin tools:

```shell
cd ..
catkin build ublox_driver
```

Launch ```roscore``` in another terminal.

Grant permission to the port, e.g:

```shell
sudo chmod 666 /dev/ttyACM0
```

(Alternatively, add the user permanently to the `dialout` group with `sudo adduser user_name dialout` and logout and login again.)

Launch the driver:

```shell
source devel/setup.bash
roslaunch ublox_driver ublox_driver.launch
```

It might be worth it to check out [Section 5.](#5-synchronize-system-time) below to synchronize the local system time with a global time reference (and, therefore, with the timestamps of the GNSS observations.)

## Troubleshooting

If the driver prints the error `ubx rxmrawx week=0 error` or the error `open: No such file or directory`, a cause can be that the port in [the configuration file](/config/driver_config.yaml) is incorrectly set.

If a *permission-denied* error occurs, then the driver might not have access rights to the port.

If no messages appear despite there being view of the sky, you might have to manually enable the binary messages of the receiver, see [below](#enabeling_and_disabling_messages).

## Receiver configuration

You can use different software to change the configuration of the receiver firmware (e.g., the measurement frequency or which messages are broadcasted):
* [u-center](https://www.u-blox.com/en/product/u-center) provides a graphical user interface for Windows. As of writing, _v22.07_ is the most recent version that supports the ZED-F9P chip.
* [ubxtool](https://gpsd.gitlab.io/gpsd/ubxtool.html) is a Linux/MacOS command line tool that is part of [GPSd](https://gpsd.gitlab.io/gpsd/). To get the full functionality for the ZED-F9P, you need a recent version. I built _GPSd v25.0_ from source following [these](https://gpsd.gitlab.io/gpsd/building.html) instructions and it worked.

## Enabeling and disabling messages

The receiver can broadcast two types of messages: human-readable NMEA messages and binary UBX messages.
This driver needs the binary messages.
If only NMEA meesages are enabled, you need to manually enable the binary messages (and, optionally, turn NMEA messages off).
Specifically, you need to enable one or more of the following messages (depending on your use case):
* `UBX-RXM-RAWX` for raw GNSS measurements from individual satellites.
* `UBX-RXM-SFRBX` for satellite navigation data broadcasted by the satellites.
* `UBX-NAV-PVT` for pre-computed GNSS fixes.
You can find a description of all messages [in the u-blox F9 interface description
](https://content.u-blox.com/sites/default/files/documents/u-blox-F9-HPG-1.32_InterfaceDescription_UBX-22008968.pdf)
To enable one or more of these, install _u-center_ or _GPSd/ubxtool_, as described [above](#receiver-configuration).
If you use u-center, start the graphical user interface, connect the receiver via the left-most button, open the _Messages View_ and enable/disable messages by right-clicking on them.
Next, head to the _Configuration View_, select _CFG_, then _Save current configuration_, then _Send_ to save these settings to the flash on-board the F9P module.
If you use ubxtool, then you can use some or all of the following commands:
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,1,7,1` to enable `UBX-NAV-PVT`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,1,7,0` to disable `UBX-NAV-PVT`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,2,19,1` to enable `UBX-RXM-SFRBX`.
* `ubxtool -f /dev/ttyACM0 -p CFG-MSG,2,19,0` to disable `UBX-RXM-SFRBX`.
* `ubxtool -f /dev/ttyACM0 -e RAWX` to enable `UBX-RXM-RAWX`.
* `ubxtool -f /dev/ttyACM0 -d RAWX` to disable `UBX-RXM-RAWX`.
* `ubxtool -f /dev/ttyACM0 -e NMEA` to enable NMEA messages.
* `ubxtool -f /dev/ttyACM0 -d NMEA` to disable NMEA messages.
* 
Afterwards, run `ubxtool -f /dev/ttyACM0 -p SAVE`.

## Differential GNSS

If you want to obtain differential fixes in real-time (with potentially cm-accuracy), then you need to feed data from a nearby base station to the GNSS receiver.

If the **receiver has Bluetooth** (like the C099-F9P), then you can do the following:

* Pair the receiver with your phone via Bluetooth. The C099-F9P should have a name of the form *BT_C099-F9P_XXXX*. The receiver does not appear as a Bluetooth device? For the C099-F9P, you might have to enable it first: Plug in the receiver via USB, open a serial terminal (baud rate: 460800; serial frame: 8 bits, 1 stop bit, no parity; flow control: none; local echo off/disabled) and run the command `/bt_visible/run`.
* Install an NTRIP client (for example, [the Lefebure NTRIP Client](https://play.google.com/store/apps/details?id=com.lefebure.ntripclient&hl=en_GB&gl=US)) on your phone.
* In the receiver settings, select the Bluetooth option and connect to the GNSS receiver.
* In the NTRIP caster section, select a base station. For example, the one near Bicester has the IP address 3.23.52.207 ([rtk2go.com](rtk2go.com)), the port 2101, the mount point / stream name OXTS1, and no username and no password. It uses the NTRIP Rev 1 protocol. You can find other base stations [in this table](http://rtk2go.com:2101). For example, you could search for stations with country code GBR, which provide at least GPS, Glonass (GLO), Galileo (GAL), and BeiDou (BDS) data. However, some streams might be password protected. Another option is to use one of the NTRIP broadcatsers from [the BKG](https://igs.bkg.bund.de/ntrip/). Their streams are lsited [here](https://igs.bkg.bund.de/ntrip/streams). For example, the station [LICC00GBR](https://igs.bkg.bund.de/api/collections/stations/items/LICC00GBR) is in London. However, you need to [register](https://register.rtcm-ntrip.org/cgi-bin/registration.cgi) first, which is free. Then, enter your username and password in your NTRIP client.
* Run the client and stream the data from the base to the receiver.
* Check that the flag `diff_soln` in the fixes in the `receiver_pvt` topic is `True` after your started to stream the data.

If the **receiver does not have Bluetooth**, but the **NUC has internet**, then follow the instructions in the section *Obtain RTK Solution (Optional)* below. 

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

