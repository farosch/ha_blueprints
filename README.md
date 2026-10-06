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

Both blueprints require at least one nonblank text device address per configured rule. A missing address list, an empty list, or any blank entry makes the entire rule ineligible to run. Incoming events must also contain a device address.
The fixed 20-rule editor cannot enforce conditional required fields when saving enabled rules without valid addresses are ignored. Unused disabled rules can remain empty.

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

Requires Home Assistant 2024.10.0 or newer. After publishing the new file, import:
https://github.com/farosch/ha_blueprints/blob/main/com611-keypad-actions.yaml

## COM611 keypad with dynamic If/Then rules

The [sequential blueprint](com611-keypad-sequential.yaml) accepts an unlimited
list of named native If/Then blocks with inline actions. No separate scripts or
fixed rule slots are required. Choose **Run All Matching Rules** (the default) or
**Run First Matching Rule Only**. Rule order determines priority in first-match
mode. An omitted rule `enabled` field means enabled; set it to `false` to disable
a rule.

### Device cooldown setup

Create one dedicated Timer helper per device address in **Settings > Devices &
services > Helpers**. Enable the timer's **Restore** option to preserve active
cooldowns across restarts. Map each address to its timer in the blueprint's
**Devices and Cooldown Timers** input. Addresses and timers must be unique within
the automation, and timers must not be shared with other automations.

For example, the `devices` input can contain:

```yaml
devices:
	- address: "6"
		timer: timer.keypad_6
	- address: "7"
		timer: timer.keypad_7
	- address: "9"
		timer: timer.keypad_9
	- address: "8"
		timer: timer.keypad_8
	- address: "2"
		timer: timer.keypad_2
	- address: "4"
		timer: timer.keypad_4
```

An attempt starts that device's cooldown before rule evaluation, including on
rejection. There is no global cooldown delay. A timer that is active, paused,
missing, or unavailable prevents actions for that device. Stale queued attempts
are discarded rather than replayed after cooldown. Runs are queued to serialize
the timer check/start operation; a long-running inline action can therefore delay
other devices, but the cooldown itself does not hold the queue. At most 100 runs
can be running or queued. Excess events are discarded.

### Adding rules

Use **Named If/Then Rules** to add, remove, name, and reorder rules. Each rule's
**If Conditions** uses the normal condition editor; **Then Actions** uses the
normal action editor. Conditions can use `keypad.address`, `keypad.event`, and
`keypad.code`. The `keypad` dictionary preserves codes as text, including leading
zeros. Include an explicit address condition and exact code test in every
function/bell rule. Door-button rules do not require a code test because the
blueprint already requires an empty incoming code for that event.

The native list is inserted directly with `!input` when the automation loads.
It cannot be transformed or inspected at runtime without prematurely rendering
its templates. Consequently, each rule needs two pieces of bookkeeping:

- Keep the match-mode template as the first If condition.
- Keep the Variables action as the first Then action. Its `keypad_rule` must
	match the rule name. Keep the three assignments in the order shown below.

These steps implement first-match mode, record matched rule names, and prevent
rejection feedback after successful matches. Removing them can break those
features. The blueprint validates device/timer configuration, not the contents
of arbitrary native rules. Do not change its bookkeeping variables in later
actions. A Stop action or an action failure ends the run before its final
diagnostics/rejection handling.

The following `rules` input illustrates three groups. Codes and helper names
are placeholders; do not publish real codes. Replace the harmless log actions
after Variables with the inline actions you want:

```yaml
rules:
	- alias: Function group
		if:
			- condition: template
				value_template: "{{ keypad_match_mode == 'all' or not keypad_matched }}"
			- condition: template
				value_template: >-
					{{ keypad.address in ['6', '7', '9']
						 and keypad.event == 'function' and keypad.code == '0001' }}
		then:
			- variables:
					keypad_rule: Function group
					keypad_matched: true
					keypad_matched_rules: "{{ keypad_matched_rules + [keypad_rule] }}"
			- action: logbook.log
				data:
					name: Function group
					message: Accepted
	- alias: Bell group
		if:
			- condition: template
				value_template: "{{ keypad_match_mode == 'all' or not keypad_matched }}"
			- condition: template
				value_template: >-
					{{ keypad.address in ['8', '2']
						 and keypad.event == 'ring' and keypad.code == '0002' }}
		then:
			- variables:
					keypad_rule: Bell group
					keypad_matched: true
					keypad_matched_rules: "{{ keypad_matched_rules + [keypad_rule] }}"
			- action: logbook.log
				data:
					name: Bell group
					message: Accepted
	- alias: Door button
		if:
			- condition: template
				value_template: "{{ keypad_match_mode == 'all' or not keypad_matched }}"
			- condition: template
				value_template: "{{ keypad.address == '4' and keypad.event == 'door_button' }}"
		then:
			- variables:
					keypad_rule: Door button
					keypad_matched: true
					keypad_matched_rules: "{{ keypad_matched_rules + [keypad_rule] }}"
			- action: logbook.log
				data:
					name: Door button
					message: Accepted
```

Registered devices with invalid payloads or no matching rule run the shared
rejected actions once. Unregistered devices and unsupported events are ignored.
With an empty rule list, registered devices are rejected, never authorized.

**Safe Diagnostics** is off by default. When enabled, built-in Activity entries
contain only address, event, outcome, and matched rule names; they omit codes.
Keep codes out of rule names and your own logs/actions. Event payloads still
contain codes, and custom actions can expose them. Trace storage remains disabled.

Requires Home Assistant 2024.10.0 or newer. After publishing, import:
https://github.com/farosch/ha_blueprints/blob/main/com611-keypad-sequential.yaml
