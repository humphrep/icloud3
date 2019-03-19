# iCloud3  Device Tracker Custom Component  

[![Version](https://img.shields.io/badge/Version-1.0.0-blue.svg "Version")](https://github.com/gcobb321/icloud3)
[![Released](https://img.shields.io/badge/Released-3/15/2019-blue.svg "Released")](https://github.com/gcobb321/icloud3)
[![Project Stage](https://img.shields.io/badge/ProjectStage-General.Availability-red.svg "Project Stage")](https://github.com/gcobb321/icloud3)
[![Type](https://img.shields.io/badge/Type-Custom.Component-orange.svg "Type")](https://github.com/gcobb321/icloud3)
[![Licensed](https://img.shields.io/badge/Licesned-MIT-green.svg "License")](https://github.com/gcobb321/icloud3)

----------

## INTRODUCTION

'iCloud3' is an improved version of the iCloud device_tracker component installed with Home Assistant.  It's  purpose is: 

- To provide easy-to-use presence detection that does not rely on any other program, other than Home Assistant and the Home Assistant IOS app.
- To be report accurate information (location, distance from home, current zone) on a timely basis that can be used reliably in automations.
- To conserve the devices battery. 
- To correct GPS wandering errors leading to incorrect triggering of automations.
- To provide more distance, travel time, and zone attributes than the base iCloud component that can trigger and control automations and to easily display additional device information on lovelace cards. 

Below are some sample Lovelace screenshots showing how iCloud3 information can be displayed (see *ui-lovelace-icloud3.yaml* in the *configuration files* directory). Example configuration files for sensors, switches, badges, automations and scripts are also found in the *configuration files* directory that report location information and device status, along with running automations (opening a garage door) when arriving home. Other uses (security, lighting, heating & cooling control, etc.) can be added to the automations to meet your needs. 

*Special Note: I want to thank Walt Howd, (iCloud2 fame) who inspired me to tackle this project. I also want to give a shout out to Kovács Bálint, Budapest, Hungary who wrote the Python WazeRouteCalculator and some awesome HA guys (Petro31, scop, tsvi, troykellt, balloob, Myrddyn1, mountainsandcode,  diraimondo, fabaff, squirtbrnr, and mhhbob) who gave me the idea of using Waze in iCloud3...Gary Cobb aka GeeksterGary.* 

![Screenshots](screenshots/Readme-Screenshots.jpg)


### Installation

iCloud3 uses the GitHub Releases framework to download all the necessary installation files (iCloud3 custom component, documentation, sample configuration files, sample lovelace cards, etc). Go to the 'Releases' tab at the top of this repository, select the version of iCloud3 you want and download the .zip file. 

- HA 89+ -- Create a *config/custom_component/icloud3* directory on the device (Raspberry Pi) running Home Assistant. Copy the  ``` device_tracker.py ```' file into that directory.
- HA 88 and earlier --   Create a *config/custom_component/device_tracker* directory on the device (Raspberry Pi) running Home Assistant. Copy the  ``` device_tracker.py ```  file into that directory and rename it ``` to ``` ``` icloud3.py ``` .

*Note:* The WazeRouteCalculator component is use to calculate driving distance and time from your location to your Home zone. Normally, it is installed with the Home Assistant and Hass.io frameworks. However, if it is not installed on your system, you can go [here](https://github.com/kovacsbalu/WazeRouteCalculator) for instructions to download and instal Waze. If you don't want to use Waze or are in an area where Waze is not available, you can use the 'direct distance' method of calculating your distance and time from Home. Add the `distance_method: calc` parameter to your device_tracker: icloud3 configuration setup (see more information on this and other parameters later).



### What's different

iCloud3 has many features not in the base iCloud device_tracker that is part of Home Assistant. It exposes many new attributes, provides many new features, is based on enhanced route tracking methods, is much more accurate, and includes additional service calls. Lets look at the differences.

| Feature | Original iCloud | iCloud3 |
|---------|-----------------|---------|
| Integration with the HA IOS App            | No                                                           | Yes, Geographic Zone Enter/Exit, Background Fetch, Significant Location Update & Manual transactions are detected and processed immediately after they are issued. |
| Device Selection                           | None                                                         | Include and/or exclude device by type (iPhone, iPad, etc) or by device name (garyiphone, lillianiphone, etc). |
| Minimum Poll Interval | 1 minute | 15 second |
| Distance Accuracy | 1 km/mile | .01 km/mile |
| Variable Polling | Yes - Based on distance from home, battery level, GPS Accuracy | Yes, based on distance from home, Waze travel time to home, direction of travel, if the device is in a zone, battery level, GPS Accuracy, 'old location' status. |
| Detects zone changes | No - Also requires other device_trackers (OwnTracks, Nmap, ping, etc. | Yes, no other programs are needed. |
| Detects when leaving Home Zone | Delayed to next polling cycle (default of 30-minutes) | Detects when the Home Assistant IOS app issues a Zone Enter/Exit notification |
| Discards old transactions | No | Yes, if older than 2-minutes or with poor gps accuracy. |
| Fixes 'Not Home' issue when in Sleep Mode | No | Yes, by discarding all transactions that are closer than 1-km when the device is in a zone and by providing zone template sensors that are used to trigger automations rather than device_tracker state changes. |
| Integrates with Waze route/map tracker | No | Yes, the Waze travel time is used to determine the 'polling interval'. Waze can be disabled if not available using configuration parametres. |
| Device Polling Interval when close to home | 1+ minutes (?) | 15-seconds |
| Dynamic Stationary Zone | No | Yes, a Stationary Zone is created if no movement has been detected in 8-minutes (configurable). The polling interval is set to 30-minutes (default) until zone exit notification is received. |
| Service call commands | Set polling interval, Reset devices | Set polling interval, Reset devices, Pause/restart polling, Change zone, Enable/disable Waze Route information usage, information logging (some commands can be for all devices or for a specific device) |
| Device Filters | None | By device type or device name |
| | | |
| <u>Geekster Statistics:</u> | | |
| ● Config variables | 5 | 22 |
| ● Attributes | 20 | 35 |
| ● Service Calls | 4 | 4 + 12 special commands |
| ● Lines of code | 425 | 4000+ |

### How it works

iCloud3 operates on a 5-second cycle, looking for transactions from the HA IOS App. It looks for Geographic Zone Enter/Exit, Background Fetch, Significant Location Update and Manual transactions. When one is detected, it determines if the transaction is current before it is processed. Transactions older than 2-minutes are discarded. Additionally, to minimize GPS wandering and out-of-zone state changes, transactions within 1km of a zone are discarded when the device is in a zone.  Every 15-seconds, it determines if any devices needed to be polled using the iCloud Location Services service as described below.

iCloud3 polls the device on a dynamic schedule and determines the polling interval time based on:

 - If the device is in a zone or not in a zone. The zones include the ones you have set up in zone.yaml and a special Dynamic Stationary Zone that is created by iCloud3 when you haven't changed your location in a while (shopping center, friends house, doctors office, etc.)
 - A 'line-of-sight' calculated distance from 'home' to your current location using the GPS coordinates.
 - The driving time and distance between 'home' and your current location using the [Waze](http://www.waze.com) map/driving/direction service. 
 - The direction you are going — towards home, away from home or stationary.
 - The battery level of the device.
 - The accuracy of the GPS location and if the last poll returned a location that the iCloud service determined was 'old'.

The above analysis results in a polling interval. The further away from home and the longer the travel time back to home, the longer the interval; the closer to home, the shorter the interval. The polling interval checks each device being tracked every 15 seconds to see if it's location should be updated. If so, it and all of the other devices being tracked with iCloud3 are updated (more about this below). With a 15 second interval, you track the distance down 1/10 of a mile/kilometer. This gives a much more accurate distance number that can used to trigger automations. You no longer limited to entering or exiting a zone. 

Note: The `pyicloud.py` Python component is part of Home Assistant and used to poll the device, requesting location and other information. If the iCloud account is associated with multiple devices, all of the devices are polled, whether or not the device is being tracked by Home Assistant. This is a limitation of pyicloud.py. 




#### What other programs do I need

The `Home Assistant IOS App` is all. You do not need `OwnTracks` or other location based trackers and you do not need `nmap`, `netgear`, `ping` or any network monitor. The `Home Assistant IOS App` will notify Home Assistant when you leave home and iCloud3 device tracker will start keeping up with the device's location, the distance to home and the time it will take to get there.  

*Note:* The IOS App settings `Zone enter/exit`, `Background Fetch` and `Significant Location Change` location settings need to be enabled. 

The `iCloud` platform allows you to detect presence using the  [iCloud](https://www.icloud.com/) service. iCloud allows users to track their location on iOS devices. Your device needs to be registered with “Find My iPhone”.

The HA proximity component also determines distance between zones and the device, determines direction of travel, and other device_tracker related functions. Unfortunately, the IOS App reports old location information on a regular basis that is processed by the proximity component. 

​		[It is highly recommended to not use the proximity component when using iCloud3.]()

​		[iCloud3 duplicates the proximity functions and discards bad location information]()

​		[where the proximity component does not.]()



#### What happens if I don't have the IOS app on my device

The  `Home Assistant IOS App`  issues zone enter/exits and pushes location updates to HA; the iCloud Find-my-Friends issues location updates and other device information when asked by iCloud3. They both must map to the same device (see about assigning device_names a little later). If they do not, or if the IOS app is not installed on the device, the zone enter/exits will not be picked up when they actually happen.

They will be updated, however, on the next poll by iCloud3. The problem with not having the IOS app installed is if you are in a zone and on a 2-hour polling interval, it could be 2-hours before the device goes to a not_home state. With the IOS app, the zone exit is pushed it to HA where it gets picked up by iCloud3 and the device's state is changed. This happens within 10-seconds of getting the exit notification the zone. Naturally, if you are in a poor service area, this notification may be delayed.

```
# Example device names:
iPhone Settings -- General>About = Gary-iPhone
IOS App name    -- gary_iphone 
    
Note: Special characters (’-’) get mapped to ‘_’ by HA and must be accounted for. 
I could have set the iPhone name to Gary_iPhone but didn’t
```



### Home Assistant Configuration

To integrate iCloud3 in Home Assistant, add the following section to your `configuration.yaml` file:

```yaml
device_tracker:
  - platform: icloud3
    username: gary_Apple_iCloud_Account_ID 
    password: gary_Apple_iCloud_Account_Password
    account_name: gary_icloud
    include_device_type: iphone      <<<<< Note: See Configuration below for more info
```

If you have several devices that are associated with different Apple iCloud accounts, add a second iCloud3 platform with the other iCloud account. Since there may be a chance that a device is associated with several accounts, you should add *include_device* or *exclude_device* statements to the other account configuration.   

*Note:* It is recommended that you have only one account with the *include_device_type* statement.

```yaml
device_tracker:
  - platform: icloud3
    username: gary_Apple_iCloud_Account_ID 
    password: gary_Apple_iCloud_Account_Password
    account_name: gary_icloud
    include_device_type: iphone
    exclude_device: lillian_iphone

- platform: icloud3
    username: lillian_Apple_iCloud_Account_ID 
    password: lillian_Apple_iCloud_Account_Password
    account_name: lillian_icloud
    include_device: lillian_iphone
```



### About Your Apple iCloud Account

Apple has enabled '2 Step Authentication' for iCloud accounts. To permit Home Assistant, and iCloud3, to access your iCloud account,  you need to have an authentication code sent via a text message to a trusted device, which is then entered in Home Assistant. The duration of this authentication is determined by Apple, but is now at 2 months.  

When your account needs to be authorized, or reauthorized, you will be notified and the Notification symbol (the bell in the upper right of the Home Assistant screen) will be highlighted. Take the following steps to complete the process:  
  1. Press the Notification Bell in the upper right-hand corner of your Home Assistant screen.

  2. A window is displayed, listing the trusted devices associated with your account. It will list an number (0, 1, 2, etc.) next to the phone number that can receive the text message containing the 2 Step Authentication code number used to authenticate the computer running Home Assistant (your Raspberry Pi for example).

  3. Type the number.

  4. A text message is sent. Type the authentication code you receive in the next window that is displayed.

*Note:* The Python program used to access your iCloud account, ` pyicloud `, does not support 2 Factor Authentication, the improved version of 2 Steps Authentication.

**Associating the iPhone Device Name with Home Assistant using the Home Assistant IOS App**   
The Device Name field of the device in Settings App>General>About>Name field on the iPhone and iPad and in the Apple Watch App for the iWatch is stored in the iCloud account and used by Home Assistant to identify the device. HA v0.86+ converts any special characters found in the Device Name field to an underscore ( _ ) while HA v0.85 and earlier dropped the special characters altogether; e.g., 'Gary-iPhone' becomes 'gary_iphone' in known_devices.yaml, automations, sensors, scripts, etc. The value, 'gary_iphone', in the Device ID field in the Home Assistant IOS App>Settings ties everything together. 

*Note:* When you use iCloud account is accessed on a new device, you may receive an email from Apple stating that someone has logged into your account.  

 


## CONFIGURATION VARIABLES

### User, Account and Device Configuration Items

**username**  
*(Required)* The username (email address) for the iCloud account. 

**password**  
*(Required)* The password for the account.

**account_name**  
The friendly name for the account_name.  
*Note:* If this isn’t specified, the account_name part of the username (the part before the `@` in the email address) will be used.

**include_device_types**  (or  **include_device_type**)  
**exclude_device_types**  (or  **exclude_device_type**)  
Include or exclude device type(s) that should* be tracked.  
*Default: Include all device types associated with the account*

```yaml
include_device_type: iphone

--- or ---
include_device_types:
  - iphone
  - ipad
```

**include_devices**  (or  **include_device**)  
**exclude_devices**  (or  **exclude_device**)  
Include or exclude devices that should be tracked.  
*Default: Include all devices associated with the account*  

```yaml
include_device_type:
  - iphone
exclude_device:
  - gary_iphone
```

*Note:* It is recommended that to you specify the devices or the device types you want to track to avoid confusion or errors. All of the devices you are tracking are shown in the `devices_tracked ` attribute. 

[Special Note for iCloud2 Users: It is recommended that the *filter_type* configuration  entry be changed to *include_devices*.](https://github.com/gcobb321/icloud3)



### Zone, Interval and Sensor Configuration Items

**inzone_interval**  
The interval between location updates when the device is in a zone. This can be in seconds, minutes or hours, e.g., 30 secs, 1 hr, 45 min, or 30 (minutes are assumed if no time qualifier is specified).  
*Default: 2 hrs*

**stationary_inzone_interval**  
The interval between location updates when the device is in a Dynamic Stationary Zone. This can be minutes or hours, e.g., 1 hr, 45 min, or 30 (minutes are assumed if no time qualifier is specified).  
*Default: 30 min*

**stationary_still_time**  
The number minutes of with little movement minutes before the device will be put into its Dynamic Stationary Zone.   
*Valid values: Number. Default: 8*  

**sensor_name_prefix**  
Sensors contain the same values as the device_tracker attributes and are created and updated by iCloud3.  A sensor's value is a state value; it is easier to refer to, does not require HA to decode templates to extract the device_tracer's attribute and additional, non-attribute sensors can be made available. They will trigger automations, can be used in conditions and are easily displayed on lovelace cards.

Example sensors are  ``` sensor.gary_iphone_distance ``` or ``` sensor.gary_iphone_zone ```. The ``` gary_iphone ``` part of the name is the sensor prefix. It can be the device's name (``` gary_iphone ```), the iCloud device name (``` gary ```) or a custom name such as ``` garyc ```. See the Sensors section below for more information on the different types of sensors and different ways to set up the sensor prefix.
*Valid values: devicename, name, customnamevalue. Default: devicename*  

**unit_of_measurement**  
The unit of measure for distances in miles or kilometers.   
*Valid values: mi, km. Default: mi*

**gps_accuracy_threshold**  
iCloud location updates come with some gps_accuracy varying from 10 to 5000 meters. This setting defines the accuracy threshold in meters for a location updates. This allows more precise location monitoring and fewer false positive zone changes. If the gps_accuracy is above this threshold, a location update will be retried again in 2 minutes to see if the accuracy has improved. After 5 retries, the normal interval that is based on the distance from home, the waze travel time and the direction will be used.  
*Default: 100*

*Note:* The accuracy and retry count are displayed in the `info` attribute field (*GPS.Accuracy-263(2)*) and on the `poll_count`  attribute field (*2-GPS*). In this example, the accuracy has been poor for 2 polling cycles.  



### Waze Configuration Items

**distance_method**  
iCloud3 uses two methods of determining the distance between home and your current location — by calculating the straight line distance using geometry formulas (like the Proximity sensor) and by using the Waze Route Tracker to determine the distance based on the driving route.  If you do not have Waze in your area or are having trouble with Waze. change this parameter to *calc* to set the interval using the distance between your current location and home rather than the Waze travel time. 
*Valid values: waze, calc. Default: waze*  

**waze_min_distance, waze_max_distance**  
These values are also used to determine if the polling internal should be based on the Waze distance. If the calculated straght-line distance is between these values, the Waze distance will be requested from the Waze mapping service. Otherwise, the calculated distane is used to determine the polling interval. 
*Default: min=1, max=1000*  

*Note:* The Waze distance becomes less accurate when you are close to home. The calculation method is better when the distances less than 1 mile or 1 kilometer.  
*Note:* If you are a long way from home, it probably doesn't make sense to use the Waze distance. You probably don't have any automations that would be triggered from that far away. 

**waze_realtime**  
Waze reports the travel time estimate two ways — by taking the current, real time traffic conditions into consideration (True) or as an average travel time for the time of day (False).  
*Valid values: True, False. Default: False*  

**waze_region**  
The area used by Waze to determine the distance and travel time.  
*Valid values: US (United States), NA (North America), EU (Europe), IL (Isreal). Default: US*  

**travel_time_factor**  
When using Waze and the distance from your current location to home is more than 3 kilometers/miles, the polling interval is calculated by multiplying the driving time to home by the `travel_time_factor`.  
*Default: .60*  

*Note:* Using the default value, the next update will be 3/4 of the time it takes to drive home from your current location. The one after that will be 3/4 of the time from that point. The result is a smaller interval as you get closer to home and a larger one as you get further away.  



## SPECIAL ZONES  

There are two zones that are special to the iCloud3 device tracker - the Dynamic Stationary Zone and the NearZone zone.

### Dynamic Stationary Zone  

A Dynamic Stationary Zone is a zone that iCloud3 creates when the device has not moved much over a period of time. Examples might be when you are at a mall, doctor's office, restaurant, friend's house, etc. If the device is stationary, it's Stationary Zone location (latitude and longitude) is automatically updated with the gps location, the device state is changed to Stationary and the interval time is set to the *stationary_inzone_interval* value (default is 30 mins). This almost eliminates the number of times the device must be polled to see how far it is from home when you haven't moved for a while. When you leave the Stationary Zone, the IOS App notifies Home Assistant that the Stationary Zone has been exited and the device tracking begins again.

*Note:* You do not have to create the Stationary Zone in the zones.yaml file, the iCloud3 device tracker automatically creates one for every device being tracked when Home Assistant is started. The initial location is latitude 90°, longitude 180° (the North Pole). It's name is *devicename_Stationary*.  

Details about the Stationary Zone:

- You must be at least 2.5 times the Home zone radius.
- It's radius is 2 times the Home zone radius.
- The maximum distance you can move in a specific amount of time is 1.5 times the Home zone radius.
- The amount of time you must be still is specified in the ``` stationary_still_time ``` configuration parameter (default is 8 minutes).

### near_zone Zone  

There may be times when the Home Zone's (or another zone's) cell service is poor and does not track the device adequately when the device nears a zone. This can create problems triggering automations when the device enters the zone since the Find-My-Friends location service has problems monitoring it's location.  

To solve this, a special 'near_zone' zone can be created that is a short distance from the real zone that will wake the device up. The IOS App stores the zone's location on the device and will trigger a zone enter/exit notification which will then change the device's device_tracker state to the 'near_zone' zone and change the polling interval to every 15-secs. It is not perfect and might not work every time but it is better than automations never being triggered when they should.

*Note:* You can have more than one 'near_zone' zone in the zones.yaml file. Set them up with a unique name that starts with 'near_zone';, e.g., near_zone_home, near_zone_quail, near_zone_work, etc. The *friendly_name* attribute should be NearZone for each one.



## ATTRIBUTES

There are numerous attributes that are available for use in automations or to monitor the location of your device. They are shown in following table.  

### Location and Polling Attributes

**interval**  
The current interval between update requests to your iCloud account for the location and other information. They increase as you get further away and decrease as you get closer to home.  

**travel_time**  
The Waze travel time to return home from your current location.  

**distance**  
The distance from home being used by the interval calculator. This will be either the Waze distance or the calculated distance.  

**waze_distance**  
The driving distance from home returned by Waze based on the shortest route.  

**calc_distance**  
The 'straight line' distance that is calculated using the latitude and longitude of home and your current location using geometric formulas.  

**zone, last_zone**  
The device's current and last zone. This is not to be confused with the device's state. The state can be changed by other programs (IOS app or automations issuing device_tracker.see service calls) where the zone attribute is only updated by iCloud3. Using the Zone attribute to trigger an automation eliminates the gps wandering problems (or greatly reduces them). 

**zone_timestamp**  
When the device's zone was last changed.

**dir_of_travel**  
The direction you are traveling — towards home, away from home, near home, or stationary. This is determined by calculating the difference between the distance from home on this location update and the last one. Stationary can be a little difficult to determine at times and sometimes needs several updates to get right.  

**last_update**  
The time of the last iCloud location update.  

**next_update**  
The time of the next iCloud location update.  

**last_located**  
The last time your iCloud account successfully located the device. Normally, this will be a few seconds after the update time, however, if you are in a dead zone or the GPS accuracy exceeds the threshold, the time will be older. In this case, a description of the issues is displayed in the `info` attribute field.  

**poll_count**  
The number of iCloud, IOS App and discarded transaction counts for the day. It's format is '##:##:##', e.g., '10:14:21' with the iCloud count (10) first, the IOS App (14) second and discarded transactions (21) third.

**info**  
A message area displaying information about the device. This includes the battery level, Waze status, GPS accuracy issues, how long the device has been stationary, etc.  

**trigger**  
The action or notification that caused the last update (Geographic Zone Entered or Exited, Background Fetch, Manual, iCloud, etc.).

**timestamp**  
When the last update was completed.



### Device Status Information Attributes

**battery**  
The battery level of the device..  

**battery_status**  
Charging or NotCharging.  

**latitude, longitude, altitude**  
The location of the device.  

**source_type**  
How the the `Home Assistant IOS App` located the device. This includes gps, beacon, router.  

**device_status**  
The status of the device — online if the device is located or offline if polling has been paused or it can not be located.  

**low_power_mode**  
If the device is running in low power mode.

**speed, altitude, course, floor, vertical_accuracy**  
Device information provided by the iCloud account.  



### Other Attributes

**authenticated**  
When the device's iCloud account was last authenticated.  

**tracked_devices**  
The devices that are being tracked based on the 'includes' and 'excludes' specified in the configuration.yaml file.  This will be the same for all devices tracked.  

**icloud3_version**  
The version of iCloud3 you are running.  



## SENSORS CREATED FROM DEVICE ATTRIBUTES

### How they are used in Automations and on Lovelace Cards

Normally in HA, a template sensor is used to convert one entity's attributes into a state that can be used in automations or displayed on lovelace cards. The ``` device_tracker.gary_iphone.attributes.distance ```  value is converted to ``` sensor.gary_iphone_distance ``` entity using a template sensor. To do this, HA to monitors the attribute's value to see if it has been changed, and if it has, convert the template to the new value and then update the sensor entity while it is doing all the other stuff it does.  

iCloud3 creates and updates sensor entities without the need for template sensors. This makes the device_tracker's attribute values easier to reference in automations and scripts, and immediately available without waiting on HA to do the conversion.

Below is a sample automation using the sensors created and updated by iCloud3 .

```yaml
automation:
  - alias: Gary Arrives Home
    id: gary_arrives_home
    trigger:
    - platform: state
      entity_id: sensor.gary_iphone_zone
      to: 'home'
    - platform: template
      value_template: '{{states.sensor.gary_iphone_distance.state | float <= 0.2}}'
```



The following sensors are updated using the device_tracker's attributes values:

| Tracking Sensors | Special Sensors | Device Sensors  |
| ---------------- | --------------- | --------------- |
| _interval        | _zone           | _battery        |
| _travel_time     | _last_zone      | _battery_status |
| _distance        | _zone_timestamp | _gps_accuracy   |
| _waze_distance   | _zone_name1     | _trigger        |
| _calc_distance   | _zone_name2     | _speed          |
| _last_update     | _zone_name3     |                 |
| _next_update     |                 |                 |
| _last_located    | _badge          |                 |
| _poll_count      |                 |                 |
| _info            |                 |                 |



### Special Sensors

**_zone_name1, _zone_name2, _zone_name3 Sensors** 

| *_zone value*          | *_zone_name1* | *_zone_name2*          | *_zone_name3*        |
| ---------------------- | ------------- | :--------------------- | -------------------- |
| home                   | Home          | Home                   | Home                 |
| not_home               | Away          | Not Home               | NotHome              |
| whse                   | Whse          | Whse                   | Whse                 |
| gary_iphone_stationary | Stationary    | Gary Iphone Stationary | GaryIphoneStationary |

**_badge Sensor**  
The 'badge' sensor is used to display either the zone or distance from home for a device. Set up a template sensor with the appropriate name and the state value will be updated when the other sensors are updated. Below is an example of the template sensor. The *sensor_name_prefix* is used to name the badge sensor.

```yaml
- platform: template
  sensors:
    gary_iphone_badge:  
      friendly_name: Gary
      value_template: ''
      entity_picture_template: /local/gary.png
      
    lillian_iphone_badge:  
      friendly_name: Lillian
      value_template: ''
      entity_picture_template: /local/lillian.png
```

![badge](screenshots\badge.jpg)



### Use *sensor.devicename_zone* instead of *device_tracker.devicename state* for zone change

There are times when gps wanders and you receive a zone exit state change when the device has not moved in the middle of the night. The sequence of events that takes place under the covers is (1) a zone change notification is sent by the IOS App based on bad gps information, (2) the device's state and location is changed, (3) triggering an automation that runs when you exit the Home zone. (4) iCloud3 sees the new state and location and processes the data and (5) sees the notification data is old and it was caused an incorrect state change and (6) puts the device back into the Home zone where it belongs. The net effect is HA triggers the automation before iCloud3 gets control so the correction takes place after the automation has already run.

The solution to eliminating this problem is to not trigger automations based on device state changes but to trigger them on zone changes. A *zone* and *last_zone* template sensor, updated by iCloud3, is used to do this. These template sensors are only updated by iCloud3 so they are not effected by incorrect device state changes.  See the example *gary_leaves_zone* automation in the ``` sn_home_away_gary.yaml ``` sample file where the ``` sensor.gary_iphone_zone ``` is used as a trigger. 



### Naming the Sensor

The sensors can be named several ways using new the *sensor_name_prefix* configuration parameter. They can be based on the devicename (default), the iCloud's name, or a custom name for each device. If using a custom name, devices not specified use the default devicename for the prefix.



Examples of the sensor_name_prefix:

```yaml
sensor_name_prefix: devicename       - gary_iphone_distance

sensor_name_prefix: name             - gary_distance

sensor_name_prefix:
  - gary_iphone @ garyc              - garyc_distance
  - lillian_iphone @ lillianc        - lillianc_distance
 
sensor_name_prefix:
  - gary_iphone @ garyc              - garyc_distance
                                     - lillian_iphone_distance
```

> The  ``` sn_device_tracker_attributes.yaml ``` file containing the attribute template sensors distributed in the configuration section of the iCloud3 repository must be deleted (or commented out so it will not load). The new version of the  ``` sn_badges ``` sensor template file must be used.



## DEVICE TRACKER SERVICES 

Four services are available for the iCloud3 device tracker component that are used in automations. 

| Service | Description |
|---------|-------------|
| icloud_update | Send commands to iCloud3 that change the way it is running (pause, resume, Waze commands, etc.) |
| icloud_set_interval | Override the dynamic interval calculated by iCloud3. |
| icloud_lost_phone | Play the Lost Phone sound. |
| icloud_reset | Reset the iCloud3 custom component. |

Description of each service follows.

### *icloud_update* Service  --  Control How iCloud3 Operates
This service allows you to change the way iCloud3 operates. The following parameters are used:

| Parameter | Description |
|-----------|-------------|
| account_name | account_name of the iCloud3 custom component specified in the Configuration Variables section described at the beginning of this document. *(Required)*|
| device_name | Name of the device to be updated. All devices will be updated if this parameter is not specified. *(Optional)* |
| command | The action to be performed (see below). *(Required)* |
| parameter | Additional parameters for the command. |

The following describe the commands that are available. 

| Command |  Parameter | Description |
|---------|------------|-------------|
| pause |  | Stop updating/locating a device (or all devices). Note: You may want to pause location updates for a device if you are a long way from home or out of the country and it doesn't make sense to continue locating your device. |
| resume |  | Start updating/locating a device (or all devices) after it has been paused. |
| resume |  | Reset the update interval if it was overridden the 'icloud_set_interval' service. |
| pause-resume |  | Toggle pause and resume commands |
| zone | zonename | Change iCloud3 state to 'zonename' (like the device_tracker.see service call) and immediately update the device interval and location data. Note: Using the device_tracker.see service call instead will update the device state but the new interval and location data will be delayed until the next 15-second polling iteration (rather than immediately). |
| waze | on | Turn on Waze. Use the 'waze' method to determine the update interval. |
| waze | off | Turn off Waze. Use the 'calc' method to determine the update interval. |
| waze | toggle | Toggle waze on or off |
|  waze | reset_range | Reset the Waze range to the default distances (min=1, max=99999). |
| info | interval | Show how the interval is determined by iCloud3. This is displayed real time in the `info` attribute field. |
| info | logging | Toggle writing detailed debug information records to the HA log file. |


```yaml
#Commands to Control Device Polling

icloud_command_pause_resume_polling:
  alias: 'Toggle Pause/Resume Polling'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: pause-resume
 
icloud_command_resume_polling:
  alias: 'Resume Polling'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: resume
        
icloud_command_pause_polling:
  alias: 'Pause Polling'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: pause

icloud_command_pause_polling_gary:
  alias: 'Pause (Gary)'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        device_name: gary_iphone
        command: pause

icloud_command_toggle_waze:
  alias: 'Toggle Waze On/Off'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: waze toggle

icloud_command_garyiphone_zone_home:
  alias: 'Gary - Zone Home'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        device_name: gary_iphone
        command: zone home
        
icloud_command_garyiphone_zone_not_home:
  alias: 'Gary - Zone not_home'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        device_name: gary_iphone
        command: zone not_home
```

```yaml
#Commands to Generate Detailed Information on iCloud3's Operations

icloud_command_info_interval_formula:
  alias: 'Display Interval Formula'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: info interval

icloud_command_info_logging_toggle:
  alias: 'Write Details to Log File (Toggle)'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: info logging
```



###  *icloud_set_interval*  Service  --  Override the interval

This service allows you to override the interval between location updates to a fixed time. It is reset when a zone is entered or when the icloud_update service call is processed with the 'resume'command. The following parameters are used:

| Parameter | Description |
|-----------|-------------|
| account_name | account_name of the iCloud3 custom component specified in the Configuration Variables section described at the beginning of this document. *(Required)* |
| device_name | Name of the device to be updated. All devices will be updated if this parameter is not specified. *(Optional)* |
| interval | The interval between location updates. This can be in seconds, minutes or hours. Examples are 30 sec, 45 min, 1 hr,  hrs, 30 (minutes are assumed if no time descriptor is specified). *(Required)* |

```yaml
#Commands to Change Intervals

icloud_set_interval_15_sec_gary:
  alias: 'Set Gary to 15 sec'
  sequence:
    - service: device_tracker.icloud_set_interval
      data:
        account_name: gary_icloud
        device_name: gary_iphone
        interval: '15 sec'
 
icloud_set_interval_1_min_gary:
  alias: 'Set Gary to 1 min'
  sequence:
    - service: device_tracker.icloud_set_interval
      data:
        account_name: gary_icloud
        device_name: gary_iphone
        interval: 1

icloud_set_interval_5_hrs_all:
  alias: 'Set interval to 5 hrs (all devices)'
  sequence:
    - service: device_tracker.icloud_set_interval
      data:
        account_name: gary_icloud
        interval: '5 hrs'
 
```

### *icloud_lost_iphone*  Service --  Play a tune to find your phone
This service will play the Lost iPhone sound on a specific device. 

| Parameter | Description |
|-----------|-------------|
| account_name | account_name of the iCloud3 custom component specified in the Configuration Variables section described at the beginning of this document. *(Required)* |
| device_name | Name of the device *(Required)* |

### *icloud_restart* Service -- Restart the iCloud3 Component
This service will refresh all of the devices being handled by iCloud3 and can be used when you have added a new device to your Apple account. You will have to restart Home Assist if you have made changes to the platform parameters (new device type, new device name, etc.) 

| Parameter | Description |
|-----------|-------------|
| account_name | account_name of the iCloud3 custom component specified in the Configuration Variables section described at the beginning of this document. *(Required)* |



## TECHNICAL INFORMATION

### How the Interval is Determined

The iCloud3 device tracked uses data from several sources to determine the time interval between the iCloud Find my Friends location update requests.  The purpose is to provide accurate location data without exceeding Apple's limit on the number of requests in a time period and to limit the drain on the device's battery.

The algorithm uses a sequence of tests to determine the interval. If the test is true, it's interval value is used and no further tests are done. The following is for the nerd who wants to know how this is done. 


| Test | Interval | Method Name|
|------|----------|------------|
| --- *The Zone (State) changed* --- |  | |
| Zone changed to Stationary Zone | stationary_inzone_interval | 1sz-Stationary |
| Zone Changed to other zone | inzone_interval | 1ez-Zone |
| In near_zone close to home | 15 seconds | 1nz-InHomeNearZone |
| In near_zone far from Home | 15 seconds | 1nhz-InOtherNearZone |
| Left Home zone | 4 minutes | 1ehz-ExitHomeZone |
| Left other zone | 2 minutes | 1ez-ExitOtherZone |
| Entered Another Zone | 4 minutes | 1cz-ZoneChanged |
| --- *The Zone (State) did not change* -- |  |  |
| Poor GPS Accuracy | 1 minute | 2-PoorGPS |
| Override interval specified | inzone_interval | 3-Override |
| In Stationary zone | stationary_inzone_interval | 4sz-Stationary |
| In Home zone or near Home zone and direction is Towards | inzone_interval | 4iz-InZone |
| In near_zone | 15 seconds | 4nz-NearZone |
| In other zone & inzone_interval > waze time | inzone_interval | 4iz-InZone |
| Just left a zone | 2.5 minutes | 5-LeftZone |
| Distance < 2.5km/1.5mi | 15 seconds | 10a-Dist < 2.5km |
| Distance < 3.5km/2mi | 30 seconds | 10b-Dist < 3.5km |
| Waze used and Waze time < 5 min. | time*`travel_time_factor` | 10c-WazeTime |
| Distance < 5km/3mi | 1 minute | 10d-Dist < 5km |
| Distance < 8km/5mi | 3 minutes | 10e-Dist < 8km |
| Distance < 12km/7.5mi | 15 minutes | 10f-Dist < 12km |
| Distance < 20km/12mi | 10 minutes | 10g-Dist < 20km |
| Distance < 40km/25mi | 15 minutes | 10h-Dist < 40km |
| Distance < 150km/90mi | 1 hour | 10i-Dist < 150km |
| Distance > 150km/90mi | distance/1.5 | 20-Calculated |

*Notes:* The interval is then multiplied by a value based on other conditions. The conditions are:
1. If Stationary, keep track of the number of polls when you are stationary (the stationary count reported in the `info` attribute). Multiply the interval time by 2 when the stationary count is an even number and by 3 when it is divisible by 3.

2. If the direction of travel is Away, multiply the interval time by 2.

3. Is the battery is low, the GPS accuracy is poor or the location data is old, don't make any of the above adjustments to the interval.

   

### Displaying Interval Calculation Information in the *Info* Field

iCloud3 can display information on how the *Interval* time was calculated when the device is polled. As mentioned, this is dependent on the zone, direction of travel, Waze travel time (if Waze is used), the distance between your current location and home and the accuracy of the gps information provided by the iCloud service and the IOS App. Below are samples what is displayed in the *Info* field.

 In this case, the device is not_home and just left the Stationary Zone.

```reStructuredText
●Interval=3 min (0-iosAppTrigger, Zone=not_home, Last=stationary, This=not_home), ●DirOfTrav=away_from (Dist=1.88), ●State=stationary->not_home, Zone=not_home ●Battery-85%
```

In this case, Gary just arrived home and is now in the Home Zone.

```reStructuredText
●Interval=2 hrs (1ez-EnterZone, Zone=home, Last=not_home, This=home), ●DirOfTrav=in_zone (Zone=home), ●State=not_home->home, Zone=home ●Battery-84%'
```

The following script will toggle writing the debug information:

```yaml
icloud_command_info_interval_formula:
  alias: 'Display Interval Formula'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: info interval
```

### Writing Debug Information to the HA Log File

iCloud3 can toggle writing debug information to the HA Log file on and off. Below is a sample of the information that is written. In this case, Gary was arriving home and updating gary_iphone data was triggered by a Zone/State Change (not_home to home) in the automation ``` au_home_away_gary.yaml ```

You have to have the *Logger: info* entry in the *configuration.yaml* file but you do not have to have any other 'debug' parameters.

This entry was triggered by a Zone/State change from 'not_home' to 'home'.

![](screenshots\SampleLog.jpg)



The following script will toggle writing the debug information:

```yaml
icloud_command_info_logging_toggle:
  alias: 'Write Details to Log File (Toggle)'
  sequence:
    - service: device_tracker.icloud_update
      data:
        account_name: gary_icloud
        command: info logging
```

