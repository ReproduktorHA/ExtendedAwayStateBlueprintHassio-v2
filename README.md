# Home Assistant automation blueprint: Extended Away state

## Installation
[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FReproduktor%2FExtendedAwayStateBlueprintHassio%2Fblob%2Fmaster%2FExtendedAwayState.yaml)

Or import to HA [from GitHub](https://github.com/Reproduktor/ExtendedAwayStateBlueprintHassio/blob/64d16860f8f4637aff34cc243499ada4981d6161/ExtendedAwayState.yaml)  
`https://github.com/Reproduktor/ExtendedAwayStateBlueprintHassio/blob/64d16860f8f4637aff34cc243499ada4981d6161/ExtendedAwayState.yaml`

## Description

This automation maintains a boolean helper, indicating if a person or device is away from home for an extended (configurable) amount of time. To configure, two helpers need to be provided for the automation to function properly.

## Parameters
- **Person** An entity to track. It is designed for either a *person* entity, or *device_tracker* entity, but any entity which indicates home presence by being set to state 'home' will work
- **Extended away indicator** A helper of type `input_boolean`. This value will be set by this automation to `on`, when the person is away from home for more than the defined amount of time. The value will be reset to `off` as soon as the person returns home.
- **Extended away time** Time, in hours, after which the person is considered away for an extended amount of time.
- **Timer** A helper of type `timer`, which is needed by this automation to count down when the person leaves from home.

## Hints
I use this automation for every home member independently. My schema is:
1. Create `input_boolean` helpers for each person: `input_boolean.extendedaway_person1`, `input_boolean.extendedaway_person2`, ...
2. Create `timer` helpers for each person: `timer.extendedawaytimer_person1`, `timer.extendedawaytimer_person2`, ...
3. Based on this blueprint, create automations for each person: `Maintain an 'extended away' state for Person1`, `Maintain an 'extended away' state for Person2`, ...

## Practical applications
I use the state for controlling, to whom the notifications from Home Assistant will be sent. In my scenario, I have a script which delivers a given message to family members, that are home-ish. The script is like this:
```
notify_family_members_at_home_these_days:
  alias: Notify family members at home these days
  icon: mdi:bell-badge-outline
  description: "Whoever is not away for extended period of time, gets notified."
  mode: queued
  max: 10
  fields:
    message:
      name: Message
      description: "Message to send"
      selector:
        text:
    title:
      name: Title
      description: "Title of the message to send"
      selector:
        text:
  sequence:
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.extendedaway_person1
        state: 'off'
      sequence:
      - service: notify.mobile_app_person1
        data:
          message: "{{ message }}"
          title: "{{ title }}"
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.extendedaway_person2
        state: 'off'
      sequence:
      - service: notify.mobile_app_person2
        data:
          message: "{{ message }}"
          title: "{{ title }}"

... etc ...

    default: []

```
