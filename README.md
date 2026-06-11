# Rule-Based Compliance

In Catalyst Center v3.1.5 and above, the new Rule-Based Compliance policy feature was added to address the need for custom, granular, regex-based configuration introspection to ensure devices are in compliance with customers' standards.  This feature is very similar to Cisco Prime Infrastructure's Configuration Compliance Auditing feature, which was the original basis for the feature request in Catalyst Center.

In this repository, we will describe the functionality, limitations, and detail specific example policies to help you understand how to leverage Rules-Based Compliance in your own Catalyst Center environment.

## Requirements & Limitations

- Catalyst Center v3.1.5 or greater

Recommended capacity limitations:

| Category | System Limits | Individual Limits |
| -------- | ------------- | ----------------- |
| Policy   | 500           | N/A               |
| Rules    | 5000          | Max 20 per Policy |
| Variables | 12,500 (50% of rules use variables, avg. 5 per rule) | Max 10 per Rule |
| Conditions | 25,000 (avg 5 per rule) | Max 10 per Rule |

> *[Catalyst Center User Guide](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center/3-1-x/user_guide/b_cisco_catalyst_center_user_guide_3_1_x/m-configure-rule-based-compliance-policies.html#_35e9d03a-1db5-4fff-a64c-e3c1fae1dbda)*

## Structure of a Rule-Based Compliance Policy

Rule-Based Compliance Polices are constructed in a modular format, which consists of a top-level Policy containing one or more compliance Rules.  Each rule contains one or more Conditions which define the configuration "signatures" to check for and the action to take.

Below is a diagram depicting the relationship of each of the components of a Rule-Based Compliance Policy:

![rbc_structure.png](/assets/rbc_structure.png)

- **Policy:** A Policy is the top-level structure of Rule-Based Compliance.  Policies are assigned to one or more Sites in the Network Hierarchy of Catalyst Center.
- **Rule:** A Rule is the first container level inside a Policy.  Rules are used to group together individual Conditions (specific configuration signature checks) and relate them to one or more Device families (Routers, Switches & Hubs, or even specific device model numbers) and Software platforms (IOS, IOS XE, etc.).
- **Variable:** Rules may contain zero or more Variables, which can be referenced inside multiple Conditions.  Variables can be one of the following data types:
  - **String:** A string of alpha-numeric text, treated as plain text.
  - **Integer:** A whole number, positive or negative.
  - **Boolean:** A binary value which can be either `True` or `False`.
  - **IP address:** A valid IPv4 address between `0.0.0.0` and `255.255.255.255`.
  - **Interface:** A string of text representing the expanded full name of a device interface, along with any module/slot/port numbers.  DO NOT use IOS abbreviations for interface names because they will not match the contents of a device's running configuration.  Also, ensure that the correct expanded full name of the interface is used; some interface names are abbreviated for length in IOS XE (ex: TwentyFiveGigE, HundredGigE). [IOS XE Interface Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-15/command_reference/b_1715_9600_cr/interface_and_hardware_commands.html#wp1054269631)
  - **IP mask:** A valid IPv4 Subnet Mask between `0.0.0.0` and `255.255.255.255` (automatically checked for valid values).
- **Condition:** An individual, atomic compliance check to be run against each target device.  Conditions can check the following inputs:
  - **Device configuration:** The device's running-configuration output.
  - **Device command outputs:** The retrieved response from Catalyst Center running a specified CLI command on the target device.
  - **Device properties:** A small list of known device properties, which Catalyst Center has stored.
  - **Previously matched blocks:** A very powerful feature that allows Catalyst Center to capture the block of text matched in a previous condition and use that as input for the next condition check.

## Component Deep Dive

### Rules

When creating a new Rule under a Policy, the following fields are available for you to populate:

- **Rule name**
- **Description**
- **Impact:** A text field allowing you to describe the expected impact of violations of this rule.
- **Suggested fix:** A text field allowing you to describe suggestions for resolving violations of this rule.
- **Software type:** Select from `IOS`, `IOS-XE`, `Cisco Controller`.
- **Device family:** Select from `Routers`, `Switches and Hubs`, `Wireless Controller`
- **Device Series** & **Device Model:** Select specific device model series groups, or even specific device models to more granularly control what devices this Rule should apply to.

Once you've completed the necessary fields to create your Rule, you can then begin adding Conditions and, optionally, Variables.

### Variables

Variables allow you to create placeholders which can be referenced in multiple Condition statements.  Variables can be configured as one of several Data Types, which allow you to control what values are allowed to be stored in a Variable.

Once you've defined a Variable, you can use it as part (or all) of a Condition search expression, as well as any custom violation message that the Condition might trigger.  This level of flexibility allows you to define common configuration parameters one time and then reuse them across multiple Conditions, preventing duplication of efforts.  You can also define different Variable values for various locations in your Site Hierarchy - essentially, site-specific values.

Here are just a few examples of what you might store in a Variable:

- SNMP Server IP Addresses
- NTP Server IP Addresses
- SNMP Community Strings
- Interface Descriptions
- ACL Names or Numbers
- Static Routes
- VRF Names
- Policy Names
- DHCP Option strings
- Local Usernames

As mentioned above, there are several Data Types that you can choose from when creating a Variable.  All Variables must have a "Variable name" and an "Identifier", which you can create on your own or have Catalyst Center generate automatically.  The variable "Identifier" is the unique name used within your condition statements to reference the variable, and it must meet these criteria:

- Starts with an underscore `_`
- Can only contain letters, numbers or underscores
- Can be a maximum of 51 characters in length

Many of these Data Types have additional configuration parameters that are relevant certain types, so we'll cover those in detail below:

#### String

