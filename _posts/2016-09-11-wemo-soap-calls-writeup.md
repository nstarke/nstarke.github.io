---
layout: post
title:  "SOAP Calls for UPnP Services in WeMo Devices"
date:   2016-09-11 00:00:00 -0600
categories: soap upnp wemo security
author: Nicholas Starke
---

### Note: this write up doesn't contain any vulnerabilties or exploits!

I was recently taking a look at a few WeMo embedded devices.  WeMo Devices are IoT contraptions like light switches, space heaters, and coffee machines that are network enabled. I examined the "Holmes Smart Heater". Both had port 41953 open, which is a common port for UPnP services. I decided to dig a little deeper and figure out a way to interact with the SOAP services which UPnP relies on in order to hunt for bugs.  My goal was to retrieve sensitive information, such as the WiFi password, from the device.  

Using [Miranda](https://github.com/0x90/miranda-upnp)'s MSEARCH (which comes preinstalled on Kali Linux), I was able to discover the `setup.xml` file for the service I was examining.  This file will always be XML, but the actual file name can change.  Another way to discover this initial entry point is to examine the network traffic with WireShark.  The MSEARCH HTTP requests are easy to spot.

The `setup.xml` for the Holmes Smart Heater looks like this:

```xml
<root xmlns="urn:Belkin:device-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<device>
<deviceType>urn:Belkin:device:HeaterB:1</deviceType>
<friendlyName>HeaterB</friendlyName>
<manufacturer>Belkin International Inc.</manufacturer>
<manufacturerURL>http://www.belkin.com</manufacturerURL>
<modelDescription>Belkin Plugin Socket 1.0</modelDescription>
<modelName>HeaterB</modelName>
<modelNumber>1.0</modelNumber>
<modelURL>http://www.belkin.com/plugin/</modelURL>
<serialNumber>221421S0000AA2</serialNumber>
<UDN>uuid:HeaterB-1_0-221421S0000AA2</UDN>
<UPC>123456789</UPC>
<macAddress>94103E58FB38</macAddress>
<firmwareVersion>WeMo_WW_2.00.10630.PVT-OWRT-Smart</firmwareVersion>
<iconVersion>1|49153</iconVersion>
<binaryState>0</binaryState>
<DeviceID>65543</DeviceID>
<DeviceFWVersion>V1.00</DeviceFWVersion>
<brandName>Holmes</brandName>
<iconList>
<icon>
<mimetype>jpg</mimetype>
<width>100</width>
<height>100</height>
<depth>100</depth>
<url>icon.jpg</url>
</icon>
</iconList>
<serviceList>
<service>
<serviceType>urn:Belkin:service:WiFiSetup:1</serviceType>
<serviceId>urn:Belkin:serviceId:WiFiSetup1</serviceId>
<controlURL>/upnp/control/WiFiSetup1</controlURL>
<eventSubURL>/upnp/event/WiFiSetup1</eventSubURL>
<SCPDURL>/setupservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:timesync:1</serviceType>
<serviceId>urn:Belkin:serviceId:timesync1</serviceId>
<controlURL>/upnp/control/timesync1</controlURL>
<eventSubURL>/upnp/event/timesync1</eventSubURL>
<SCPDURL>/timesyncservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:basicevent:1</serviceType>
<serviceId>urn:Belkin:serviceId:basicevent1</serviceId>
<controlURL>/upnp/control/basicevent1</controlURL>
<eventSubURL>/upnp/event/basicevent1</eventSubURL>
<SCPDURL>/eventservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:deviceevent:1</serviceType>
<serviceId>urn:Belkin:serviceId:deviceevent1</serviceId>
<controlURL>/upnp/control/deviceevent1</controlURL>
<eventSubURL>/upnp/event/deviceevent1</eventSubURL>
<SCPDURL>/deviceservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:firmwareupdate:1</serviceType>
<serviceId>urn:Belkin:serviceId:firmwareupdate1</serviceId>
<controlURL>/upnp/control/firmwareupdate1</controlURL>
<eventSubURL>/upnp/event/firmwareupdate1</eventSubURL>
<SCPDURL>/firmwareupdate.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:rules:1</serviceType>
<serviceId>urn:Belkin:serviceId:rules1</serviceId>
<controlURL>/upnp/control/rules1</controlURL>
<eventSubURL>/upnp/event/rules1</eventSubURL>
<SCPDURL>/rulesservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:metainfo:1</serviceType>
<serviceId>urn:Belkin:serviceId:metainfo1</serviceId>
<controlURL>/upnp/control/metainfo1</controlURL>
<eventSubURL>/upnp/event/metainfo1</eventSubURL>
<SCPDURL>/metainfoservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:remoteaccess:1</serviceType>
<serviceId>urn:Belkin:serviceId:remoteaccess1</serviceId>
<controlURL>/upnp/control/remoteaccess1</controlURL>
<eventSubURL>/upnp/event/remoteaccess1</eventSubURL>
<SCPDURL>/remoteaccess.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:deviceinfo:1</serviceType>
<serviceId>urn:Belkin:serviceId:deviceinfo1</serviceId>
<controlURL>/upnp/control/deviceinfo1</controlURL>
<eventSubURL>/upnp/event/deviceinfo1</eventSubURL>
<SCPDURL>/deviceinfoservice.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:smartsetup:1</serviceType>
<serviceId>urn:Belkin:serviceId:smartsetup1</serviceId>
<controlURL>/upnp/control/smartsetup1</controlURL>
<eventSubURL>/upnp/event/smartsetup1</eventSubURL>
<SCPDURL>/smartsetup.xml</SCPDURL>
</service>
<service>
<serviceType>urn:Belkin:service:manufacture:1</serviceType>
<serviceId>urn:Belkin:serviceId:manufacture1</serviceId>
<controlURL>/upnp/control/manufacture1</controlURL>
<eventSubURL>/upnp/event/manufacture1</eventSubURL>
<SCPDURL>/manufacture.xml</SCPDURL>
</service>
</serviceList>
<presentationURL>/pluginpres.html</presentationURL>
```

There are 11 different UPnP services that the WeMo device offers:

* WiFiSetup
* Timesync
* Basicevent
* Deviceevent
* Firmwareupdate
* Rules
* Metainfo
* Remoteaccess
* Deviceinfo
* Smartsetup
* Manufacture

Each one of these services had a corresponding XML file documenting the UPnP "actions" available for that service. For example, the `WiFiSetup` service has `<SCPDURL>/setupservice.xml</SCPDURL>`.  The output of the "actions" XML  file for each service is listed in Appendix A.

Here is a list of actions available per service:

- WiFiSetup
  - GetApList
  - GetNetworkList
  - ConnectHomeNetwork
  - GetNetworkStatus
- Timesync
  - TimeSync
  - GetTime
  - GetDeviceTime
- Event
  - SetBinaryState
  - SetLogLevelOption
  - GetFriendlyName
  - ReSetup
  - SetHomeId
  - GetHomeId
  - SetDeviceId
  - GetDeviceId
  - GetMacAddr
  - GetSerialNo
  - GetPluginUDN
  - GetSmartDevInfo
  - ShareHWInfo
  - ChangeFriendlyName
  - SetSmartDevInfo
  - GetRuleOverrideStatus
  - GetDeviceIcon
  - GetIconURL
  - GetLogFileURL
  - ChangeDeviceIcon
  - GetBinaryState
  - SetMultiState
  - SetCrockpotState
  - GetCrockpotState
  - SetJardenStatus
  - GetJardenStatus
  - GetWatchdogFile
  - GetSignalStrength
  - SetServerEnvironment
  - GetServerEnvironment
  - ControlCloudUpload
- Device
  - GetAttributeList
  - GetAttributes
  - SetAttributes
  - SetBlobStorage
  - GetBlobStorage
- Firmwareupdate
  - UpdateFirmware
  - GetFirmwareVersion
- Rules
  - UpdateWeeklyCalendar
  - EditWeeklycalendar
  - GetRulesDBPath
  - SetRulesDBVersion
  - GetRulesDBVersion
  - SetRuleID
  - DeleteRuleID
  - SimulatedRuleData
  - SetTemplates
  - GetTemplates
  - SetRules
  - GetRules
- Metainfo
  - GetMetaInfo
  - GetExtMetaInfo
- Remoteaccess
  - RemoteAccess
- Deviceinfo
  - GetDeviceInformation
  - GetInformation
  - OpenInstaAP
  - CloseInstaAP
  - GetConfigureState
  - InstaConnectHomeNetwork
  - UpdateBridgeList
  - GetRouterInformation
  - InstaRemoteAccess
  - SetAutoFwUpdateVar
  - GetAutoFwUpdateVar
- Smartsetup
  - PairAndRegister
  - GetRegistrationData
  - GetRegistrationStatus
- Manufacture
  - GetManufactureData
  
Some of these functions require input parameters.  I only made limited attempts to supply input as I found that the entire UPnP service could easily crash with invalid input.  When this happened, I had to power down the heater, which takes 60 seconds a pop.

If you want to see input parameters, please refer to Appendix A.

I wrote a small shell script to aid in making SOAP calls against this API:

```bash
#!/bin/bash
SERVICETYPE=$1
ACTION=$2
CONTROLURL=$3
HOST=$4

DATA="<?xml version=\"1.0\"?><SOAP-ENV:Envelope xmlns:SOAP-ENV=\"http://schemas.xmlsoap.org/soap/envelope/\" SOAP-ENV:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\"><SOAP-ENV:Body><m:$ACTION xmlns:m=\"$SERVICETYPE\"></m:$ACTION></SOAP-ENV:Body></SOAP-ENV:Envelope>"
curl -H "Host: $HOST" -H "Content-type: text/xml" -H "SOAPACTION: \"$SERVICETYPE#$ACTION\"" "http://$HOST:49153$CONTROLURL" --data "$DATA"
exit 0
```

Please note that this script would need to be modified to accept input parameters.

This script can be executed as such:

`bash soap.sh "urn:Belkin:service:deviceinfo:1" "GetRouterInformation" "/upnp/control/deviceinfo1" "<HOST_IP_ADDRESS>"`

`$SERVICETYPE` comes from `<serviceType>urn:Belkin:service:deviceinfo:1</serviceType>` in /setup.xml
`$ACTION` comes from the action XML document at /deviceinfoservice.xml: `<action><name>GetRouterInformation</name></action>`
`$CONTROLURL` comes from `<controlURL>/upnp/control/deviceinfo1</controlURL>` in /setup.xml
`$HOST` is the hostname or IP address of the WeMo device.

The most interesting endpoints where the `deviceinfo` endpoints, particularly the `GetRouterInformation` action.  However, this endpoint always returned an error, which is good, because otherwise it could potentially leak out the SSID and PSK of the WiFi network that the WeMo device is connected to.  

Another interesting thing I noticed was in /rulesservice.xml: 

```xml
<stateVariable sendEvents="no">
<name>RulesDBPath</name>
<dataType>string</dataType>
<defaultValue>/rules.db</defaultValue>
</stateVariable>
```

It turns out there is a zip file of a sqlite database at /rules.db on port 49153.  Taking it apart revealed the times I had set for my heater to turn on and off.  Beyond that, nothing too interesting in that database.

Rules.db (unzipped):
```bash
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
sqlite> .tables
BLOCKEDRULES        GROUPDEVICES        RULEDEVICES         RULESNOTIFYMESSAGE
DEVICECOMBINATION   LOCATIONINFO        RULES               SENSORNOTIFICATION
sqlite> .dump
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE 'RULES'                      ('RuleID'PRIMARY KEY,                     'Name' TEXT NOT NULL,                     'Type' TEXT NOT NULL,                     'RuleOrder' INTEGER,                     'StartDate' TEXT,                     'EndDate' TEXT,                     'State' TEXT,                     'Sync' TEXT);
INSERT INTO "RULES" VALUES('1','Nightly','Time Interval',2,'12201982','07301982','1','NOSYNC');
CREATE TABLE 'RULEDEVICES'                     ('RuleDevicePK' INTEGER PRIMARY KEY AUTOINCREMENT,                     'RuleID' INTEGER,                     'DeviceID' TEXT,                     'GroupID' INTEGER,                     'DayID' INTEGER,                     'StartTime' INTEGER,                     'RuleDuration' INTEGER,                     'StartAction' REAL,                     'EndAction' REAL,                     'SensorDuration' INTEGER,                     'Type' INTEGER,                     'Value' INTEGER,                     'Level' INTEGER,                     'ZBCapabilityStart' TEXT,                     'ZBCapabilityEnd' TEXT,                     'OnModeOffset' INTEGER,                     'OffModeOffset' INTEGER,                     'CountdownTime' INTEGER,                     'EndTime' INTEGER);
INSERT INTO "RULEDEVICES" VALUES(1,1,'uuid:Insight-1_0-231544K120002B',0,1,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
INSERT INTO "RULEDEVICES" VALUES(2,1,'uuid:Insight-1_0-231544K120002B',0,2,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
INSERT INTO "RULEDEVICES" VALUES(3,1,'uuid:Insight-1_0-231544K120002B',0,3,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
INSERT INTO "RULEDEVICES" VALUES(4,1,'uuid:Insight-1_0-231544K120002B',0,4,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
INSERT INTO "RULEDEVICES" VALUES(5,1,'uuid:Insight-1_0-231544K120002B',0,5,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
INSERT INTO "RULEDEVICES" VALUES(6,1,'uuid:Insight-1_0-231544K120002B',0,6,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
INSERT INTO "RULEDEVICES" VALUES(7,1,'uuid:Insight-1_0-231544K120002B',0,7,61200,14400,1.0,0.0,0,-1,-1,-1,'','',0,0,0,75600);
CREATE TABLE 'DEVICECOMBINATION'                     ('DeviceCombinationPK' INTEGER PRIMARY KEY AUTOINCREMENT,                     'RuleID' INTEGER,                     'SensorID' TEXT,                     'SensorGroupID' INTEGER,                     'DeviceID' TEXT,                     'DeviceGroupID' INTEGER);
CREATE TABLE 'SENSORNOTIFICATION'                     ('SensorNotificationPK' INTEGER PRIMARY KEY AUTOINCREMENT,                     'RuleID' INTEGER,                     'NotifyRuleID' INTEGER,                     'NotificationMessage' TEXT,                     'NotificationDuration' TEXT);
CREATE TABLE 'GROUPDEVICES'                     ('GroupDevicePK' INTEGER PRIMARY KEY AUTOINCREMENT,                     'GroupID' INTEGER,                     'DeviceID' TEXT);
CREATE TABLE 'LOCATIONINFO'                     ('LocationPk' INTEGER PRIMARY KEY AUTOINCREMENT,                     'cityName' TEXT,                     'countryName' TEXT,                     'latitude' TEXT,                     'longitude' TEXT,                     'countryCode' TEXT,                     'region' TEXT);
CREATE TABLE 'BLOCKEDRULES'                     ('Primarykey' INTEGER PRIMARY KEY AUTOINCREMENT,                     'ruleId' TEXT);
CREATE TABLE 'RULESNOTIFYMESSAGE' ('RuleID' INTEGER,'NotifyRuleID' TEXT NOT NULL,'Message' TEXT NOT NULL,'Frequency' INTEGER);
DELETE FROM sqlite_sequence;
INSERT INTO "sqlite_sequence" VALUES('RULEDEVICES',7);
COMMIT;
```

However, it did inspire me to run [Wfuzz](https://github.com/xmendez/wfuzz) looking for any other `.db` files in the web root.  I found another database at /notification.db, but again, nothing too interesting inside that sqlite database.

Notifcation.db:
```bash
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
sqlite> .tables
notificationData
sqlite> .dump
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE notificationData (SERIALNO int,Attr1 int,Attr2 int,Attr3 int,Attr4 int,Attr5 int,dstURL TEXT,dstType int);
COMMIT;
```

While I was running [Wfuzz](https://github.com/xmendez/wfuzz), I decided to also run it against `/FUZZservice.xml`, and it came up with `/insightservice.xml` - which was not enabled on the heater.

## Conclusions for now:
This analysis was conducted in my spare time over the course of a few hours and is in no means an in depth analysis.  However, from a very cursory overview, it seems like WeMo devices have decent security.  A few hours on an embedded device usually leads me to a root shell.

## Appendix A: Action XML files

### WiFiSetup service - /setupservice.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>GetApList</name>
<argumentList>
<argument>
<retval/>
<name>ApList</name>
<relatedStateVariable>ApList</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetNetworkList</name>
<argumentList>
<argument>
<retval/>
<name>NetworkList</name>
<relatedStateVariable>NetworkList</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>ConnectHomeNetwork</name>
<argumentList>
<argument>
<retval/>
<name>ssid</name>
<relatedStateVariable>ssid</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>auth</name>
<relatedStateVariable>auth</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>password</name>
<relatedStateVariable>password</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>encrypt</name>
<relatedStateVariable>encrypt</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>channel</name>
<relatedStateVariable>channel</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetNetworkStatus</name>
<argumentList>
<argument>
<retval/>
<name>NetworkStatus</name>
<relatedStateVariable>NetworkStatus</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>CloseSetup</name>
<argumentList></argumentList>
</action>
<action>
<name>StopPair</name>
<argumentList></argumentList>
</action>
</actionList>
<serviceStateTable>
<!--
 connected, connecting, disconnected, time out error 
-->
<stateVariable sendEvents="yes">
<name>NetworkStatus</name>
<dataType>string</dataType>
<defaultValue>Disconnected</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>PairingStatus</name>
<dataType>string</dataType>
<defaultValue>Connecting</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ApList</name>
<dataType>string</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="yes">
<name>NetworkList</name>
<dataType>string</dataType>
<defaultValue/>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Timesync service - /timesyncservice.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>TimeSync</name>
<argumentList>
<argument>
<retval/>
<!--  UTC value, long  -->
<name>UTC</name>
<relatedStateVariable>UTC</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>TimeZone</name>
<relatedStateVariable>TimeZone</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>dst</name>
<relatedStateVariable>dst</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DstSupported</name>
<relatedStateVariable>DstSupported</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetTime</name>
</action>
<action>
<name>GetDeviceTime</name>
<argumentList>
<argument>
<retval/>
<name>DeviceTime</name>
<relatedStateVariable>DeviceTime</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="no">
<!--  UTC seconds -->
<name>UTC</name>
<dataType>long</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="no">
<name>TimeZone</name>
<dataType>int</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>DeviceTime</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="no">
<name>dst</name>
<dataType>Boolean</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Event Service - /eventservice.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>SetBinaryState</name>
<argumentList>
<argument>
<retval/>
<name>BinaryState</name>
<relatedStateVariable>BinaryState</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetLogLevelOption</name>
<argumentList>
<argument>
<retval/>
<name>Level</name>
<relatedStateVariable>Level</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Option</name>
<relatedStateVariable>Option</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetFriendlyName</name>
<argumentList>
<argument>
<retval/>
<name>FriendlyName</name>
<relatedStateVariable>FriendlyName</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>ReSetup</name>
<argumentList>
<argument>
<retval/>
<name>Reset</name>
<relatedStateVariable>Reset</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetHomeId</name>
<argumentList>
<argument>
<retval/>
<name>SetHomeId</name>
<relatedStateVariable>HomeId</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetHomeId</name>
</action>
<action>
<name>SetDeviceId</name>
<argumentList>
<argument>
<retval/>
<name>SetDeviceId</name>
<relatedStateVariable>DeviceId</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetDeviceId</name>
</action>
<action>
<name>GetMacAddr</name>
</action>
<action>
<name>GetSerialNo</name>
</action>
<action>
<name>GetPluginUDN</name>
</action>
<action>
<name>GetSmartDevInfo</name>
</action>
<action>
<name>ShareHWInfo</name>
<argumentList>
<argument>
<retval/>
<name>Mac</name>
<relatedStateVariable>Mac</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Serial</name>
<relatedStateVariable>Serial</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Udn</name>
<relatedStateVariable>Udn</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>RestoreState</name>
<relatedStateVariable>RestoreState</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>HomeId</name>
<relatedStateVariable>HomeId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>PluginKey</name>
<relatedStateVariable>PluginKey</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>ChangeFriendlyName</name>
<argumentList>
<argument>
<retval/>
<name>FriendlyName</name>
<relatedStateVariable>FriendlyName</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetSmartDevInfo</name>
<argumentList>
<argument>
<retval/>
<name>SmartDevURL</name>
<relatedStateVariable>SmartDevURL</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetRuleOverrideStatus</name>
<argumentList>
<argument>
<retval/>
<name>RuleOverrideStatus</name>
<relatedStateVariable>RuleOverrideStatus</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetDeviceIcon</name>
<argumentList>
<argument>
<retval/>
<name>DeviceIcon</name>
<relatedStateVariable>DeviceIcon</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetIconURL</name>
<argumentList>
<argument>
<retval/>
<name>URL</name>
<relatedStateVariable>URL</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetLogFileURL</name>
<argumentList>
<argument>
<retval/>
<name>LOGURL</name>
<relatedStateVariable>LOGURL</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>ChangeDeviceIcon</name>
<argumentList>
<argument>
<retval/>
<name>PictureSize</name>
<relatedStateVariable>PictureSize</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>PictureHeight</name>
<relatedStateVariable>PictureWidth</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>PictureColorDeep</name>
<relatedStateVariable>PictureColorDeep</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetBinaryState</name>
<argumentList>
<argument>
<retval/>
<name>BinaryState</name>
<relatedStateVariable>BinaryState</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetMultiState</name>
<argumentList>
<argument>
<retval/>
<name>state</name>
<relatedStateVariable>StateList</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>state</name>
<relatedStateVariable>StateList</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>state</name>
<relatedStateVariable>StateList</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetCrockpotState</name>
<argumentList>
<argument>
<retval/>
<name>mode</name>
<relatedStateVariable>mode</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>time</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetCrockpotState</name>
<argumentList>
<argument>
<retval/>
<name>mode</name>
<relatedStateVariable>mode</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>time</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>cookedTime</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>timeStamp</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetJardenStatus</name>
<argumentList>
<argument>
<retval/>
<name>mode</name>
<relatedStateVariable>mode</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>time</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetJardenStatus</name>
<argumentList>
<argument>
<retval/>
<name>mode</name>
<relatedStateVariable>mode</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>time</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>cookedTime</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>timeStamp</name>
<relatedStateVariable>timeVariable</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetWatchdogFile</name>
<argumentList>
<argument>
<retval/>
<name>WDFile</name>
<relatedStateVariable>WDFile</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetSignalStrength</name>
<argumentList>
<argument>
<retval/>
<name>SignalStrength</name>
<relatedStateVariable>SignalStrength</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetServerEnvironment</name>
<argumentList>
<argument>
<retval/>
<name>ServerEnvironment</name>
<relatedStateVariable>ServerEnvironment</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>TurnServerEnvironment</name>
<relatedStateVariable>TurnServerEnvironment</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>ServerEnvironmentType</name>
<relatedStateVariable>ServerEnvironmentType</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetServerEnvironment</name>
<argumentList>
<argument>
<retval/>
<name>ServerEnvironment</name>
<relatedStateVariable>ServerEnvironment</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>TurnServerEnvironment</name>
<relatedStateVariable>TurnServerEnvironment</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>ServerEnvironmentType</name>
<relatedStateVariable>ServerEnvironmentType</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>ControlCloudUpload</name>
<argumentList>
<argument>
<name>EnableUpload</name>
<direction>in</direction>
<relatedStateVariable>EnableUpload</relatedStateVariable>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="yes">
<name>BinaryState</name>
<dataType>Boolean</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>mode</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>time</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>cookedTime</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>timeStamp</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>level</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>option</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>Reset</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>FriendlyName</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>HomeId</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>DeviceId</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>SmartDevInfo</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>MacAddr</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>SerialNo</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>PluginUDN</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>DeviceIcon</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="No">
<name>StateList</name>
<dataType>list</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>URL</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>RuleOverrideStatus</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>WDFile</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>SignalStrength</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ServerEnvironment</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>TurnServerEnvironment</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ServerEnvironmentType</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>EnableUpload</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Device Event Service - /deviceservice.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>GetAttributeList</name>
<argumentList>
<argument>
<name>attributeList</name>
<relatedStateVariable>String</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetAttributes</name>
<argumentList>
<argument>
<retval/>
<name>attributeList</name>
<relatedStateVariable>String</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetAttributes</name>
<argumentList>
<argument>
<retval/>
<name>attributeList</name>
<relatedStateVariable>String</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetBlobStorage</name>
<argumentList>
<argument>
<retval/>
<name>attributeList</name>
<relatedStateVariable>String</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetBlobStorage</name>
<argumentList>
<argument>
<retval/>
<name>attributeList</name>
<relatedStateVariable>String</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="yes">
<name>attributeList</name>
<dataType>String</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Firmware Update Service - /firmwareupdate.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>UpdateFirmware</name>
<argumentList>
<argument>
<retval/>
<name>NewFirmwareVersion</name>
<relatedStateVariable>NewFirmwareVersion</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>ReleaseDate</name>
<relatedStateVariable>ReleaseDate</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>URL</name>
<relatedStateVariable>URL</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Signature</name>
<relatedStateVariable>Signature</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DownloadStartTime</name>
<relatedStateVariable>DownloadStartTime</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>WithUnsignedImage</name>
<relatedStateVariable>WithUnsignedImage</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetFirmwareVersion</name>
<retval/>
<argumentList>
<argument>
<name>FirmwareVersion</name>
<relatedStateVariable>FirmwareVersion</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="yes">
<name>FirmwareVersion</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>FirmwareUpdateStatus</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="no">
<name>WithUnsignedImage</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Rules Service - /rulesservice.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>UpdateWeeklyCalendar</name>
<argumentList>
<argument>
<retval/>
<!--
 entire day's timers list: 
NumberOfTimers|time,action,[deviceUDN;deviceUDN;...]|time,action,[deviceUDN;deviceUDN...]|time,action 
 
-->
<!--
time, seconds from 00:00 am within 24 hours. action: 0x00 (OFF/NO detection), 0x01: (ON/Detection)
-->
<!--
PLEASE note that, for simpler timer stored on socket, deviceUDN is NULL
-->
<name>Mon</name>
<relatedStateVariable>Mon</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Tues</name>
<relatedStateVariable>Tues</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Wed</name>
<relatedStateVariable>Wed</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Thurs</name>
<relatedStateVariable>Thurs</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Fri</name>
<relatedStateVariable>Fri</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Sat</name>
<relatedStateVariable>Sat</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Sun</name>
<relatedStateVariable>Sun</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>EditWeeklycalendar</name>
<argumentList>
<argument>
<retval/>
<!--  0x00: disbale; 0x01: enable; 0x02: remove -->
<!--
- Disable will disable entire week schedule, now only remove will be applied 
since app will manage all rules and store on device on other way 
-->
<name>action</name>
<relatedStateVariable>action</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<!-- Should create detailed low level protocol -->
<action>
<name>GetRulesDBPath</name>
<argumentList>
<argument>
<retval/>
<name>RulesDBPath</name>
<relatedStateVariable>RulesDBPath</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetRulesDBVersion</name>
<argumentList>
<argument>
<retval/>
<name>RulesDBVersion</name>
<relatedStateVariable>RulesDBVersion</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetRulesDBVersion</name>
<argumentList>
<argument>
<retval/>
<name>RulesDBVersion</name>
<relatedStateVariable>RulesDBVersion</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetRuleID</name>
<argumentList>
<argument>
<retval/>
<name>RuleID</name>
<relatedStateVariable>RuleID</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>RuleMsg</name>
<relatedStateVariable>RuleMsg</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>RuleFreq</name>
<relatedStateVariable>RuleFreq</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>DeleteRuleID</name>
<argumentList>
<argument>
<retval/>
<name>RuleID</name>
<relatedStateVariable>RuleID</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SimulatedRuleData</name>
<argumentList>
<argument>
<retval/>
<name>DeviceList</name>
<relatedStateVariable>DeviceList</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DeviceCount</name>
<relatedStateVariable>DeviceCount</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<!--  ** BEGIN ** Rules TNG API: XML Schema  -->
<action>
<name>SetTemplates</name>
<argumentList>
<argument>
<retval/>
<name>templateList</name>
<relatedStateVariable>templateList</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetTemplates</name>
<argumentList>
<argument>
<retval/>
<name>templateList</name>
<relatedStateVariable>templateList</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetRules</name>
<argumentList>
<argument>
<retval/>
<name>ruleList</name>
<relatedStateVariable>ruleList</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetRules</name>
<argumentList>
<argument>
<retval/>
<name>ruleList</name>
<relatedStateVariable>ruleList</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<!--  ** END ** Rules TNG API: XML Schema  -->
</actionList>
<serviceStateTable>
<!--
 connected, connecting, disconnected, time out error 
-->
<stateVariable sendEvents="no">
<name>RulesDBPath</name>
<dataType>string</dataType>
<defaultValue>/rules.db</defaultValue>
</stateVariable>
<stateVariable sendEvents="no">
<name>RulesDBVersion</name>
<dataType>string</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Mon</name>
<dataType>string</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Tues</name>
<dataType>string</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Wed</name>
<dataType>string</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Thurs</name>
<dataType>Thurs</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Fri</name>
<dataType>Fri</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Sat</name>
<dataType>Sat</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="no">
<name>Sun</name>
<dataType>Sun</dataType>
<defaultValue/>
</stateVariable>
<stateVariable sendEvents="yes">
<name>RuleID</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>RuleMsg</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>RuleFreq</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>DeviceList</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>DeviceCount</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<!--  ** BEGIN ** Rules TNG API  -->
<stateVariable sendEvents="yes">
<name>templateList</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ruleList</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<!--  ** END ** Rules TNG API  -->
</serviceStateTable>
</scpd>
```

### Meta Info Service - /metainfoservice.xml

```xml
This XML file does not appear to have any style information associated with it. The document tree is shown below.
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>GetMetaInfo</name>
<argumentList>
<argument>
<retval/>
<name>MetaInfo</name>
<relatedStateVariable>MetaInfo</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetExtMetaInfo</name>
<argumentList>
<argument>
<retval/>
<name>ExtMetaInfo</name>
<relatedStateVariable>ExtMetaInfo</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="no">
<name>ExtMetaInfo</name>
<dataType>String</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>MetaInfo</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Remote Access Service - /remoteaccess.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>RemoteAccess</name>
<argumentList>
<argument>
<retval/>
<name>DeviceId</name>
<relatedStateVariable>DeviceId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>dst</name>
<relatedStateVariable>dst</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>HomeId</name>
<relatedStateVariable>HomeId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DeviceName</name>
<relatedStateVariable>DeviceName</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>MacAddr</name>
<relatedStateVariable>MacAddr</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>pluginprivateKey</name>
<relatedStateVariable>pluginprivateKey</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>smartprivateKey</name>
<relatedStateVariable>smartprivateKey</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>smartUniqueId</name>
<relatedStateVariable>smartUniqueId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>numSmartDev</name>
<relatedStateVariable>numSmartDev</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>ArpMac</name>
<relatedStateVariable>ArpMac</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="yes">
<!--  DEV ID  -->
<name>homeId</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>"pluginprivateKey"</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>"smartprivateKey"</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>statusCode</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>resultCode</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>description</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>smartUniqueId</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>smartDeviceDescription</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>pluginprivateKey</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>smartprivateKey</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>numSmartDev</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ArpMac</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Device Info Service - /deviceinfoservice.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>GetDeviceInformation</name>
<argumentList>
<argument>
<retval/>
<name>UTC</name>
<relatedStateVariable>UTC</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>TimeZone</name>
<relatedStateVariable>TimeZone</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>dst</name>
<relatedStateVariable>dst</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DstSupported</name>
<relatedStateVariable>DstSupported</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DeviceInformation</name>
<relatedStateVariable>DeviceInformation</relatedStateVariable>
<direction>out</direction>
</argument>
<!--   Adding Countdown Time  -->
<argument>
<retval/>
<name>CountdownTime</name>
<relatedStateVariable>CountdownTime</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>TimeSync</name>
<relatedStateVariable>TimeSync</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetInformation</name>
<argumentList>
<argument>
<retval/>
<name>UTC</name>
<relatedStateVariable>UTC</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>TimeZone</name>
<relatedStateVariable>TimeZone</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>dst</name>
<relatedStateVariable>dst</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DstSupported</name>
<relatedStateVariable>DstSupported</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>Information</name>
<relatedStateVariable>Information</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>TimeSync</name>
<relatedStateVariable>TimeSync</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>OpenInstaAP</name>
<argumentList>
<argument></argument>
</argumentList>
</action>
<action>
<name>CloseInstaAP</name>
<argumentList>
<argument></argument>
</argumentList>
</action>
<action>
<name>GetConfigureState</name>
<argumentList>
<argument>
<retval/>
<name>ConfigureState</name>
<relatedStateVariable>ConfigureState</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>InstaConnectHomeNetwork</name>
<argumentList>
<argument>
<retval/>
<name>ssid</name>
<relatedStateVariable>ssid</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>auth</name>
<relatedStateVariable>auth</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>password</name>
<relatedStateVariable>password</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>encrypt</name>
<relatedStateVariable>encrypt</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>channel</name>
<relatedStateVariable>channel</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>brlist</name>
<relatedStateVariable>brlist</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>UpdateBridgeList</name>
<argumentList>
<argument>
<retval/>
<name>BridgeList</name>
<relatedStateVariable>BridgeList</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetRouterInformation</name>
<argumentList>
<argument>
<retval/>
<name>mac</name>
<relatedStateVariable>mac</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>ssid</name>
<relatedStateVariable>ssid</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>auth</name>
<relatedStateVariable>auth</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>password</name>
<relatedStateVariable>password</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>encrypt</name>
<relatedStateVariable>encrypt</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>channel</name>
<relatedStateVariable>channel</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>InstaRemoteAccess</name>
<argumentList>
<argument>
<retval/>
<name>DeviceId</name>
<relatedStateVariable>DeviceId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>dst</name>
<relatedStateVariable>dst</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>HomeId</name>
<relatedStateVariable>HomeId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>DeviceName</name>
<relatedStateVariable>DeviceName</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>MacAddr</name>
<relatedStateVariable>MacAddr</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>pluginprivateKey</name>
<relatedStateVariable>pluginprivateKey</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>smartprivateKey</name>
<relatedStateVariable>smartprivateKey</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>smartUniqueId</name>
<relatedStateVariable>smartUniqueId</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>numSmartDev</name>
<relatedStateVariable>numSmartDev</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>SetAutoFwUpdateVar</name>
<argumentList>
<argument>
<retval/>
<name>AutoFwUpdateVar</name>
<relatedStateVariable>AutoFwUpdateVar</relatedStateVariable>
<direction>in</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetAutoFwUpdateVar</name>
<argumentList>
<argument>
<retval/>
<name>AutoFwUpdateVar</name>
<relatedStateVariable>AutoFwUpdateVar</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="yes">
<name>DeviceInformation</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<!--   Adding Countdown Time  -->
<stateVariable sendEvents="yes">
<name>CountdownTime</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>Information</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ConfigureState</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>BridgeList</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>mac</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>ssid</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>auth</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>password</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>encrypt</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>channel</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>PairingStatus</name>
<dataType>string</dataType>
<defaultValue>Connecting</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>statusCode</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>TimeSync</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>AutoFwUpdateVar</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Smart Setup Service - /smartsetup.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>PairAndRegister</name>
<argumentList>
<argument>
<retval/>
<name>PairingData</name>
<relatedStateVariable>PairingData</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>RegistrationData</name>
<relatedStateVariable>RegistrationData</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>PairingStatus</name>
<relatedStateVariable>PairingStatus</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetRegistrationData</name>
<argumentList>
<argument>
<retval/>
<name>SmartDeviceData</name>
<relatedStateVariable>SmartDeviceData</relatedStateVariable>
<direction>in</direction>
</argument>
<argument>
<retval/>
<name>RegistrationData</name>
<relatedStateVariable>RegistrationData</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
<action>
<name>GetRegistrationStatus</name>
<argumentList>
<argument>
<retval/>
<name>RegistrationStatus</name>
<relatedStateVariable>RegistrationStatus</relatedStateVariable>
<direction>out</direction>
</argument>
<argument>
<retval/>
<name>StatusCode</name>
<relatedStateVariable>StatusCode</relatedStateVariable>
<direction>out</direction>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvents="yes">
<name>PairingStatus</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>RegistrationData</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>StatusCode</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
<stateVariable sendEvents="yes">
<name>RegistrationStatus</name>
<dataType>string</dataType>
<defaultValue>0</defaultValue>
</stateVariable>
</serviceStateTable>
</scpd>
```

### Manufacture Service - /manufacture.xml

```xml
<scpd xmlns="urn:Belkin:service-1-0">
<specVersion>
<major>1</major>
<minor>0</minor>
</specVersion>
<actionList>
<action>
<name>GetManufactureData</name>
<argumentList>
<argument>
<name>ManufactureData</name>
<direction>out</direction>
<relatedStateVariable>ManufactureData</relatedStateVariable>
</argument>
</argumentList>
</action>
</actionList>
<serviceStateTable>
<stateVariable sendEvent="no">
<name>ManufactureData</name>
<dataType>string</dataType>
</stateVariable>
</serviceStateTable>
</scpd>
```