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
```yaml
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

# ... etc ...

    default: []

```

## Blueprint
```yaml
blueprint:
  name: Maintain an 'extended away' state for a person
  description: Select a person, a boolean helper, the desired time period, and a timer helper,
      and this automation will take care of maintaining the provided boolean with a value indicating
      if the given person is away from home for more than the given amount of time.
  domain: automation
  input:
    person:
      name: Person
      description: 'Person, or tracker, to watch (e.g. person.dad, or device_tracker.iphone).'
      selector:
        entity:
    extended_away_boolean:
      name: Extended away indicator
      description: 'A helper of type input_boolean to set.'
      selector:
        entity:
          domain: input_boolean
    extended_away_time:
      name: Extended away time
      description: 'The time, after which a person is considered to be Extended Away (in hours).'
      default: 24
      selector:
        number:
          min: 1
          max: 168
          step: 1
          unit_of_measurement: hrs
          mode: box
    extended_away_timer:
      name: Timer
      description: 'A helper timer this automation needs to keep track of the time a person is being away from home.'
      selector:
        entity:
          domain: timer
  source_url: https://github.com/Reproduktor/ExtendedAwayStateBlueprintHassio/blob/master/ExtendedAwayState.yaml

trigger:
- platform: event
  event_type: timer.finished
  id: timerfinished
  event_data:
    entity_id: !input 'extended_away_timer' #timer.extendedawaytimer_ocino
- platform: state
  entity_id: !input 'person'
  id: home-entered
  to: home
  for:
    hours: 0
    minutes: 0
    seconds: 0
- platform: state
  entity_id: !input 'person'
  id: home-left
  from: home
- platform: homeassistant
  event: start
  id: hass-started
- platform: time_pattern
  id: hass-midnight
  minutes: '01'
  seconds: '30'
  hours: '0'

condition: []

variables:
  var_hours: !input 'extended_away_time'
  var_seconds: "{{ var_hours | int * 3600 }}"

action:
- choose:
  - conditions:
    - condition: trigger
      id: timerfinished
    sequence:
    - service: input_boolean.turn_on
      target:
        entity_id: !input 'extended_away_boolean'

  - conditions:
    - condition: trigger
      id: home-left
    sequence:
    - service: timer.start
      data:
        duration: '{{ var_seconds }}'
      target:
        entity_id: !input 'extended_away_timer'

  - conditions:
    - condition: trigger
      id: home-entered
    sequence:
    - service: timer.cancel
      target:
        entity_id: !input 'extended_away_timer'
    - service: input_boolean.turn_off
      target:
        entity_id: !input 'extended_away_boolean'

  - conditions:
    - condition: or
      conditions:
      - condition: trigger
        id: hass-started
      - condition: trigger
        id: hass-midnight
    sequence:
    - choose:
      - conditions:
        - condition: not
          conditions:
          - condition: state
            entity_id: !input 'extended_away_timer'
            state: active
          - condition: state
            entity_id: !input 'person'
            state: home
          - condition: state
            entity_id: !input 'extended_away_boolean'
            state: 'on'
        sequence:
        - service: timer.start
          data:
            duration: '{{ var_seconds }}'
          target:
            entity_id: !input 'extended_away_timer'
      - conditions:
        - condition: state
          entity_id: !input 'person'
          state: home
        sequence:
        - service: input_boolean.turn_off
          target:
            entity_id: !input 'extended_away_boolean'
      default: []

  default: []

mode: queued
max: 5
```
