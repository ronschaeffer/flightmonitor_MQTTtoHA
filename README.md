# flightmonitor_MQTTtoHA

The objectives of this project is to display in [Home Assistant](https://www.home-assistant.io) some basic information from a system running an ADS-B receiver (dump1090-fa) that feeds servers [Fligtradar24](https://www.flightradar24.com) and [FlightAware](https://www.flightaware.com).
The bash script collects information and publish JSON MQTT messages. To ease the integration with Home Assistant, MQTT discovery messages declaring all sensors supported by the script are also published when the script/service starts.
Below is a screenshot of a Home Assistant tab on which are dislayed the sensors values handled by the script.
![Screenshot of Home Assistant tab with the sensors handled by the script](/images/screenshot_sensors_inHA.png)

## Installation
The script has been written with the assumption of a standard installation of [FlightAware](https://www.flightaware.com) and [Fligtradar24](https://www.flightradar24.com) programs, according to:
* [this page](https://flightaware.com/adsb/piaware/install) for FlightAware
* [this page]() for Flightradar24 (option 2)

The script is a bash script with few dependencies: bc, jq and mosquitto-clients. If not already on your machine, they can be installed by:
`sudo apt-get install bc jq mosquitto-clients`
### Installation steps:
1. Clone this repository or copy/download the scripts on the Rapsberry Pi running ADS-B receiver (dump1090-fa) and the feeders (fr24feed and/or piaware)
2. Edit the script file and set the values according to your configuration:
  * configuration of the MQTT broker: IP address, port, username and password (if needed)
  * configuration of the topics where are published the messages. For instance, if you do not use fr24feed, set fr24feed_subtopic to an empty string (`fr24feed_topic=""`) or comment the line (` #fr24feed_subtopic="fr24feed"`).
  * configuration of Home Assistant: 
    * the topic prefix for discovery (by default `discovery_prefix`="homeassistant" as mentionned [here](https://www.home-assistant.io/docs/mqtt/discovery/))
    * a suffix (for instance the machine nickname "_RPi4-Kitchen") that is appended to the unique identifier of the sensors (in case the script runs on several machines connected to the same instance of Home Assistant)
    * `use_device=0` or `use_device=1`. If `use_device` is equal to 1, the entities linked with a source (i.e. fr24feed, dump1090-fa or piaware) are linked to a device in Home Assistant. The device name is equal to the MQTT sub-topic appended with the `unique_id_suffix`. This option eases the integration in Home Assistant: when the device is selected in the 'Configuration' menu, an entity-card with all linked entities can be directly added to one of your panel.
3. launch the script and check new entities are available in Home Assistant
  * `bash flightmonitor_MQTTtoHA.sh`
  * or `./flightmonitor_MQTTtoHA.sh` in case you previously set the execution permission to the file (`chmod a+x flightmonitor_MQTTtoHA.sh`)
 
 When the above step is successful, you can run the install script:
 `sudo bash install.sh`
 It will create a file `flightmonitor_MQTTtoHA.service` which is copied to `/etc/systemd/system` directory and launch the service. The status of the service is displayed at the end of the install script. The status of the service is reflected in Home Assistant: sensors will be marked "unavailable" if the service is stopped.
 
 ## Problems and investigations
 This script has been tested on
 * a Raspberry Pi, model 2+ running RaspiOS distribution `2021-05-07-raspios-buster-armhf-lite`
 * an Odroid-XU4 running [DietPi](https://www.dietpi.com) distribution `DietPi_OdroidXU4-ARMv7-Buster`

In case it does not work properly on your system, belw is a list of suggestions to investigate the issues.
  When debugging, it is recommanded to lower the value of the `update_rate` variable to few seconds (e.g.`update_rate=5`).
 * After each change of the script, do not forget to restart the service `sudo systemctl restart flightmonitor_MQTTtoHA`
 * After each change of the service file (`/etc/systemd/system/flightmonitor_MQTTtoHA.service`), do not forget to relaunch systemd:  
 `sudo systemctl daemon-reload`  
 `sudo systemctl restart flightmonitor_MQTTtoHA`
 1. Check the file `flightmonitor_MQTTtoHA.service` can be found in `/etc/systemsd/system` directory
 2. Check service status: `sudo systemctl status flightmonitor_MQTTtoHA`
 3. Check MQTT messages are publish in the topics configured by the variables at the begining of the script. For instance assuming `mqtt_topic_prefix="flightmonitor"` and `dump1090_subtopic="dump1090"` and that the MQTT borker runs on the same machine:
 `mosquitto_sub -h 127.0.0.1 -p 1883 -t flightmonitor/dump1090`. 
 3. Check MQTT discovery messages are published when the service start/re-start. For instance:
 `mosquitto_sub -h 127.0.0.1 -p 1883 -t homeassistant/# -v`
 
 
