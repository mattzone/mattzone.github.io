---
title:  "Smarten your Dumb Washer with Home Assitant"
categories: [homelab, homeassistant]
tags: [homelab, homeassistant, automation]
---
Make that smart appliance dream a bit more affordable.

![washer](../../assets/washer.jpg)

## A Cycle of Automation

After much nagging about my ~~desire~~ need for home automation, I was granted a basic proof of concept to validate what I already knew. My ~~boss~~ Wife wanted me to make the first automation a memorable and useful one, without spending much money, ideally none at all. So I got to work on one of the most mundane and taxing chores - Laundry.

Now, Laundry by itself is not a challenge as it was years ago. Most of the newer laundry machines are energy efficent and save on water and use less detergent too. But if you have spent a large amount on a still-functional washer/dryer set, the prospect of buying a newer set just to have that alexa/google home tie-in seems almost pointless. This was my thought as well, but seeing as my laundry room is in the basement, and our house having the Bedrooms on the 2nd floor, you want to be sure the current wash cycle is complete before making the trek down to the basement. 

But tlet's get into the details of this automation.

## The Plan

If you aren't familiar with what Home-Assistant is, I will try to write out a deep dive into exactly what it is capable of as I, too, learn these facts, but think of it as an automation ecosystem similar to Alexa. It can integrate with hundreds of other services and use that to tailor automations through a self-hosted solution. It takes the internet/cloud aspect out, and lets you have much more controll over your data.

Now to the Washer - The idea was simple enough, While a wash cycle is on, the washer would be using much more power than if it was just on, self cleaning, or in standby. So an obvious choice here was to make use of another automation ecosystem I have used for a few other purposes - [TP-Link owned Kasa smart plugs](https://amzn.to/3FerGeV). 
These plus would act as a pass-through for the washer's typcial 120Vac plug, and allow me to read the current being used.

Then it would just be a matter of figuring out how to setup a toggle for this sensor to indicate if the washer was in one of 3 states

**OFF** > **RUNNING** > **ON**

That, at least, was the idea.

## Home Assistant Configuration

Now Since the [Kasa](https://amzn.to/3yyo6Zf) Plugs also had a self [service integration with Home-Assistant](https://www.home-assistant.io/integrations/tplink/), it made this part easy enough. Once the login was complete, I had my aptly named `washer` plug added and ready to be used. I ran the washer through a single cycle to make sure I was getting some good information.

### Picking the sesonr that was just right

I Figured the Easiest way to Figure out what I should use was to take a look at all the availble sensor outputs from the Kasa plug. This gave me the few following results after scoping the time to before, during and after the cycle completed.

First was the Washer Current Sensor:
![washer_current_sensor](../../assets/home_assistant/washer/washer_current_sensor.png)

Then the Washer Current Consumption Sensor: 
![washer_consumption_sensor](../../assets/home_assistant/washer/washer_consumption_sensor.png)

Next the Voltage (not sure what you could really expect)
![washer_voltage_sensor](../../assets/home_assistant/washer/washer_voltage_sensor.png)

And finally the Total usage sensor
![washer_total_sensor](../../assets/home_assistant/washer/washer_total_sensor.png)

The obvious 2 choices were for the Current or Consumption sensors. However, the Current Had a much smaller room for deviation errors, Idle was at 0.03A and in cycle had periods of 0.17A down to just 0.1A, with an max of ~9.55A. This was also only one wash cycle and I couldn't be sure this wouldn't be a problem in my determination of state.

With the Current Consumption, however, Idle was sitting right at 0.4W, but during cycle never actually dropping below 5W, and a max of almost 750W. Just in terms of number calculations, this made more sense and so I selected the `sensor.washer_current_consumption` As the winner.

### Making this sensor do some work

Now since I had absolutely no idea how to make this work as expected, I thought just jamming the details into an Automation would work. And at first it seemed to work just fine: 

```yaml
alias: Washer Automation
description: "Notify me on Washer complete!"
mode: single
trigger:
  - type: power
    platform: device
    entity_id: sensor.washer_current_consumption
    domain: sensor
    below: 5
condition: []
action:
  - service: notify.alexa_echo_show
    data:
      message: Washer Done!
      data:
        type: tts

```

A quick run down on what this automation is actually doing:

```yaml
trigger:
  - type: power
    platform: device
    entity_id: sensor.washer_current_consumption
    domain: sensor
    below: 5
```

The trigger is what causes the automation to fire, so in this example, the automation will fire whenever the washer plug consumption sensor drops below 5W.

```yaml
action:
  - service: notify.alexa_echo_show
    data:
      message: Washer Done!
      data:
        type: tts
```

And the action is to send a TTS message over the Alexa HACS integration to tell me that the washer is done. It's as simple as that!

Or so I thought...

### There's a bug in the programming

Now the first few runs of this automation actually went great, but it didn't allow for much in terms of additional automation. So I did a bit of research and found another type of sensor that you can use. Ones that you can assign a value to with templates.

```yaml
- platform: template
  sensors:
    washer_status:
      friendly_name: 'Washer Status'
      value_template:  >
        {% set washing_state = states("sensor.washer_current_consumption") | float %}
        {% if washing_state > 3 %}
          Running
        {% elif 3 > washing_state > 0 %}
          On
        {% else %}
          Off
        {% endif %}
```
Since these are not self service in the UI, they need to be added directly to your [home-assistant configuration.yaml](https://www.home-assistant.io/docs/configuration/) as a sensor block.

The above simply reads in the washer current and sets it as a float, and then uses this value to set the washer to Running if above 3, On if less than 3 and more than 0, or Off. The numbers here associated as Watts.

Now I had a sensor with meaningful state that I could also read for transition state without adding integers to any automations I had for the washer.

Although it is not the cleanest Automation, here is the current Automation for My Washer complete cycle:

```yaml
alias: Washer Finished
description: "Run automation if Washer status transisitons from Running"
trigger:
  - platform: state
    entity_id:
      - sensor.washer_status
    from: Running
    for:
      hours: 0
      minutes: 2
      seconds: 0
condition: []
action:
  - repeat:
      count: 2
      sequence:
        - if:
            - condition: device
              type: is_on
              entity_id: light.living_room
              domain: light
          then:
            - type: turn_off
              entity_id: light.living_room
              domain: light
            - delay:
                hours: 0
                minutes: 0
                seconds: 1
                milliseconds: 0
            - type: turn_on
              entity_id: light.living_room
              domain: light
            - delay:
                hours: 0
                minutes: 0
                seconds: 1
                milliseconds: 0
          else:
            - type: turn_on
              entity_id: light.living_room
              domain: light
            - delay:
                hours: 0
                minutes: 0
                seconds: 1
                milliseconds: 0
            - type: turn_off
              entity_id: light.living_room
              domain: light
            - delay:
                hours: 0
                minutes: 0
                seconds: 1
                milliseconds: 0
  - service: notify.alexa_echo_show
    data:
      data:
        type: tts
      message: The washing machine is done.
mode: single
```

A few new toggles here, to make the automation a bit more noticable: 

```yaml
trigger:
  - platform: state
    entity_id:
      - sensor.washer_status
    from: Running
    for:
      hours: 0
      minutes: 2
      seconds: 0
```

Rather than the automation determining the washer state, the sensor now just is assigned the value that it should be, so if the sensor switches **FROM** the Running state, that should indicate the washer is complete. It otherwise will ignore a switch **TO** the Running state.


```yaml
- if:
    - condition: device
        type: is_on
        entity_id: light.living_room
        domain: light
  then:
    - type: turn_off
        entity_id: light.living_room
        domain: light
    - delay:
        hours: 0
        minutes: 0
        seconds: 1
        milliseconds: 0
    - type: turn_on
        entity_id: light.living_room
        domain: light
    - delay:
        hours: 0
        minutes: 0
        seconds: 1
        milliseconds: 0
  else:
    - type: turn_on
        entity_id: light.living_room
        domain: light
    - delay:
        hours: 0
        minutes: 0
        seconds: 1
        milliseconds: 0
    - type: turn_off
        entity_id: light.living_room
        domain: light
    - delay:
        hours: 0
        minutes: 0
        seconds: 1
        milliseconds: 0
```

If the Living Room is On, flicker the Lights off/on to indicate an automation action, or if they are off, flicker lights on/off. This makes sure the state is in the original condition and not flickering to an off state when they were on. I will look into making this cleaner, as the if/else and repeat command does make scream for refactoring.

But at the end of this Weekend Home Assistant Crash Course, I have a Slightly Smarter Washer.

The next goal is to make my Dryer smarter, using [Vibration Sensors from Aqara](https://amzn.to/3YwEK69)


### Reference

[Home-Assistant](https://www.home-assistant.io/)

[home-assistant configuration.yaml](https://www.home-assistant.io/docs/configuration/)

[Kasa Home-Assistant Integration](https://www.home-assistant.io/integrations/tplink/)

[TP-Link Kasa smart plugs](https://amzn.to/3FerGeV)

[Kasa Smart Plugs 4 pack](https://amzn.to/3yyo6Zf)

[Vibration Sensors from Aqara](https://amzn.to/3YwEK69)
