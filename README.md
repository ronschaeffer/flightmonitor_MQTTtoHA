######
This fork uses mqtt-cli instead of mosquitto, and MQTT broker settings are hardcoded. mqtt-cli: https://www.hivemq.com/blog/mqtt-cli/
######

# flightmonitor_MQTTtoHA

## Introduction
The objective of this project is to display in [Home Assistant](https://www.home-assistant.io) some basic information from a system running an ADS-B receiver (dump1090-fa) that feeds servers [Fligtradar24](https://www.flightradar24.com) and/or [FlightAware](https://www.flightaware.com).
The bash script collects information and publish JSON MQTT messages. To ease the integration with Home Assistant, MQTT discovery messages declaring all sensors supported by the script are also published when the script/service starts.
Below is a screenshot of a Home Assistant tab on which are dislayed the sensors values handled by the script (titles, weblinks and history graphs were added manually).
![Screenshot of Home Assistant tab with the sensors handled by the script](/images/screenshot_sensors_inHA.png)

## Installation
The script has been written with the assumption of a standard installation of [FlightAware](https://www.flightaware.com) and [Fligtradar24](https://www.flightradar24.com) programs, according to:
* [this page](https://flightaware.com/adsb/piaware/install) for FlightAware.
  * In addition, piaware has been configured with receiver type set to 'beast' with the command: `sudo piaware-config receiver-type beast`.
* [this page](https://www.flightradar24.com/share-your-data) for Flightradar24 (download and run bach script `install_fr24_rpi.sh`)

The script is a bash script with few dependencies: bc, jq, curl and mosquitto-clients. If not already on your machine, they can be installed by:
`sudo apt-get install bc jq curl mosquitto-clients`
### Installation steps:
1. Clone this repository or copy/download the scripts on the Rapsberry Pi running ADS-B receiver (dump1090-fa) and the feeders (fr24feed and/or piaware)
2. Edit the script file and set the values according to your configuration:
  * configuration of the MQTT broker: IP address, port, username and password (if needed)
  * configuration of the topics where are published the messages. For instance, if you do not use fr24feed, set fr24feed_subtopic to an empty string (`fr24feed_topic=""`) or comment the line (` #fr24feed_subtopic="fr24feed"`).
  * configuration of Home Assistant: 
    * the topic prefix for discovery (by default `discovery_prefix`="homeassistant" as mentionned [here](https://www.home-assistant.io/docs/mqtt/discovery/))
    * a suffix (for instance the machine nickname "_RPi4-Kitchen") that is appended to the unique identifier of the sensors (in case the script runs on several machines connected to the same instance of Home Assistant)
    * `use_device=0` or `use_device=1`. If `use_device` is equal to 1, the data associated with a source (i.e. fr24feed, dump1090-fa or piaware) are declared as entities linked to a device in Home Assistant. The device name is equal to the MQTT sub-topic appended with the `unique_id_suffix`. This option eases the integration in Home Assistant: when the device is selected in the 'Configuration' menu, an entity-card with all linked entities can be directly added to one of your panel. A screnshot is displayed below (with the parameters `dump1090_subtopic="dump1090"` and `unique_id_suffix="_OdroidXU4"`):
    ![Add entities in Lovelace with the parameter use_device set to 1](/images/use_device_1_add_to_lovelace.png)
  * configuration of the execution of the script
    * `pub_discovery="yes"` to publish the MQTT discovery messages to Home Assistant at the begining of the execution of the script, `pub_discovery="no"` not to publish them
    * `run_mode="loop"` to publish the status messages periodically, `run_mode="once"` to publish them once and exit, `run_mode="no"` not to publish them
    * `update_rate=60` which define the delay between the status messages (with `run_mode="loop"`)
    
    The three above parameters can be overwritten by launching the script with the arguments 
    *  `-d [yes|no]` to overwrite `pub_discovery`
    *  `-r [loop|once|no]` to overwrite `run_mode`
    *  `-t nn` to overwrite `update_rate`
3. Launch the script `flightmonitor_MQTTtoHA.sh` and check new entities are available in Home Assistant
  * `bash flightmonitor_MQTTtoHA.sh`
  * or `./flightmonitor_MQTTtoHA.sh` in case you previously set the execution permission to the file (`chmod a+x flightmonitor_MQTTtoHA.sh`)
 
 4. When the above step is successful, you can run the install script:
 `sudo bash install_service.sh`
 It will create a file `flightmonitor_MQTTtoHA.service` which is copied to `/etc/systemd/system` directory and launch the service. The status of the service is displayed at the end of the install script. By default, the service is configured to send status messages every minute. This value can be changed by running the `install_service.sh` script with argument `-t` followed by the value in seconds. For example, for an update every 5 minutes: `sudo bash install_service.sh -t 300`.
 
 The status of the service is reflected in Home Assistant: sensors will be marked "unavailable" if the service is stopped (or when the script in "loop" mode is stopped with Ctrl+C)
  
 | active       | stopped    |
 |--------------|------------|
 ![Sensors available (service: active)](/images/ServiceStart-SensorsAvailable.png) | ![Sensors unavailable (service: stop)](/images/ServiceStop-SensorsUnavailable.png)
 
## Principles
### dump1090-fa
The parameters of the MQTT message are derived from the JSON file `/run/dump1090-fa/aircraft.json` (which is also used to display the aircrafts on the map @ http://127.0.0.1:8080). For example:
```JSON
{
    "problem":"OFF",
    "total_aircraft":"14",
    "aircraft_with_positions":"11"
}

```
In Home Assistant,
 * the fields `problem` is declared as binary sensors with device class "problem".
 * the fields `total_aircraft` and `aircraft_with_positions` are declared as sensors.

### fr24feed
The parameters of the MQTT message are derived from the JSON file `http://127.0.0.1:8754/monitor.json` (which is also used for status available @ http://127.0.01:8754). For example:
```JSON
{
    "problem":"OFF",
    "connection":"ON",
    "mlat_problem":"OFF",
    "lastACsent":"1",
    "numACtracked":"14",
    "numACuploaded":"9"
}
```
In Home Assistant,
 * the fields `problem` and `mlat_problem` are declared as binary sensors with device class "problem".
 * the field `connection` is declared as binary sensors with device class "connectivity".
 * the fields `lastACsent`, `numACtracked` and `numACuploaded` are declared as sensors.

### piaware
The parameters of the MQTT message are derived from the output of the command `piaware-status`. For example:
```json
{
    "piaware_problem":"OFF",
    "faup1090_problem":"OFF",
    "faup978_problem":"ON",
    "mlat_problem":"OFF",
    "dump1090_problem":"OFF",
    "faup1090dump1090_connection":"ON",
    "piawareserver_connection":"ON",
    "data3005_problem":"OFF"
}
```
 In Home Assistant, 
  * the fields `piaware_problem`, `faup1090_problem`, `faup978_problem`, `mlat_problem`, `dump1090_problem` and `data3005_problem` are declared as binary sensors with device class "problem".
  * the fields `faup1090dump1090_connection` and `piawareserver_connection` are declared as binary sensors with device class "connectivity".

### Home Assistant
When the script or service is laucnhed, the MQTT discovery messages are published.

## Problems and investigations
This script has been tested on
 * a Raspberry Pi, model 2+ running [RaspiOS](https://www.raspberrypi.org/software/operating-systems/) distribution `2021-05-07-raspios-buster-armhf-lite`
 * an Odroid-XU4 running [DietPi](https://www.dietpi.com) distribution `DietPi_OdroidXU4-ARMv7-Buster`

In case it does not work properly on your system, below is a list of suggestions to investigate the issues.
  When debugging, it is recommanded to lower the value of the `update_rate` variable to few seconds (e.g.`update_rate=5`).
 * After each change to the script, restart the service `sudo systemctl restart flightmonitor_MQTTtoHA`
 * After each change to the service file (`/etc/systemd/system/flightmonitor_MQTTtoHA.service`)
   * relaunch systemd: `sudo systemctl daemon-reload`
   * restart the service: `sudo systemctl restart flightmonitor_MQTTtoHA`
 1. Check the file `flightmonitor_MQTTtoHA.service` can be found in `/etc/systemsd/system` directory
 2. Check service status: `sudo systemctl status flightmonitor_MQTTtoHA`
 3. Check MQTT messages are publish in the topics configured by the variables at the begining of the script. For instance assuming `mqtt_topic_prefix="flightmonitor"` and `dump1090_subtopic="dump1090"` and that the MQTT borker runs on the same machine:
 `mosquitto_sub -h 127.0.0.1 -p 1883 -t flightmonitor/dump1090`. 
 4. Check MQTT discovery messages are published when the service start/re-start. For instance:
 `mosquitto_sub -h 127.0.0.1 -p 1883 -t homeassistant/# -v`
 
 
