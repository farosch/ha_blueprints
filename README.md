# Home Assistant blueprints

A collection of personal Home Assistant blueprints

## COM611 keypad rules

The [keypad blueprint](com611-keypad.yaml) accepts any number of independent
rules in one automation. Each rule has a required name and Enabled/Disabled state,
device addresses, one keypad event, valid codes, and a script to run. Select
**Enabled** explicitly; a rule with any other or missing state is ignored.
The rule overview shows **Enabled** or **Disabled** beneath each rule name.
Names must be nonblank, and must not contain access codes. Create the scripts in Home Assistant
and select them in the rule editor; each script can contain any sequence of
actions.

Example rules (codes and script names are placeholders, not recommended access codes):

| Rule | Enabled | Device Addresses | Event | Valid Codes | Script |
| --- | --- | --- | --- | --- | --- |
| Function group | Enabled | 6, 7, 9 | `function` | `0001` | `script.function_group` |
| Bell group | Enabled | 8, 2 | `ring` | `0002` | `script.bell_group` |
| Door button | Enabled | 4 | `door_button` | None | `script.door_button` |

Each rule requires at least one nonblank text device address. A missing address
list, an empty list, or any blank entry makes the entire rule ineligible to run.
Incoming events must also contain a device address.

An event from any address in a rule can match; the devices do not need to send
events together. Bell events use `ring`, not `bell`. Function and bell codes
must match exactly and contain 1-8 ASCII digits. Keep codes as text to preserve
leading zeros. Door-button rules require an empty event code and do not use
the rule's code list.

**Matching Mode** defaults to **Run All Matching Rules**, in list order, waiting
for each script to finish. **Run First Matching Rule Only** runs only the first
matching enabled rule; reorder rules to set priority. Disabled rules do not match
and do not cause rejection feedback.

Scripts receive a single structured `keypad` variable with `keypad.address`,
`keypad.event`, and `keypad.rule`. Addresses and rule names remain strings, even
when they look numeric. Codes and configured code lists are never forwarded to
scripts, preventing this blueprint from exposing them through called-script
traces. The previous scalar variables are no longer provided; update scripts
to use the structured fields.

If an enabled rule matches the address/event but no code rule accepts the attempt,
the shared rejected actions run once. **Report Rejections in Activity** is enabled
by default and logs address, event, applicable rule names, and one of these reasons:

- `invalid_code_format`: missing, non-text, non-ASCII, non-digit, empty, or longer
	than eight characters for a function/bell event.
- `code_not_allowed`: a correctly formatted code is not authorized by any matching rule.
- `unexpected_door_button_code`: a door-button event did not contain an empty code.

Rejection reports never contain codes. Custom feedback actions and rule names
must also avoid exposing them. Unconfigured addresses/events are ignored.
**Rejection Cooldown** applies only after rejected actions finish and is shared
across all rules and devices. Accepted attempts have no cooldown delay. Events
while scripts, rejection feedback, or the rejection cooldown are running are
ignored, not queued. A reporting or feedback error can prevent the cooldown.

Only the rule-based configuration is supported. Existing automations must be
reconfigured with rules after updating this blueprint. The rejected actions and
cooldown are shared. No rules or scripts are enabled by default, and an empty
rule list handles no events. Keep real codes out of version control. Event
payloads contain codes; blueprint automation trace storage is disabled.

## Importing and updating

Requires Home Assistant 2026.9.4 or newer, the released object-selector schema
verified for this blueprint. Re-select **Enabled** or **Disabled** for existing
rules after upgrading; the previous boolean states are no longer accepted.
Update called scripts to use the structured metadata after upgrading.

Import this unpinned URL in Home Assistant to track updates on `main`:
https://github.com/farosch/ha_blueprints/blob/main/com611-keypad.yaml

After publishing changes to GitHub, use **Re-import blueprint** in Home Assistant
and reload automations, then reopen the automation editor.
