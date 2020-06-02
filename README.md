# sigWah
A Sigma to Wazuh / OSSEC converter

## Description
sigWah is a tool for converting [Sigma](https://github.com/Neo23x0/sigma/tree/master/rules/windows/sysmon) rules to Wazuh rules, it is **not** fully automatic and still in a very early stage.
Manual review is recommended and some cases required (it's indicated).

This repository does also contain rules generated with sigWah, these rules are manually reviewed.

### Supported rules
sigWah does not (yet) support all rule conditions, it does support:

* OR statements like: `one of $` `$ or $ or ..`
* AND NOT statements like: `$ and not $`
* AND statements like: `all of them` `$ and $ and ..`
  *  The AND statement fails to many times due wrong syntax in Sigma, so there will 
  be still a `Manual check needed!` note
  
It does not support more complex conditions, all these rules will be processed as `OR` rules and 
gets a `Manual check needed!` note, some examples:
* `selection1 and not 1 of filter*`
* `(selection1 AND NOT filter) OR (selection2 AND NOT filter)`
* `selection_1 and ( selection_2 and selection_3 ) or selection_1 and ( selection_4 and selection_5 )`

A rule may also gets a `Manual check needed!` note if:
* the field name is unknown
* it does not have a `title`, `id`, `description`, `level`, `logsource` / `EventID` or `detection` key
* somethings unexpected fails in the rule parse

### Regex converter
All detection rules will be converted to the Wazuh Regex syntax, this includes:
* Convert the Sigma wildcard `.` to the Wazuh wildcard `\.*`
* Characters escaping, according to the [documentation](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/regex.html#regex-os-regex-syntax)
  * All `\ ` in Windows Events will be processed by Wazuh as `\\`,  so  `\ ` in a Sigma rule will be `\\\\` in a 
  Wazuh rule. (Double backslashes and backslash escaping)
* Convert the Sigma Modifiers to the Wazuh syntax (`contains|all`, `endswith` and `startswith`)

In order to avoid a lot of false negatives, some extra adjustments are made:
* All spaces are replaced with `\s+`, except at start and end of string:  
It often happens that 2 spaces are behind each other, this solves this problem. Example: `WMIC.exe⋅⋅shadowcopy⋅delete`
* The following variables are replaced:
  * `C:\\` with `\.:\\`
  * `%AppData%` with `\.AppData\.`
  * `%System%` with `\.system32\.`
  * `%WinDir%` with `\.Windows\.`
  
### Rule information
The aim was to include as much information as possible for the researcher who gets the alert. The following
information from the Sigma rule is parsed to the Wazuh rule:
* The Wazuh title (`discription`) field consist of:
  * The first part: `ATT&CK {}` with the Mitre technique from `tags`
  * Followed by the title of the rule
* The Wazuh `info` field consist of:
  * The rule `description`
  * All `references` if presented
  * All `falsepositives` if presented
  * The Sigma UUID

The level conversion:
- critical: 15
- high: 14
- medium: 10
- low: 8

## Usage

```
usage: sigWah.py [-h] [-r int] [-t YYYY-MM-DD/HH:MM:SS] input_path output_path

Sigma rules converter for OSSEC and Wazuh

positional arguments:
  input_path            Path to scan for Sigma files
  output_path           Path to output the Wazuh files

optional arguments:
  -h, --help            show this help message and exit
  -r int                Rule number to start with (adds up by 10, default 250000)
  -t YYYY-MM-DD/HH:MM:SS
                        The start modified date incl. time which files need to be processed, used to only process the files which changed
```
The conversion of /windows/process_creation rules, with start rule number `260000`:

```
python sigWah.py -r 260000 sigma/rules/windows/process_creation ossec-rules/windows/process_creation
```
After the rules have been generated hit CTRL-F and search for `Manual check needed!` note, assess and adjust if necessary.

### The whitelist override problem
If you're not careful with writing level zero Wazuh rules you can make some rules unreachable. We will explain why.

All rules with the `AND NOT` condition will by default consist of 2 parts: first the detection rule, second the whitelist rule.
Lets take for example [this](https://github.com/Neo23x0/sigma/blob/master/rules/windows/process_creation/win_control_panel_item.yml)
Sigma rule, sigWah will convert it like this:

```xml
<rule id="260300" level="15">
	<if_group>sysmon_event1</if_group>
	<field name="win.eventdata.CommandLine">.cpl</field>
	<description>ATT&CK T1196: Control Panel Items</description>
	<info type="text">Detects the use of a control panel item (.cpl) outside of the System32 folder </info>
	<info type="text">Falsepositives: Unknown. </info>
	<info type="text">Sigma UUID: 0ba863e6-def5-4e50-9cea-4dd8c7dc46a4 </info>
	<group>attack.execution,attack.t1196,attack.defense_evasion,MITRE</group>
</rule>

<rule id="260301" level="0">
	<if_sid>260300</if_sid>
	<field name="win.eventdata.CommandLine">\\\\System32\\\\|system32</field>
	<description>Whitelist Interaction: Control Panel Items</description>
	<group>attack.execution,attack.t1196,attack.defense_evasion,MITRE</group>
</rule>
```
This rule will be fine, unless there is another rule which does match on `<field name="win.eventdata.CommandLine">xx .cpl</field>`
with a level of 14. Wazuh will never reach the second rule because it did already match on `260300`. There is no other rule which 
matches on `.cpl` so here it is not a problem, but we will take another one. 

For example the rule [renamed Powershell](https://github.com/Neo23x0/sigma/blob/master/rules/windows/process_creation/win_renamed_powershell.yml),
sigWah will convert it to this:

```xml
<rule id="262390" level="15">
	<if_group>sysmon_event1</if_group>
	<field name="win.eventdata.Description">Windows PowerShell</field>
	<field name="win.eventdata.Company">Microsoft Corporation</field>
	<description>ATT&CK: Renamed PowerShell</description>
	<info type="text">Detects the execution of a renamed PowerShell often used by attackers or malware </info>
	<info type="text">Falsepositives: Unknown. </info>
	<info type="text">Sigma UUID: d178a2d7-129a-4ba4-8ee6-d6e1fecd5d20 </info>
	<info type="link">https://twitter.com/christophetd/status/1164506034720952320 </info>
	<group>car.2013-05-009,MITRE</group>
</rule>

<rule id="262391" level="0">
	<if_sid>262390</if_sid>
	<field name="win.eventdata.Image">\\\\powershell.exe|\\\\powershell_ise.exe</field>
	<description>Whitelist Interaction: Renamed PowerShell</description>
	<group>car.2013-05-009,MITRE</group>
</rule>
```
In this particular case there is a problem. Each generated `event_1` with the Powershell description and company will match
`262390`, but if the image contains `\\powershell.exe` or `\\powershell_ise.exe` the level will be set to zero and no
alert will be generated. Any other rules that might have matched will not be checked anymore.

The solution is to use the `<match>` option and negate the expression:
```xml
<rule id="262390" level="15">
	<if_group>sysmon_event1</if_group>
	<field name="win.eventdata.Description">Windows PowerShell</field>
	<field name="win.eventdata.Company">Microsoft Corporation</field>
	<match>!\\powershell.exe|\\powershell_ise.exe</match>
	<description>ATT&CK: Renamed PowerShell</description>
	<info type="text">Detects the execution of a renamed PowerShell often used by attackers or malware </info>
	<info type="text">Falsepositives: Unknown. </info>
	<info type="text">Sigma UUID: d178a2d7-129a-4ba4-8ee6-d6e1fecd5d20 </info>
	<info type="link">https://twitter.com/christophetd/status/1164506034720952320 </info>
	<group>car.2013-05-009,MITRE</group>
</rule>
```

However this solution would not work for the rule `260300`, the `<match>` option will match on any field (like the image path),
not only the commandline in this particular case. So it is recommended to double check all rules with a level zero and check
if it can be converted to the negated `<match>` option to prevent the whitelist overriding.

All rules in this repository are double checked for this problem.

