# Honeywell/Ademco Vista ECP ESPHome custom component and library

## Table of contents

- [Honeywell/Ademco Vista ECP ESPHome custom component and library](#honeywell-ademco-vista-ecp-esphome-custom-component-and-library)
  * [About the project](#about-the-project)
  * [Features](#features)
- [How to install](#how-to-install)
  * [⚠️ Warning!!! ⚠️](#---warning------)
  * [Prerequisites](#prerequisites)
    + [Identify Vista panel model](#identify-vista-panel-model)
  * [Project Structure and Wiring](#project-structure-and-wiring)
  * [Install ESPHome on the ESP device](#install-esphome-on-the-esp-device)
  * [Flash project into ESP device](#flash-project-into-esp-device)
  * [Connecting everything](#connecting-everything)
- [Miscellaneous](#miscellaneous)
  * [Example in Home Assistant](#example-in-home-assistant)
    + [Sample sensor configuration for card using mqtt](#sample-sensor-configuration-for-card-using-mqtt)
  * [HA Services](#ha-services)
  * [OTA updates](#ota-updates)
  * [MQTT integration](#mqtt-integration)
  * [Setting up the alarm panel keyboard card on HA](#setting-up-the-alarm-panel-keyboard-card-on-ha)
- [FAQ](#faq)
  * [Supported Models](#supported-models)
- [Report issue](#report-issue)
- [References](#references)
- [Author & Contributors](#author---contributors)

## About the project

This project allows you to monitor and control you old vista alarm panel over the internet through a cheap ESP device and an home automation platform*.

\* *Home Assistant or any other automation platform supporting MQTT protocol.*<br>

Got a DSC PowerSeries panel? Have a look at the <a href="https://github.com/Dilbert66/esphome-dsckeybus">following project</a>.<br>

This is an implementation of an ESPHOME custom component and ESP Library to interface directly to a Safewatch/Honeywell/Ademco Vista 15/20 alarm system using the ECP interface and very inexpensive ESP8266/ESP32 modules .  The ECP library code is based on the arduino source code from Mark Kimsal's repository located at  https://github.com/TANC-security/keypad-firmware.  It has been  completely rewritten as a class and adapted to work on the ESP8266/ESP32 platform using interrupt driven communications and pulse timing. A custom modified version of Peter Lerup's ESPsoftwareserial library (https://github.com/plerup/espsoftwareserial) was also used for the serial communications to work more efficiently within the tight timing of the ESP8266 interrupt window. 

From documented info, it seems that some panels send an F2 command with extra system details but the panel I have here (Vista 20P version 3.xx ADT version) does not.  Only the F7 is available for zone and system status in my case but this is good enough for this purpose. 

As far as writing on the bus and the request to send pulsing sequence, most documentation only discusses keypad traffic and this only uses the the 3rd pulse.  In actuality the pulses are used as noted below depending on the device type requesting to send:
```
Panel pulse 1. Addresses 1-7, expander board (07), etc
Panel pulse 2. Addresses 8-15 - zone expanders, relay modules
Panel pulse 3. Addresses 16-23 - keypads
```
For example, a zone expander that has the address 07, will send it's address on the first pulse only and will send nothing for the 2nd and 3rd pulse.  A keypad with address 16, will send a 1 bit pulse for pulse1 and pulse2 and then it's encoded address on pulse 3. This info was determined from analysis using a zone expander board and Pulseview to monitor the bus. 

If you are not familiar with ESPHome , I suggest you read up on this application at https://esphome.io and home assistant at https://www.home-assistant.io/.   The library class itself can be used outside of the esphome and home assistant systems.  Just use the code as is without the vistalalarm.yaml and vistaalarm.h files and call it's functions within your own application.  

To use this software you simply place the one of the yaml configuration files from the src directory to your main esphome directory, renaming to something of your choosing, then copy the *.h and *.cpp files from the `vistaEcpInterface` directory to a similarly named subdirectory (case sensitive) in your esphome main directory and then compile the yaml as usual. The directory name is in the "includes:" option of the yaml.  

There are 3 versions of the yaml configurations provided.  Pick one and modify to your setup.

1. `vistaAlarm_binary_sensors.yaml` uses dual state binary sensors to indicate the zone statuses.  IE. open/closed.   
2. `vistaAlarm_multi_partition.yaml` provides an example of  multi partition support  as well as useing text sensors for the zone statuses. I.e. open/closed/alarmed/bypassed.
3. `vistaAlarm_single_partition.yaml` provices a simpler single partition support as well as using text sensors for all zones.

For multi partition support, it is important that you first program new keypad addresses assigned to each partiton you want to support using programs `*190` - `*196` on the vista panel.  Once done, assign these addresses to keypadaddr1 (partition1) , keypadaddr2 (partition2), keypaddr3 (partition3).  For unused partitions, leave the associated keypadaddrx config line at 0. 


**Read more in the next wiki paragraphs for a detailed step by step guide.**

## Features


* Full zone expander emulation (4219/4229) which will give you  an additional 8 zones to the system per emulated expander plus associated relay outputs. Currently the library will provide emulation for 2 boards for a total of 16 additionals zones. You can even use free pins on the chip as triggers for those zones as well. 

* Full independent partition support. The firmware allows control and view status of all 3 partitions.

* Relay module emulation. (4204). The system can support 4 module addresses for a total of 16 relay channels. 

* Long Range Radio (LRR) emulation (or monitoring) statuses for more detailed status messages

* Zone status - Open, Alarmed, Closed and Bypass with named zones

* Arm, disarm or send any sequence of commands to the panel for any partition

* Status indicators - fire, alarm, trouble, armed stay, armed away, instant armed, armed night,  ready, AC status, bypass status, chime status,battery status, check status, zone and relay channel status field of individual partitions.


* Optional ability to monitor other devices on the bus such as keypads, other expanders, relay boards, RF devices, etc. This requires the #define MONITORTX to be uncommented in vista.h as well as the addition of two resistors (R4 and R5) to the circuit as shown in the schematic.   This adds another serial interrupt routine that captures and decodes all data on the green tx line.  If enabled this data will be used to update zone statuses for external modules.


The following services are published to home assistant for use in various scripts. 
```
alarm_disarm: Disarms the alarm with the user code provided, or the code specified in the configuration.
alarm_arm_home: Arms the alarm in home mode.
alarm_arm_away: Arms the alarm in away mode.
alarm_arm_night: Arms the alarm in night mode (no entry delay).
alarm_trigger_panic: Trigger a panic alarm.
alarm_trigger_fire: Trigger a fire alarm.
alarm_keypress: Sends a string of characters to the default partition. (Controlled by the keypad1 address)
alarm_keypress_partition: Sends a string of characters to any specified partition. (1, 2 or 3)
```

# How to install

## ⚠️ Warning!!! ⚠️

⚠️ **When opening the panel and dealing with the alarm panel itself throughout the reading of this wiki be careful as AC current is used to power the panel, make sure to switch it off before touching the terminals or any other part of the circuits.** ⚠️

##  Prerequisites

In order to install the project you need to:
- [Identify your Vista alarm panel model](#identify-vista-panel-model)
- Make sure to have a running instance of Home Assistant or any other home automation platform supporting MQTT protocol (or something else, but able to listen and publish on MQTT topics) // TO BE CHECKED
- [Get the needed components for the project and wire them](#project-structure-and-wiring)
- [Install ESPHome on the ESP device](#install-esphome-on-the-esp-device)
- Flash the project on the ESP device
- [Connect everything](#connecting-everything)


### Identify Vista panel model
Open the panel and check the interior side of the panel's door, there should be the wiring diagram indicating the Vista model right below (usually on bottom right or center). <br>

*Example: Vista-20P, Vista-20SE, Vista-20SEa, Vista-25IT, ...*<br>

If the wiring diagram is not present, check the circuit board. You should be able to see what model it is by looking either at the stamps/labels on the circuit board or by looking at the model on the CPU. The model here sometimes is in form of "SAVS**20P**5" or similar.<br> It is recommended to check also the circuit board even if you found the model on the wiring diagram, just to be sure that they match.

See image below, red arrows indicates where it could be.

![Vista Model Circuit Board IMG](readme_material/vista_panel_door_3d.png)

Check [Supported Models](#supported-models) to get an overview of compatible models and continue to next section. 

If your device is not present in the list you could still try using one of the code versions available in the repository, **but the functioning is not guaranteed**.

Got a DSC PowerSeries panel? Have a look at the <a href="https://github.com/Dilbert66/esphome-dsckeybus">following project</a>.

##  Project Structure and Wiring

After you identified the vista panel's model then you can start preparing for the hardware part and the first HW part you need to get is an ESP device. <br>

If you're starting from scratch, probably it's better to start with an ESP32 as it's more powerful and versatile, otherwise make your evaluation based on the code you select to install on the ESP device. Let's analyze the project structure.<br>

Currently the project is structured as follows:
- `master` branch: stable code - **recommended**
- `dev` branch: more updated code with new features, but WIP - **stability and functioning are not guaranteed**

Both branches divides the source code to be flashed on the ESP device into two folders, one for `Vista20P-like` and one for `Vista20SE-like` panels:
1. `src/vistaEcpInterface`
2. `src/Vista20SE`

For both branches you have the possibility to install them using Home Assistant or MQTT.<br>

If you have an Home Assistant instance running and you want to be able to fully control and monitor the panel through HA you will have to use the ESPHome yaml files for Home Assistant inside `src` folder. <br>
For the `master` branch there is only one version of the yaml file (dev will be migrated to master soon), but for the `dev` one there are currently 3 yaml files:
1. `vistaAlarm_binary_sensors.yaml` uses dual state binary sensors to indicate the zone statuses. I.e. open/closed.
2. `vistaAlarm_multi_partition.yaml` provides an example of multi partition support as well as useing text sensors for the zone statuses. I.e. open/closed/alarmed/bypassed.
3. `vistaAlarm_single_partition.yaml` provides a simpler single partition support as well as using text sensors for all zones.

If instead you don't want to use Home Assistant, you can still pick one of the source code above, but you will have to use the sketch files inside the `MQTT-Example` folder and setup your home control software to process the MQTT topics.<br> // TO BE CHECKED

Select the branch and configuration/source code files you need.<br>

If you opted for the **MQTT example** or a **multi-partition** code setup, then it's recommended to get an ESP32 as it's more powerful and the resulting software will consume more memory, otherwise up to you. // TO BE CHECKED

Get the right ESP device. If you're unsure on what ESP model you need to get, then you can document at <a href="https://esphome.io/">ESPHome website</a>. *In theory*, any ESP device should work with ESPHome. Just make sure you have the GPIO numbers pin mapping, sometimes the pins don't match the ones reported in the pcb's silkscreen.<br>
*Example for the Wemos D1 R2 -> <a href="https://cyaninfinite.com/getting-started-with-the-wemos-d1-esp8266-wifi-board/">link</a>*

// REVIEW-COMMENT: Not sure if this is applicable for the MQTT example too as we will not use ESPHome

Now that you identified the panel, selected a source code and got an ESP device, it's time to choose a wiring configuration to the panel.

The available ones in the repo are:
1. Non-isolated simple version (**preferred**)

![ecpinterface](readme_material/master_non_is_simple.png)

Notes: This is the recommended version. It provides the best signal output with very minimal impact on the ECP bus.


2. Alternative Non-isolated simple version (preferred)

![ecpinterface-noopto](readme_material/master_noopto.png)

3. Ground isolated version

![ecpinterface-isolated](readme_material/master_gnd_isolated.png)

Notes: This version is the least recommended as it will load the ECP bus to a certain extent. It also does not provide the best signal output. This is all dependant on the bus it is connected to and the quality of the optocouplers used. Your mileage may vary.<br><br>
* Optocouplers should have a minimum CTR of 50. Recommendations are the 4N35 or TLP521. You can vary the resistor values for the simple version but keep the ratio similar for the voltage dividers R2/R3 and R4/R5. R1 should not be set below 150 ohm or 100ohm when using an ESP32. Resistor values are chosen to minimize load on ECP bus while still providing full output signals on the optocouplers. <br>
* My goal was to keep the design as simple as possible without causing any bus load or interference with maximum signal fidelity.  Since the transmit circuit required high side switching I opted to use an optocoupler since I had a few on hand and it simplified the amount of components needed but proved to have it's own issues as far as CTR requirements.  For those that would prefer not using optocouplers due to availability or other reasons, i've provided a version using transistors for the transmit circuit instead. <br>

The first two are the same, but the alternative versions use transistors instead of optocouplers.
The third one instead is the ground isolated version, meaning that the ground of the panel (black wire) and the ground of the ESP device are separated. There is no particular advantage of using it.<br> // TO BE CHECKED
Read the notes on the drawings for more details.

Get the other components: 
- breadboard or PCB (if you prefer soldering)
- jumper cables
- 4-inputs screw terminal for the cables from the panel to the ESP (opt)
- resistors (you can use multiple resistors in series or parallel if you can't find kits with the exact value of the diagrams, i.e. 4.7KOhm can be obtained by using two resistors in parallel -> 100kohm + 5.1kohm =~ 4.8KOhm)
- optocouplers / transistors
- power supply or USB for the ESP device (highly recommended) or a 12V to 5V/3.3V regulator (to take the power from the panel)
- juction box (optional, to protect the ESP device if installed outside the panel)

Thanks to <a href="https://community.home-assistant.io/u/howardchen3">`@howardchen3`</a> there is a PCB layout of the ground-isolated version available at <a href="https://oshwlab.com/boobeechen/vistaesphome_copy">oshwlab</a> that could be used to send to a manufacturer. Read more on the site.


Wire the breadboard/PCB part according the wiring diagram you selected, but for the moment keep the ESP unplugged from both panel and breadboard/pcb since we're going to connect the ESP to the PC and it's not handy to have all the wires around. 

You should be in a situation like this now:
![ecp_breadboard](readme_material/esp_breadboard.png)

Esp is ready to be connected to the PCB/breadboard and the latter is ready to be connected at the panel.<br>
⚠️ **DO NOT CONNECT THEM YET** ⚠️

**Very important step**: Double check that the connections are ok (add some black tape to keep the loose connections in place if you opted for a breadboard). It is difficult to debug for HW problems once phisically attached to the panel or near to it as often the panels are not in comfy locations.

##  Install ESPHome on the ESP device
// TO BE CHECKED

This paragraph covers only the installation of the software that interacts with Home Assistant and ESPHome.

Connect the ESP device to the PC. Make sure you have the right drivers (CP2102 (square chip) or CH341 -> <a href="https://esphome.io/guides/physical_device_connection.html">ESPHome - physical device connection</a>) installed on your PC or ESPHome will not be able to recognize it.

Now you have two options:
1. Install a basic version of ESPHome on the ESP device (thorugh command line esphome or web.esphome.io) and then adopt it on ESPHome of your HA instance (in order to install right after the needed software over the air) **OR**
2. Install the final project on the device by using the ESPHome addon on HA (might be required to connect the ESP on the PC that is running HA)

Before proceeding with any of the two options, make sure to identify the right board and framework (opt) for your ESP device. Check the ESPHome website for more info.<br>
E.g. for the ESP8266 Wemos D1 R2 
```yaml
  platform: esp8266
  board: d1_mini
```

If you opted for the first option, install a basic version of ESPHome using https://web.esphome.io/ and follow the instructions there or follow a guide on <a href="https://esphome.io/">ESPHome website</a> to set a ESPHome project using command line. 

For the second option instead, follow the next steps to configure the final project before flashing. Then when the project is ready you will have to follow the guide on the ESPHome plugin when adding a new device.

##  Flash project into ESP device

This paragraph covers only the installation of the software that interacts with Home Assistant and ESPHome.

Connect to the HA instance and let's browse to the `config/esphome/` folder.<br>
Here we need to create the following folder structure:
```
config/
├─ esphome/
│  ├─ vistaEcpInterface or vista20SE/
│  │  ├─ .cpp files
│  │  ├─ .h files
│  ├─ project_config.yaml
│  ├─ secrets.yaml
```

Populate the `secrets.yaml` with the secrets you need to configure. To know what secretes, just open the `project_config` file and search for "!secret". Every secret needs to be referenced in that file.

If you already adopted the ESP on ESPHome on the HA instance, please use the secrets yaml on the web ui. Same reasoning when updating the `project_config.yaml` file: use the one on the UI.

Example:<br>
```yaml
wifi_ssid: "SSID"
wifi_password: "Wifi Password"
access_code: "access code - e.g. 1234" # used only for disarm service
ap_wifi_password: "the password for the access point of the esp when loses connection"
ota_password: "the OTA password for the ESP"
api_password: "API password for HA"
```

Edit the `project_config.yaml` and set all the variables to the values you need. <br>

The yaml attributes should be fairly self explanatory for customization. The yaml example also shows how to setup named zones.

In general, the things you want to change are:
- `keypadAddr` -> this must be set to an unused keypad address, if using 20P code. Otherwise you can use whatever number you want (I set it to `max zone number I have + 1`). *Little note: address = zone number*
- `rxPin`, `txPin`, `monitorPin` -> use the pinout scheme to map them correctly
- `lrrSupervisor` -> if you don't have any LRR device (e.g. (IP or GSM) interface and monitored by a central monitoring station) set it to `True`

If you use the zone expanders and/or LRR functions, you might need to clear CHECK messages for the LRR and expanded zones from the panel on boot or restart by entering your access code followed by 1 twice. eg 12341 12341 where 1234 is your access code.

- `includes` -> here you need to point to the directory where you put the .h and .cpp files. 
E.g. 

```yaml
  includes:
    - vistaEcpInterface/`.
```

- `text_sensor` -> here you need to add a new bullet point or remove one for each zone you have. E.g. You have 30 zones, then you must have 30 bullet points. For each bullet point you add or remove make sure to reference the zone id also in the `VistaECP->onZoneStatusChange` method in the `custom_component/lambda` section of the yaml file. A similar thing must be done for the relays you configure.

E.g. Adding zone 22
```yaml
...
  - platform: template
    id: z22
    name: "$systemName Custom Sensor Name"

...

case 22: id(z22).publish_state(open); break;
```

Then you can set other stuff like the `TTL`.<br>
To compensate for the limitations of the minimal zone data sent by the panel, a time to live (TTL) attribute for each faulted zone was used.  The panel only sends fault messages when a zone is faulted or alarmed and does not send data when the zone is restored, therefore the TTL timer is used to reset a zone after a preset duration once it stops receiving those fault/alarm messages for that zone.  You can tweak the TTL setting in the YAML.  The default timer is set to 30 seconds.  I've also added persistent storage and recovery for zone status in the event of a power failure or reboot of the ESP.  The system will use persistent storage to recover the last known status of the zone on restart.

Suggested to add a static ip to the wifi configuration (see ESPHome website for more info on the additional yaml tags) in order to be able to debug better wifi problems.
```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  manual_ip:
    static_ip: XXX.XXX.XXX.XXX
    gateway: XXX.XXX.XXX.XXX
    subnet: XXX.XXX.XXX.XXX
```

After the yaml file has been setup, there is one additional step to be done: language adaptation.<br>
If your panel is not english or it is using different prompts for the status reporting, change the definitions in the `vistaAlarm.h` file. This job can be done also after the first installation in order to first sniff, looking at the esphome logs, the prompts.

*Example of italian translation for the Vista25IT (IT version of Vista20SE):*
```c
    // start panel language definitions
    const char *const FAULT = "APERT";
    const char *const BYPAS = "ESCL.";
    const char *const ALARM = "ALARM";
    ...
    const char *const CHECK = "VERIF";
    ...
    const char *const ARMED = "INSERIM.";

...

    const char *const HITSTAR = "SPENTO Premi* e ";
    // const char *const HITSTAR_ALT = "Premere *"; // if there is more than one message for the hit star prompt
    // end panel language definitions

...
	if (strstr(vista.statusFlags.prompt, HITSTAR)) // || strstr(vista.statusFlags.prompt, HITSTAR_ALT)) // add condition if there is more than one message for the hit star prompt
          vista.write('*');
...
```

Now it's time to flash everything into the ESP device.<br>

## Connecting everything 

⚠️ **Before continuing read [Warning notice](#⚠️-warning-⚠️)** ⚠️

Let's connect the ESP device to the breadboard or PCB and finally let's connect the breadboard/PCB to the panel. Depending on the configuration you selected, you need to connect the breadboard/PCB to the same keypad terminals of your existing physical keyboard **OR**, if you prefer, on the keypad terminals of another partition of your panel (in case of multi-partition panel).

**Make sure to keep the ESP device further from any RF receiver as the components and the ESP itself could cause interference.**

Power up the ESP device and check on Home Assistant. In a minute or so it should be ready and accessible from HA > ESPHome.

Ping the device address if having problems and double check the `project_config.yaml` file.

**Optional steps**

Cover the ESP device with a junction box or similar just to make a cover. Remember to keep at least two holes for some air venting.

Final result:

![everything_connected](readme_material/everything_connected.png)

Note: Even the circuitry can be placed outside if you want. The wires from circuitry to the ESP are just for demo purposes, not related at all to the wiring diagrams. The power wire depends on the setup you chose, but it's recommended to have an external power supply.

# Miscellaneous
## Example in Home Assistant

![Image of HASS example](readme_material/vista-ha.png)

The returned statuses for Home Assistant are: armed_away, armed_home, armed_night, pending, disarmed,triggered and unavailable.  

Sample Home Assistant Template Alarm Control Panel configuration with simple services (defaults to partition 1):

```
alarm_control_panel:
  - platform: template
    panels:
      safe_alarm_panel:
        name: "Alarm Panel"
        value_template: "{{states('sensor.vistaalarm_system_status')}}"
        code_arm_required: false
        
        arm_away:
          - service: esphome.vistaalarm_alarm_arm_away
                  
        arm_home:
          - service: esphome.vistaalarm_alarm_arm_home
          
        arm_night:
          - service: esphome.vistaalarm_alarm_arm_night
            data_template:
              code: '{{code}}' #if you didnt set it in the yaml, then send the code here
          
        disarm:
          - service: esphome.vistaalarm_alarm_disarm
            data_template:
              code: '{{code}}'                    
```

### Sample sensor configuration for card using mqtt

```
sensor:
  #partition 1 topics. 
  - platform: mqtt
    state_topic: "vista/Get/DisplayLine1/1"
    name: "DisplayLine1"

  - platform: mqtt
    state_topic: "vista/Get/DisplayLine2/1"
    name: "DisplayLine2"

  - platform: mqtt
    state_topic: "vista/Get/Status/AWAY/1"
    name: "vistaaway"
    
  - platform: mqtt
    state_topic: "vista/Get/Status/STAY/1"
    name: "vistastay"
  
  - platform: mqtt
    state_topic: "vista/Get/Status/READY/1"
    name: "vistaready"

  - platform: mqtt
    state_topic: "vista/Get/Status/TROUBLE/1"
    name: "vistatrouble"

  - platform: mqtt
    state_topic: "vista/Get/Status/BYPASS/1"
    name: "vistabypass"
    
  - platform: mqtt
    state_topic: "vista/Get/Status/CHIME/1"
    name: "vistachime"    

```

##  HA Services

- Basic alarm services. These services default to partition 1:

	- "alarm_disarm", Parameter: "code" (access code)
	- "alarm_arm_home" 
	- "alarm_arm_night", Parameter: "code" (access code)
	- "alarm_arm_away"
	- "alarm_trigger_panic"
	- "alarm_trigger_fire"


- Intermediate command service. Use this service if you need more versatility such as setting alarm states on any partition:

	- "set_alarm_state",  Parameters: "partition","state","code"  where partition is the partition number from 1 to 8, state is one of "D" (disarm), "A" (arm_away), "S" (arm_home), "N" (arm_night), "P" (panic) or "F" (fire) and "code" is your panel access code (can be empty for arming, panic and fire cmds )

- Generic command service. Use this service for more complex control:

	- "alarm_keypress",  Parameter: "keys" where keys can be any sequence of keys accepted by your panel. For example to arm in night mode you set keys to be "xxxx33" where xxxx is your access code. 
    
	- "alarm_keypress_partition",  Parameters: "keys","partition" where keys can be any sequence of keys accepted by your panel and partition is  1 - 3. For example to arm in night mode  on partition 1 you set keys to be "xxxx33" and partition to be "1"  where xxxx is your access code.     
    
  - "set_zone_fault",Parameters: "zone","fault" where zone is a zone from 9 - 48 and fault is 0 or 1 (0=ok, 1=open)
      The zone number will depend on what your expander address is set to.

## OTA updates

In order to make OTA updates, it is recommended that the connection switch in the frontend be switched to OFF since the ECP library is using interrupts and could cause issues with the update process.

##  MQTT integration

If your preference is to use MQTT instead of ESPHOME, you can use the Arduino sketch from the MQTT-Example directory. It supports pretty much all functions of the ESPHOME implementation.  To use, edit the configuration items at the top of the file for your setup then simply put the ino and all *.h and *.cpp vista library files in the same sketch directory and compile.  Read the comments within the sketch for more details.   

The sketch supports ArduinoOTA (https://www.arduino.cc/reference/en/libraries/arduinoota/) that will enable you to update the code via wifi once the initial upload is done using the Arduino IDE. 

Also supported are encrypted TLS connections to an SSL enabled MQTT server such as Mosquito on port 8883.  Please note that due to the high memory useage of the WifiClientSecure implamentation, the use of an ESP32 is recommended over an ESP8266.  Simply uncomment  "#define useMQTTSSL" to use.

You can also use this sketch with any other home control application that supports MQTT such as openHAB, Homebridge(HomeKit) , etc.  Some configuration examples provided in directory MQTT-Example. They are versions modified for this application from the originals at https://github.com/taligentx/dscKeybusInterface/tree/master/examples.

##  Setting up the alarm panel keyboard card on HA

I've added a sample lovelace alarm-panel card copied from the repository at https://github.com/GalaxyGateway/HA-Cards. I've customized it to work with this ESP library's services.   I've also added two new text fields that will be used by the card to display the panel prompts the same way a real keypad does. To configure the card, just place the `alarm-keypad-card.js` file into the `/config/www` directory of your homeassistant installation and add a new resource in your lovelace configuration pointing to `/local/alarm-keypad-card.js`. <br>
Inside the `/config/www` directory put also the beep mp3 files.

You can then configure the card as shown below. Just substitute your service name to your application and choose one of the two chunks.

The first example is for using the esphome component in a multi partition environment.  The second uses MQTT.  The MQTT example can also support multiple partitions. The partition number will be appended to the mqtt response topic for non zone statuses. To send cmds to an individual partition, replace the cmd payload from `!xxxxx` to `\&\<p\>xxxx` where `\<p\>` is the partition number to send the cmd to and xxxx is the key sequence to send. Cmds that do not specify the partition will be sent to the defaultpartition as set in the sketch.

```
# EX1 - Partition 1 example - HA

type: custom:alarm-keypad-card
title: Vista_ESPHOME - partition 1
unique_id: vista1
disp_line1: sensor.vistaalarm_line1
disp_line2: sensor.vistaalarm_line2
scale: 1
service_type: esphome
service: vistaalarm_alarm_keypress_partition
status_A: AWAY
status_B: STAY
status_C: READY
status_D: BYPASS
status_E: TROUBLE
status_F: CHIME
status_G: ''
status_H: ''
sensor_A: binary_sensor.vistaalarm_away
sensor_B: binary_sensor.vistaalarm_stay
sensor_C: binary_sensor.vistaalarm_ready
sensor_D: binary_sensor.vistaalarm_bypass
sensor_E: binary_sensor.vistaalarm_trouble
sensor_F: binary_sensor.vistaalarm_chime
button_A: STAY
button_B: AWAY
button_C: DISARM
button_D: BYPASS
button_F: <
button_G: '>'
button_E: A
button_H: B
cmd_A:
  keys: '12343'
  partition: 1
cmd_B:
  keys: '12342'
  partition: 1
cmd_C:
  keys: '12341'
  partition: 1
cmd_D:
  keys: '12346#'
  partition: 1
cmd_E:
  keys: 'A'
  partition: 1
cmd_H:
  keys: 'B'
  partition: 1
cmd_F:
  keys: '<'
  partition: 1
cmd_G:
  keys: '>'
  partition: 1
key_0:
  keys: '0'
  partition: 1
key_1:
  keys: '1'
  partition: 1
key_2:
  keys: '2'
  partition: 1
key_3:
  keys: '3'
  partition: 1
key_4:
  keys: '4'
  partition: 1
key_5:
  keys: '5'
  partition: 1
key_6:
  keys: '6'
  partition: 1
key_7:
  keys: '7'
  partition: 1
key_8:
  keys: '8'
  partition: 1
key_9:
  keys: '9'
  partition: 1
key_star:
  keys: '*'
  partition: 1
key_pound:
  keys: '#'
  partition: 1
text_1: 'OFF'
text_2: AWAY
text_3: STAY
text_4: MAX
text_5: TEST
text_6: BYPASS
text_7: INSTANT
text_8: CODE
text_9: CHIME
text_star: READY
text_pound: ''
text_0: ''  
beep: sensor.vistaalarm_beeps
view_pad: true
view_display: true
view_status: true
view_status_2: true
view_bottom: false
```

``` 
# EX2 - Partition 1 example - MQTT

type: 'custom:alarm-keypad-card'
title: Vista_MQTT
unique_id: vista2
disp_line1: sensor.displayline1
disp_line2: sensor.displayline2
scale: 1
service_type: mqtt
service: publish
status_A: AWAY
status_B: STAY
status_C: READY
status_E: BYPASS
status_D: TROUBLE
status_F: ''
status_G: ''
status_H: ''
sensor_A: sensor.vistaaway
sensor_B: sensor.vistastay
sensor_C: sensor.vistaready
sensor_E: sensor.vistabypass
sensor_D: sensor.vistatrouble
button_A: STAY
button_B: AWAY
button_C: DISARM
button_D: BYPASS
cmd_A:
  topic: vista/Set/Cmd
  payload: '!12343'
cmd_B:
  topic: vista/Set/Cmd
  payload: '!12342'
cmd_C:
  topic: vista/Set/Cmd
  payload: '!12341'
cmd_D:
  topic: vista/Set/Cmd
  payload: '!12346#'
key_0:
  topic: vista/Set/Cmd
  payload: '!0'
key_1:
  topic: vista/Set/Cmd
  payload: '!1'
key_2:
  topic: vista/Set/Cmd
  payload: '!2'
key_3:
  topic: vista/Set/Cmd
  payload: '!3'
key_4:
  topic: vista/Set/Cmd
  payload: '!4'
key_5:
  topic: vista/Set/Cmd
  payload: '!5'
key_6:
  topic: vista/Set/Cmd
  payload: '!6'
key_7:
  topic: vista/Set/Cmd
  payload: '!7'
key_8:
  topic: vista/Set/Cmd
  payload: '!8'
key_9:
  topic: vista/Set/Cmd
  payload: '!9'
key_star:
  topic: vista/Set/Cmd
  payload: '!*'
key_pound:
  topic: vista/Set/Cmd
  payload: '!#'
text_1: 'OFF'
text_2: AWAY
text_3: STAY
text_4: MAX
text_5: TEST
text_6: BYPASS
text_7: INSTANT
text_8: CODE
text_9: CHIME
text_star: READY
text_pound: ''
text_0: ''  
view_pad: true
view_display: true
view_status: true
view_status_2: true
view_bottom: false
beep: sensor.vistabeeps
```

![card-config](readme_material/master_card_config.png)

# FAQ

1. How do I use the Trace Analyzer for deeper investigation?

Connect the pulseview channel D0 to the yellow line input on the D1 ESP input pin and the pulseview channel D1 to the green line monitor input D5 pin on ESP and then finally ground to the common ground on the ESP side. You don't need to monitor VCC or the ESP D2 output since that will also be seen on the green line monitor input at D5. Make sure you don't connect the channel inputs to any voltage higher than around 5 volts.

You should not be sharing ground between the panel and the esp if you used the optocoupler isolated schematic to build your circuit. if you did, it won't hurt anything, it's just not needed.

The setup file is really just the decoder setup so you that you can actually see the data being sent. It's attached below.
Just unzip it and load it into PulseView software by selecting "Restore session setup" from the menu. Ensure you connect your device first before running PulseView so that it picks it up on start up. (it will show up as a Saelea 8 channel). Once it starts your session, you will see 8 inputs. Once you restore the saved session setup, you will see 3 channels and 2 decoders. One for the panel (yellow line) and one for the keypad (green line). You'll need to run the capture twice to actually capture data. Seems like the first time after loading a session, it does not see the channels but the second run is fine.

<a href="https://github.com/Dilbert66/esphome-vistaECP/files/7150101/vistaecp2.zip">vistaecp2.zip</a>

Once you have about 1 minute or so of data you can then click the save button to save the session. It will create an *.sr file.
zip that and attach it to the issue.


2. What is LRR?

LRR = Long Range Radio. IP or GSM interface and monitored by a central monitoring station.

3. What is an address?

A zone number. Zone number = Address. //TO BE CHECKED

4. How do I setup the expanders?

// TO BE CHECKED

5. What is the access code?

Your 4 digit secret number to arm/disarm the system. e.g. 1234

6. What is quickArm?

It will the hashtag instead of your access code for the arm command. E.g. it will send #3 instead of 12343 to enable "stay" mode.

7. How do I open an issue?

See [How to](#report-issue) section.


## Supported Models

As of today:

|  **Model** |  **Source Code**  |
|:----------:|:-----------------:|
| Vista 20P  | vistaEcpInterface |
| Vista 20SE | vista20SE         |
| Vista 25IT | vista20SE         |
| Vista120 | vistaEcpInterface - // TO BE CHECKED |
| Vista128bpt/250bpt | vistaEcpInterface - // TO BE CHECKED |
| | |
| | |
| | |
| | |


# Report issue

First: search for your issue into the closed and open issues in the repo, if you don't find anything, open a new issue specifying:<br>
- Panel Model
- Used ESP model
- Used wiring diagram
- YAML config (the necessary parts)
- Issue
- Installed sensors (if relevant)
- What you've tried
- Logs

# References

You can checkout the links below for further reading and other implementation examples. Some portions of the code in the repositories below was used in creating the library.
* https://github.com/TANC-security/keypad-firmware
* https://github.com/cweemin/espAdemcoECP
* https://github.com/TomVickers/Arduino2keypad/

This project is licensed under the `Lesser General Public License` version `2.1`, or (at your option) any later version as per it's use of other libraries and code. Please see `COPYING.LESSER` for more information.

# Author & Contributors

- @Dilbert66 (Author)
- @appleguru (Contributor)
- @nitej (Contributor)
- @rcmurphy (Contributor)
- @jdev-homeauto (Contributor)
- @lorenzodeveloper (Contributor)

// TO BE CHECKED