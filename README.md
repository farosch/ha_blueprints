# Home Assistant blueprints

A collection of personal Home Assistant blueprints

## COM611 keypad rules

The [keypad blueprint](com611-keypad.yaml) accepts any number of independent
rules in one automation. Each rule has a name, device addresses, one keypad
event, valid codes, and a script to run. Create the scripts in Home Assistant
and select them in the rule editor; each script can contain any sequence of
actions.

For example, the `rules` blueprint input can contain the following entries.
The codes and script names below are placeholders, not recommended access codes:

```yaml
rules:
	- name: Function group
		addresses: ["6", "7", "9"]
		event: function
		codes: ["0001"]
		script: script.function_group
	- name: Bell group
		addresses: ["8", "2"]
		event: ring
		codes: ["0002"]
		script: script.bell_group
	- name: Door button
		addresses: ["4"]
		event: door_button
		codes: []
		script: script.door_button
```

An event from any address in a rule can match; the devices do not need to send
events together. Bell events use `ring`, not `bell`. Function and bell codes
must match exactly and contain 1-8 ASCII digits. Keep codes as text to preserve
leading zeros. Door-button rules require an empty event code and do not use
the rule's code list.

All matching rules run in list order, waiting for each script to finish. Scripts
receive `keypad_address`, `keypad_event`, `keypad_code`, and `keypad_rule` as
variables. If an address/event is configured but no code rule matches, the
shared rejected actions run once. Unconfigured addresses/events are ignored.
The attempt cooldown applies to the whole automation, across all rules and
devices; events while actions or the cooldown are running are ignored.

Only the rule-based configuration is supported. Existing automations must be
reconfigured with rules after updating this blueprint. The rejected actions and
cooldown are shared. No rules or scripts are enabled by default, and an empty
rule list handles no events. Keep real codes out of version control. Event
payloads contain codes; blueprint automation trace storage is disabled.

After publishing changes to GitHub, use **Re-import blueprint** in Home Assistant
and reload automations, then reopen the automation editor.

## COM611 keypad with inline actions

The [20-rule action blueprint](com611-keypad-actions.yaml) is an alternative
that does not require separate scripts. It provides 20 collapsed rule sections,
each with an enable switch, multiple device addresses, one keypad event,
multiple valid codes, and Home Assistant's normal action editor.

Expand a rule, configure its fields and actions, and enable it. All rules start
disabled. Select `ring` for the bell event; `door_button` does not use the code
list and requires an empty code in the incoming event. All matching enabled
rules run in numeric order. Rejected actions and the cooldown are shared across
the automation, with the same code-validation and event-filtering behavior as
the script-based blueprint. Inline action templates can access `trigger.event.data`.

Requires Home Assistant 2024.6.0 or newer. After publishing the new file, import:
https://github.com/farosch/ha_blueprints/blob/main/com611-keypad-actions.yaml
