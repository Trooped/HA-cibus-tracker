# HA-cibus-tracker
Track your Cibus amount using HA, an Android phone, and by following this short guide.

This project uses the Home Assistant Companion App for Android to read push notifications from your phone whenever Cibus charges you (like for a Wolt order). It automatically extracts the exact amount and deducts it from a monthly budget, automatically resetting on the 1st of every month.

## Table of Contents
- [Requirements](#requirements)
- [Step 1: Enable the last_notification Sensor](#step-1-enable-the-last_notification-sensor)
- [Step 2: Create a Helper Entity to Store the Balance](#step-2-create-a-helper-entity-to-store-the-balance)
- [Step 3: Create the Automation](#step-3-create-the-automation)
- [Step 4: Test Your Setup](#step-4-test-your-setup)
- [Step 5: How to use it?](#step-5-how-to-use-it?)

## Requirements
- Home Assistant (Core)
- Android Phone with the Home Assistant Companion App installed and connected.
- Notification Permissions granted to the Home Assistant app on your phone.

## Step 1: Enable the last_notification Sensor
By default, Home Assistant does not read your phone's notifications. You need to enable this specific sensor in the companion app:

1. Open the Home Assistant app on your Android phone.
2. Open the sidebar menu and go to Settings > Companion App.
3. Tap on Manage Sensors.
4. Scroll down to the Last Notification sensor and enable it.
5. Your phone will prompt you to grant "Notification Access" to Home Assistant. Allow it.
6. **(Optional but very recommended)**: Under the sensor settings, look for the Allow List and add the app package name for Cibus. This prevents HA from tracking every single notification on your phone and saves battery! If you can't find Allow List, go back and enter the last_notification sensor again.

Take note of your new sensor's entity ID (e.g., sensor.sm_s936b_last_notification). You will need this for the automation. Find it by clicking on 'e' inside HA, and searching for "last_notification".

## Step 2: Create a Helper Entity to Store the Balance
You need a place to store your current Cibus balance. You can use either an input_number or a counter helper.

**Which one should you choose?**

- input_number (Recommended): Supports decimal points (אגורות). Perfect if some of your charges look like 42.30 ₪.

- counter: Only supports whole integers (no decimals). If a charge has a decimal, the automation will round it down to the nearest whole number to prevent breaking the counter.

**How to create it:**

1. In Home Assistant, go to Settings > Devices & Services > Helpers.
2. Click + Create Helper.
3. Select Number (for input_number) or Counter.
4. Name it cibus_amount.
5. Set the maximum value you receive in the beginning of each month (e.g., 990) and the minimum value to 0.
6. Set the current amount you have by going to Settings > Developer Tools > States > Set State > Choose your new input_number.cibus_amount entity and set the 'state' to the current amount you have in your Cibus.

## Step 3: Create the Automation
Now, create the automation that links the notification sensor to your helper, and resets it each month.

1. Go to Settings > Automations & Scenes.
2. Click + Create Automation > Edit in YAML (top right three dots).
3. Paste the code below.

**Note:** Be sure to change sensor.your_phone_last_notification to your actual sensor's entity ID, also - change the initial amount each month to your initial amount. The example below uses an input_number. If you chose a counter, see the note at the bottom of the code block.

```yaml
alias: Cibus Budget Tracker
description: "Deducts Cibus charges from a monthly budget and resets on the 1st of the month."
mode: single
trigger:
  - platform: template
    value_template: "{{ now().day == 1 }}"
    id: first_of_month
  - platform: state
    entity_id: sensor.your_phone_last_notification
    id: cibus

condition:
  - condition: template
    value_template: >
      {% if trigger.id == 'first_of_month' %}
        true
      {% elif trigger.id == 'cibus' %}
        {{ trigger.to_state is not none and trigger.to_state.state is search('הסיבוס שלך חוייב') }}
      {% else %}
        false
      {% endif %}

action:
  - variables:
      cibus_charge: >
        {% if trigger.id == 'cibus' %}
          {{ trigger.to_state.state | regex_findall_index('חוייב ב ([0-9.]+) ₪') | float(0) }}
        {% else %}
          0.0
        {% endif %}
  - choose:
      # Action 1: Deduct the charge when a notification arrives
      - conditions:
          - condition: trigger
            id: cibus
        sequence:
          - action: input_number.set_value
            target:
              entity_id: input_number.cibus_amount
            data:
              value: >-
                {{ (states('input_number.cibus_amount') | float(0)) - (cibus_charge | float(0)) }}
              
      # Action 2: Reset to initial value on the first day of the month
      - conditions:
          - condition: trigger
            id: first_of_month
        sequence:
          - action: input_number.set_value
            target:
              entity_id: input_number.cibus_amount
            data:
              value: 990
```

**Using a counter instead?**
If you opted for a counter helper instead of input_number, change the actions in the YAML to use action: counter.set_value, target counter.cibus_amount, and change the math filters from | float(0) to | int(0) to prevent decimal errors.

## Step 4: Test Your Setup
You can test this without actually ordering food!

1. Go to Settings > Developer Tools > States.
2. Find your phone's notification sensor (e.g., sensor.sm_s936b_last_notification).
3. In the "State" box at the top, paste a fake notification string:

עבור עסקה במסעדת Wolt - Wolt Gift Card הסיבוס שלך חוייב ב 42 ₪

4. Click Set State - check your cibus_amount helper, it should have automatically dropped by 42!
5. Make sure to set the state of the cibus_amount back to the exact amount after testing.

## Step 5: How to use it?
Instead of manual tracking, this automation serves as a Cibus Budget Optimizer to ensure no money is lost at the end of the month.

**The Logic**
The automation calculates your Required Daily Burn by dividing your current balance by the remaining working days (skipping Fridays and Saturdays by default, but it's configurable).

**The Trigger:** If your Required Burn > 100 ILS (or your daily limit, configurable), you are officially in "Use it or Lose it" territory.

**Smart Alerts:** It notifies you at 11:00 (Lunch) and 18:30 (Groceries/dinner), but only if you are over-budget.

**Dynamic:** If you spend enough at lunch to bring the ratio back under control, the evening notification won't fire.

**Configurable:** You can set your own off days (days you can't use your Cibus) and daily maximum budget, in the variables section in the automation.

It’s a "set it and forget it" tool that only nags you when you literally cannot finish your budget by spending the daily maximum anymore.

```yaml
alias: Cibus Spending Notifications
description: >-
  Monitors Cibus balance vs. remaining workdays. Notifies at lunch and evening if you need to spend extra money today to avoid having leftover credit at the end of the month.
triggers:
  - at: "11:00:00"
    trigger: time
  - at: "18:30:00"
    trigger: time
actions:
  - variables:
      daily_max: 10
      off_days:
        - 4
        - 5
      cibus_entity: input_number.cibus_amount
      balance: "{{ states(cibus_entity) | float(0) }}"
      usable_days: >
        {% set last_day = (now().replace(day=1) +
        timedelta(days=32)).replace(day=1) - timedelta(days=1) %} {% set ns =
        namespace(count=0) %} {% for d in range(now().day, last_day.day + 1) %}
          {% if now().replace(day=d).weekday() not in off_days %}
            {% set ns.count = ns.count + 1 %}
          {% endif %}
        {% endfor %} {{ ns.count | int(1) }}
      required_burn: "{{ (balance / (usable_days | float(1))) | round(2) }}"
  - if:
      - condition: template
        value_template: |
          {{ now().weekday() not in off_days and required_burn > daily_max }}
    then:
      - action: notify.mobile_app_galaxy
        data:
          title: 🍱 Cibus Credit Alert!
          message: >
            You have {{ balance }} ILS left and only {{ usable_days }} usable
            days this month. You need to spend {{ required_burn }} ILS today,
            but your limit is {{ daily_max }}! Buy food and don't lost money!
          data:
            notification_icon: mdi:food-off
            clickAction: app://com.sodexo.mysodexo

```
