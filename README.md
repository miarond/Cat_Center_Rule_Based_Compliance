# Rule-Based Compliance

In Catalyst Center v3.1.5 and above, the new Rule-Based Compliance policy feature was added to address the need for custom, granular, regex-based configuration introspection to ensure devices are in compliance with customers' standards.  This feature is very similar to Cisco Prime Infrastructure's Configuration Compliance Auditing feature, which was the original basis for the feature request in Catalyst Center.

In this repository, we will describe the functionality, limitations, and detail specific example policies to help you understand how to leverage Rule-Based Compliance in your own Catalyst Center environment.

# Download the Examples

:bulb: **Click this link to download the Example Policy from this tutorial!**

[Example_Policy.json](/assets/Example_Policy.json)

# Table of Contents

- [Requirements & Limitations](#requirements--limitations)
- [Structure of a Rule-Based Compliance Policy](#structure-of-a-rule-based-compliance-policy)
- [Component Deep Dive](#component-deep-dive)
  - [Rules](#rules)
  - [Variables](#variables)
    - [String](#string)
    - [Integer](#integer)
    - [Boolean](#boolean)
    - [IP Address](#ip-address)
    - [Interface](#interface)
    - [IP Mask](#ip-mask)
  - [Conditions](#conditions)
    - [Fields](#fields)
- [Regular Expressions](#regular-expressions)
  - [Key RegEx Concepts](#key-regex-concepts)
  - [RegEx Examples](#regex-examples)
    - [Check Interface 802.1x Config](#example-1)
    - [Check Device Certificate Expiration](#example-2)
    - [Check Console & VTY Lines for Exec Timeout](#example-3)

[⤴️ ToC](#table-of-contents)

---

# Requirements & Limitations

- Catalyst Center v3.1.5 or greater

Recommended capacity limitations:

| Category | System Limits | Individual Limits |
|:-------- |:------------- |:----------------- |
| Policy   | 500           | N/A               |
| Rules    | 5000          | Max 20 per Policy |
| Variables | 12,500 (50% of rules use variables, avg. 5 per rule) | Max 10 per Rule |
| Conditions | 25,000 (avg 5 per rule) | Max 10 per Rule |

> *[Catalyst Center User Guide](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center/3-1-x/user_guide/b_cisco_catalyst_center_user_guide_3_1_x/m-configure-rule-based-compliance-policies.html#_35e9d03a-1db5-4fff-a64c-e3c1fae1dbda)*

[⤴️ ToC](#table-of-contents)

---

# Structure of a Rule-Based Compliance Policy

Rule-Based Compliance Polices are constructed in a modular format, which consists of a top-level Policy containing one or more compliance Rules.  Each rule contains one or more Conditions which define the configuration "signatures" to check for and the action to take.

Below is a diagram depicting the relationship of each of the components of a Rule-Based Compliance Policy:

![rbc_structure.png](/assets/rbc_structure.png)

- **Policy:** A Policy is the top-level structure of Rule-Based Compliance.  Policies are assigned to one or more Sites in the Network Hierarchy of Catalyst Center.
- **Rule:** A Rule is the first container level inside a Policy.  Rules are used to group together individual Conditions (specific configuration signature checks) and relate them to one or more Device families (Routers, Switches & Hubs, or even specific device model numbers) and Software platforms (IOS, IOS XE, etc.).
- **Variable:** Rules may contain zero or more Variables, which can be referenced inside multiple Conditions.  Variables can be one of the following data types:
  - **String:** A string of alpha-numeric text, treated as plain text.
  - **Integer:** A positive whole number, between `0` and `2,147,483,647` (31 bits).
  - **Boolean:** A binary value which can be either `True` or `False`.
  - **IP address:** A valid IPv4 address between `0.0.0.0` and `255.255.255.255`.
  - **Interface:** A string of text representing the expanded full name of a device interface, along with any module/slot/port numbers.  DO NOT use IOS abbreviations for interface names because they will not match the contents of a device's running configuration.  Also, ensure that the correct expanded full name of the interface is used; some interface names are abbreviated for length in IOS XE (ex: TwentyFiveGigE, HundredGigE). [IOS XE Interface Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-15/command_reference/b_1715_9600_cr/interface_and_hardware_commands.html#wp1054269631)
  - **IP mask:** A valid IPv4 Subnet Mask between `0.0.0.0` and `255.255.255.255` (automatically checked for valid values).
- **Condition:** An individual, atomic compliance check to be run against each target device.  Conditions can check the following inputs:
  - **Device configuration:** The device's running-configuration output.
  - **Device command outputs:** The retrieved response from Catalyst Center running a specified CLI command on the target device.
  - **Device properties:** A small list of known device properties, which Catalyst Center has stored.
  - **Previously matched blocks:** A very powerful feature that allows Catalyst Center to capture the block of text matched in a previous condition and use that as input for the next condition check.

[⤴️ ToC](#table-of-contents)

---

# Component Deep Dive

## Rules

When creating a new Rule under a Policy, the following fields are available for you to populate:

- **Rule name**
- **Description**
- **Impact:** A text field allowing you to describe the expected impact of violations of this rule.
- **Suggested fix:** A text field allowing you to describe suggestions for resolving violations of this rule.
- **Software type:** Select from `IOS`, `IOS-XE`, `Cisco Controller`.
- **Device family:** Select from `Routers`, `Switches and Hubs`, `Wireless Controller`
- **Device Series** & **Device Model:** Select specific device model series groups, or even specific device models to more granularly control what devices this Rule should apply to.

Once you've completed the necessary fields to create your Rule, you can then begin adding Conditions and, optionally, Variables.

[⤴️ ToC](#table-of-contents)

---

## Variables

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

Many of these Data Types have additional configuration parameters that are relevant to certain types, so we'll cover those in detail below.

[⤴️ ToC](#table-of-contents)

### String

In addition to a Variable name and Identifier (and optional Description), String Variables have the following configuration options:

- **Input required:** A checkbox option which dictates whether the variable MUST have a value (checked) before you can save your changes, or if it can remain blank (unchecked).  Whether or not you check this box, you can still define site-specific values for this variable on the Policy configuration screen:
  - ![site-specific_variables_button.png](/assets/site-specific_variables_button.png)
- **Is list of values:** A checkbox option which, when checked, will create a single-select dropdown box for the variable and populate it with only the specific values you define.
- **Accept multiple values:** Checking this checkbox will convert a dropdown box (from the option above) to a multi-select dropdown, or if the option above is NOT selected, multiple values can be entered in freeform text boxes.
- **Default value:** The default value for the string, if no other value is provided.
- **Maximum length:** Limits the string to X number of characters in length.  ***The maximum length accepted is 255 characters.***
- **Valid regex:** A Regular Expression that is used to validate String input values.  This option can be used to ensure that input values contain only acceptable characters or conform to a specific pattern.  *We will cover Regular Expressions in detail later in this guide.*

### Integer

Integer data types have the same options available as String data types, with the following caveats:

- **Default, Minimum and Maximum value:** These fields only accept positive whole numbers between `0` and `2,147,483,647` (31 bits).
- **Valid regex:** This option is *NOT* available because Regular Expressions are only used for string matching.

### Boolean

Boolean data types only have two options: **Input required** and **Is list of values**.  These two options have the same effect as detailed in the String section above.

### IP Address

IP Address data types have only three options: **Input required**, **Accept multiple values** and **Default value**.  All inputs will be checked automatically to ensure they are valid IPv4 addresses.

### Interface

Interface data types are treated as Strings and have the three following options: **Input required**, **Accept multiple values** and **Default value**.  The "Default value" field and any inputs created when "Accept multiple values" is checked must adhere to the following validation rules:

- First character must be a letter
- Length must be 2 or more characters
- No special characters are allowed, with the exception of: `/` `-` `_`
- All forward slashes (`/`) must be proceeded AND followed by numerical digits only.  For example:
  - Valid: `GigabitEthernet1/0/1`
  - Invalid: `GigabitEthernet/0/B`

### IP Mask

An IP Mask data type is just a Subnet Mask in IPv4 format.  "IP mask" data types must be formatted exactly like "IP address" data types however, the Subnet Mask must be one of the following values:

```
255.0.0.0
255.128.0.0
255.192.0.0
255.224.0.0
255.240.0.0
255.248.0.0
255.252.0.0
255.254.0.0
255.255.0.0
255.255.128.0
255.255.192.0
255.255.224.0
255.255.240.0
255.255.248.0
255.255.252.0
255.255.254.0
255.255.255.0
255.255.255.128
255.255.255.192
255.255.255.224
255.255.255.240
255.255.255.248
255.255.255.252
```

[⤴️ ToC](#table-of-contents)

---

## Conditions

Conditions are the heart of a Rule-Based Compliance Policy in Catalyst Center.  They allow you to parse the text contained in whatever Scope you choose as your input, search for string patterns that you define, and take action based on the results.  Conditions can use simple sub-string matches (such as "XYZ contains 'X'") or complex Regular Expressions (such as `^X[a-zA-Z]{2}$`) to search for text patterns within the input text.  If a match is found or not found, you can specify what action should be taken, including "Continue" or "Raise violation and continue", which allows you to perform subsequent conditional checks.

Both the search string/regex pattern and the "Custom violation message" can make use of any Variables that are defined within the Rule, by enclosing the Variable "identifier" inside of `<>` symbols.  For example, a Variable with the identifier `_Hostname` can be used in a search string, evaluation expression or custom violation message by entering it as: `<_Hostname>`.  This causes the `_Hostname` variable value to be substituted into the string when the Condition check runs.

In the sections below, we'll go into deep detail on how to create and customize Conditions.

### Fields

- **Sample conditions:** A prepopulated list of example Conditions that are provided for your use.
- **Advanced settings toggle:** Enables the option to parse the input text as "blocks", requiring you to define a "Block start expression" (RegEx) and optionally a "Block end expression".  Each matched block can be passed on to the next Condition in your Rule for processing.  Also adds the "Scope" options `Device properties` and `Previously matched blocks`, and breaks out the "Action" section into `Match` and `Does not match` sections.
- **Scope:** Select from a short list of available inputs which currently include:
  - `Configuration`: The device's current running configuration.
  - `Device command outputs`: Uses the output from a "show" command as the input for the condition.  Selecting this option causes the `Show command` text box to appear below, which allows you to enter any valid Show command that can be run on the target device(s).
  - `Device properties`: Provides a short list of device properties (collected by Catalyst Center) to use as input.  Selecting this option causes the `Property` dropdown menu to appear.  Currently, you can choose from: `Device name`, `IP address`, `OS name`, `OS version`.
  - `Previously matched blocks:` Uses the matched block of text from the previous Condition as the input.  This option ***also*** allows you to parse the input in blocks.
- **Parse as blocks:** This checkbox, if checked, will cause the `Block start expression` and `Block end expression` text boxes to appear below.  These text boxes accept Regular Expression strings to allow the Condition to break up the input text into one or more separate "blocks", which can then be parsed individually, in a loop.
  > :information_source: *Only visible if the "Advanced settings toggle" is switched on.*
  - **Advanced block options:** This link will open a modal dialog box, when clicked, and present you with the following options:
    - `Raise violation if ANY block has a violation`: Has two sub-options, allowing you to decide if a violation alert should be raised for *every* block that has a violation, or if all violating blocks should be aggregated into a single violation alert.
    - `Raise violation only if ALL blocks have a violation`: Only one violation will be raised for the entire Rule, and all blocks must have a violation.
- **Operator:** Allows you to choose from a short list of methods for evaluating the input text against your search or evaluation statement:
  - `Contains the string`: A simple sub-string search, which just looks for the presence of your search string within the input text.
  - `Does not contain the string`: Opposite of `Contains the string`; this will be triggered if the sub-string is NOT found in the input text.
  - `Matches the expression`: A Regular Expression pattern which will match the text string you are looking for in the input text.
  - `Does not match the expression`: Opposite of `Matches the expression`; this will be triggered if the Regular Expression pattern does NOT match the input text.
  - `Evaluate expression`: Allows the use of the following comparison operators to create evaluation expressions: `>`, `>=`, `<`, `<=`, `==`, `matches`.
    - Expression statements must be simple string or numerical comparisons, such as `<_IOS_Version> matches 17.15.3` or `<_Exec_Timeout> <= 30`.
- **Actions:** The Actions section allows you to choose what action the Condition statement should take in the event of a match, or no match.  The available options for both "Match" and "Does not match" are the same, and are detailed below:
  - **Action:**
    - `Continue`: Simply continue on to the next Condition statement without taking any action.
    - `Do not raise a violation`: Take the action of *NOT* raising a violation.  Exit this loop iteration and begin the next (if parsing as blocks), or end the conditional check workflow and move on.
    - `Raise a violation`: Take the action of raising a violation alert, then either exit this loop iteration and begin the next (if parsing as blocks), or end the conditional check workflow and move on.
    - `Raise a violation and continue`: Take the action of raising a violation alert and then continue on to the next Condition check in the Rule sequence.
  - **Violation Severity:** Choose the severity level to apply to violations alerts for this Condition statement.  You can select from the following list: `Critical`, `Major`, `Minor`, `Warning`, `Information`
  - **Custom violation message:** This optional field allows you to specify a custom message that should accompany any alerts that are generated by the Condition.  
    - :bulb: **Important Note:** This message *can* reference Variables and/or RegEx match groups from any of the previously run Condition statements (more on how to use RegEx match groups later...).
    - :information_source: Custom violation messages are limited to **100 characters** in length.

  > :exclamation: *Note: The "Match" and "Does not match" Action selection are not allowed to conflict with each other.  In order words, Catalyst Center will not allow you to generate a Violation alert for BOTH scenarios - only one condition outcome can generate an alert.  Likewise, if you attempt to create a new Condition following one which does NOT have a Continue Action, you will receive a warning message indicating that this new Condition is "unreachable" and can not be created.*

[⤴️ ToC](#table-of-contents)

---

## Regular Expressions

Regular Expressions (aka RegEx), as they are known today, are sequences of special (ASCII) characters which are interpreted by the regular expression engine and used to search for, and match, patterns of characters in text strings.  Essentially, they're a special language used to define text "patterns" so that computer programs can recognize matching strings of text as they parse through tons of lines of content.  The concept of Regular Expressions has been around since the early 1950's but they entered popular use in computer programming in 1968, and they have remained a powerful tool ever since.

That said, Regular Expression language can seem archane and can be very difficult to understand, until you have built up enough experience in using it.  Thankfully, there are many online "cheatsheets" and tools available for you to learn on-the-fly:

- https://regexr.com/ (By far the most useful tool online for working with and learning regular expressions)
- https://regex101.com/
- https://regexone.com/
- https://www.rexegg.com/regex-quickstart.php
- https://www.w3schools.com/js/js_regexp.asp
- https://www.w3schools.com/python/python_regex.asp

...and many, many more.

It is also important to understand that there are different implementations of Regular Expression language for different platforms - primarily: POSIX (simple regex) and Perl (advanced regex).  It can be difficult to know what version of Regular Expression language you are working with on a particular system, because that information is difficult to find.  Normally you figure this out by discovering that more advanced RegEx patterns simply don't work, which is an indication that the system is using the older POSIX implementation.  The best solution to this problem is to keep your Regular Expression patterns as simple as possible.

[⤴️ ToC](#table-of-contents)

---

### Key RegEx Concepts

- RegEx patterns are applied to strings of text, and return a Boolean (True/False) value based on whether the pattern was found or not found in the text string.
- A separate text rendering program is needed to "present" the text to the RegEx pattern.  This can be a text editor, a stream reader (`sed`, `awk` or `grep`, for instance), or a programming language capable of reading text (JavaScript or Python, for example).
- Text is normally parsed one line at a time, with the "Newline" (`\n`) or Carriage Return/Newline (`\r\n`) acting as the delimiter between lines.
- RegEx patterns can be configured to capture multiple lines of text and search through them as a block.

RegEx has many, many special characters or character combinations that the engine interprets to build the pattern.  Below is a table of the most commonly used:

| Symbol | Type | Description | Example Pattern | Example Match |
|:-------|:-----|:------------|:----------------|:--------------|
| `^` | Anchor / Boundary | Start of a string; this signifies the very beginning position of the input string. | `^test.*` | `test_string` |
| `$` | Anchor / Boundary | End of a string; signifies the end position of the input string. | `^test_string$` | `test_string` |
| `.` | Character | Any character ***except*** a line break (`\n` or `\r\n`) | `^test.string$` | `test_string`, `test-string` |
| `\` | Character | Escape symbol used to "escape" any special character that should appear in a string | `^test\.string$` | `test.string` | 
| `*` | Quantifier | Matches ***zero or more*** of the ***preceding*** character. | `^test.*` | `test_string`, `test-string`, `test` |
| `+` | Quantifier | Matches ***one or more*** of the ***preceding*** character. | `^test.+` | `test_string`, `test-string`, `tests` |
| `?` | Quantifier | Matches ***zero or one*** of the ***preceding*** character. | `^test.?` | `test_`, `test`, `tests` |
| `{x,y}` | Quantifier | Specifies the minimum number of the ***preceding*** character (`x`), and the maximum number (`y`). | `A{1,3}` | `A`, `AA`, `AAA` |
| `{x}` or `{x,}` | Quantifier | Specifies either *exactly* the number (`{x}`) or at *least* the number (`{x,}`) of the preceding character to match. | `A{2}` or `A{2,}` | `AA` or `AA`, `AAA`, `AAAA`, etc. |
| `\d` | Character | Matches a numerical digit, 0 through 9. (The character `\D` matches any character that is NOT a digit) | `\d{3}` | `666`, `999`, `123`, `456` |
| `\w` | Character | Matches any "word" character, which includes ASCII letters, digits or the underscore. (The `\W` character matches any character that is NOT a "word" character) | `\w.+` | `apple`, `tree`, `Cisco123` |
| `\s` | Character | Matches any whitespace character, which is typically: space, tab, newline, carriage return, vertical tab. (The `\S` character matches any character that is NOT whitespace). Also, `\t` matches a tab, `\r` matches a carriage return, and `\n` matches a newline. | `^\sdescription\s.*` | ` description IP Phone`, ` description ` |
| `[...]` | Class | A set of square brackets (`[]`) containing any number of characters, digits or symbols is a "Character Class". This specifies which characters, digits or symbols are allowed to be matched; in this example, we will specify that any upper or lowercase letter, or any digit, is valid. | `[a-zA-Z0-9]{4}` | `ABCD`, `abcd`, `a1b2`, `1234` |
| `(...)` | Logic | A set of parenthesis (`()`) containing any set of letters, numbers or symbols is a "Capture Group". Capture groups are captured in memory and can be referred to later, much like a variable, using a backslash followed by the numerical order of the captured group. | `^(123)(Cisco)_\2\1` | `123Cisco_Cisco123` |
| `\|` | Logic | The pipe character (`\|`) is the OR operator, which evaluates True if either the value on the left **OR** the value on the right is matched. | `^C(at\|isco\|\+\+)$` | `Cat`, `Cisco`, `C++` |

As you can see, the Regular Expression language is not very intuitive and can be difficult to learn, much less master, without lots of practice.  However, most use cases for Regular Expressions end up being very simple because it's not often that we need to match a massive amount of content contained inside a string of text.

The easiest way to work with Regular Expressions is to study the string you want to match and ask yourself: 

> *"What makes this string unique?"*

> *"How could I pick it out of a large block of text?"*

When we're talking about Cisco device configuration files, or the output of `show` commands, it gets even easier because we already understand the format of the text and we can anticipate the output of a `show` command.  In the next section, we'll work through some examples to help solidify your understanding of Regular Expressions.

[⤴️ ToC](#table-of-contents)

---

### RegEx Examples

Below we will illustrate some practical examples of Regular Expressions to demonstrate their usage:


#### Example 1 
Check interfaces for `authentication control-direction in`

Source Text:
```
interface GigabitEthernet1/0/35
 switchport access vlan 172
 switchport mode access
 authentication control-direction in
 device-tracking attach-policy IPDT_POLICY
!
interface GigabitEthernet1/0/48
 switchport access vlan 172
 switchport mode access
 device-tracking attach-policy IPDT_POLICY
!
```

Conditions:

1. Select interface blocks
    - Scope: `Configuration`
    - Parse as blocks: Checked
    - Block start expression: `^interface (.+)`
    - Block end expression: `^!`
    - Operator: `Matches the expression`
    - Value: `^interface (.+)$`
      > :information_source: *Note: Using the `(.+)` Capture Group expression allows us to capture the name of the interface for use later in a custom violation message.  This is necessary because the default violation message won't provide any indication of which interface it is related to.*
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
2. Match only Access Mode interfaces
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Matches the expression`
    - Value: `^ switchport mode access$`
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
3. Raise violation for "Does not match" interfaces that are missing the 802.1x command
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Matches the expression`
    - Value: `^ authentication control-direction in$`
    - Match action: `Do not raise a violation`
    - Does not match action: `Raise a violation`
    - Violation severity: Select an appropriate severity level
    - Custom violation message: `Interface <1.1> is missing "authentication control-direction in"`
      > :information_source: *Note: Here we are referencing capture group `1.1`, which refers to Condition #1, Capture Group #1.  This will contain the interface's name.  If we had a second capture group defined in Condition #1 it would be referenced by `1.2`, or if there was a capture group defined in Condition #2 it would be referenced by `2.1`, and so on.*

Example Result:

![dot1x_violation_message_test.png](/assets/dot1x_violation_message_test.png)

---

#### Example 2
Check device PKI certificates for upcoming expiration (`show crypto pki certificate`)

Source Text (truncated):
```
Certificate
  Status: Available
  Certificate Serial Number (hex): ABCDEF0123456789
  Certificate Usage: General Purpose
  Issuer:
    cn=sdn-network-infra-ca
  Subject:
    Name: switch.example.com
    cn=C9300-48P_ABCD1234_sdn-network-infra-iwan
    hostname=switch.example.com
  CRL Distribution Points:
    http://example.com
  Validity Date:
    start date: 00:51:07 EST Jun 10 2026
    end   date: 00:51:07 EST Jun 10 2027
    renew date: 00:51:07 EST Mar 29 2027
  Associated Trustpoints: sdn-network-infra-iwan
  Storage: nvram:sdn-network-example1.cer

CA Certificate
  Status: Available
  Certificate Serial Number (hex): ABCDEF0123456789ABCDEF0123456789
  Certificate Usage: Signature
  Issuer:
    cn=sdn-network-infra-ca
  Subject:
    cn=sdn-network-infra-ca
  Validity Date:
    start date: 02:24:53 EST Sep 27 2025
    end   date: 02:24:52 EST Sep 27 2045
  Associated Trustpoints: sdn-network-infra-iwan
  Storage: nvram:sdn-network-example2.cer
```

Conditions:

1. Capture and parse current date from device
    - Scope: `Device command outputs`
    - Show command: `show clock` (You'll need to add this to the custom command list)
    - Operator: `Matches the expression`
    - Value: `^([\d:\.]+)\s(\w+)\s(\w+)\s(\w+)\s(\d+)\s(\d+)$`
      - Example: 
        ```
        switch#show clock
        15:39:32.008 EST Wed Jul 1 2026
        ```
      - Capture Groups:
        ```
        (15:39:32.008) (EST) (Wed) (Jul) (1) (2026)
             1.1        1.2   1.3   1.4  1.5   1.6
        ```
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
2. Capture and parse output of `show crypto pki certificate`
    - Scope: `Device command outputs` (enable "Advanced settings")
    - Show command: `show crypto pki certificate` (You'll need to add this to the custom command list)
    - Parse as blocks: Checked
    - Operator: `Matches the expression`
    - Value: `(.*)`
      > :information_source: *Note: This just captures all of the command output, but isn't used in this Rule.*
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
3. Capture certificate serial number for later reference
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Matches the expression`
    - Value: `^.*[Cc]ertificate Serial Number.*: ([0-9a-fA-F]+)$`
      - Example:
        ```
        Certificate
          Status: Available
          Certificate Serial Number (hex): ABCDEF0123456789
        ```
      - Capture Groups:
        ```
        Certificate Serial Number (hex): (ABCDEF0123456789)
                                                3.1
        ```
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
4. Capture certificate end date
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Matches the expression`
    - Value: `^\s*end\s+date:\s?([0-9:\.]+)\s(\w+)\s(\w+)\s(\d+)\s(\d+)$`
      - Example:
        ```
          Validity Date:
            start date: 00:51:07 EST Jun 10 2026
            end   date: 00:51:07 EST Jun 10 2027
            renew date: 00:51:07 EST Mar 29 2027
        ```
      - Capture Groups:
        ```
        end   date: (00:51:07) (EST) (Jun) (10) (2027)
                       4.1      4.2   4.3   4.4  4.5
        ```
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
4. Compare month and year to `show clock` output
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Evaluate expression`
    - Value: `<4.3> matches <1.4> & <4.5> matches <1.6>`
      - Example:

        *`show clock`*
        ```
        15:39:32.008 EST Wed Jul 1 2026
        ```
        *`show crypto pki certificate`*
        ```
        end   date: 00:51:07 EST Jun 10 2027
        ```
      - Expression:
        ```
        (Jun) matches (Jul) AND (2027) matches (2026)
         4.3           1.4       4.5            1.6
        ```
      - Result: `False`
        > :exclamation: *Note: This expression will evaluate `True` ONLY if the current month and current year match.*
    - Match action: `Raise a violation`
    - Violation severity: Choose an appropriate severity level
    - Custom violation message: `Certificate <3.1> will expire soon! <4.3> <4.4> <4.5>`
    - Does not match action: `Do not raise a violation`

Example Result:
> *Alert output is invalid in this example, only for demonstration purposes.*

![cert_expiration_violation_test.png](/assets/cert_expiration_violation_test.png)

---

#### Example 3 
Check Console & VTY Lines for `exec-timeout` config

Source Text:
```
switch#show run | section line
line con 0
 exec-timeout 60 0
 stopbits 1
line vty 0 4
 exec-timeout 30 0
 authorization exec TACACS_POLICY
 login authentication TACACS_POLICY
 transport input all
line vty 5 15
 authorization exec TACACS_POLICY
 login authentication TACACS_POLICY
 transport input all
line vty 16 31
 transport input all
```

Variables:

- Variable name: `Timeout`
  - Identifier: `_Timeout`
  - Description: `Exec Timeout value in minutes for Console and VTY lines`
  - Data type: `Integer`
  - Input required: Checked
  - Default value: `30`
    > :information_source: *Note: This is optional but it allows you to specify different timeout values for different locations. Alternatively you can hard-code the timeout value in your Conditions or as the variable's default value.*

Conditions:

1. Capture output of line configuration
    - Scope: `Device command outputs` (enable "Advanced Settings")
    - Show command: `show run | section line` (You'll need to add this to the custom command list)
    - Parse as blocks: Checked
    - Block start expression: `^line.+$`
    - Block end expression: N/A
    - Operator: `Matches the expression`
    - Value: `^(line.+)\n(.*)`
      > :information_source: *Note: Using the `(line.+)` capture group allows us to use the Line interface name in a custom violation message.*
    - Match action: `Continue`
    - Does not match action: `Do not raise a violation`
2. Check for presence of `exec-timeout` configuration
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Matches the expression`
    - Value: `^\sexec-timeout\s(\d+)\s(\d*)$`
    - Match action: `Continue`
    - Does not match action: `Raise a violation and continue`
    - Violation severity: Choose an appropriate severity level
    - Custom violation message: `<1.1> is missing an exec-timeout configuration.`
3. Evaluate value of `exec-timeout` minutes
    - Scope: `Previously matched blocks` (enable "Advanced settings")
    - Parse as blocks: Unchecked
    - Operator: `Evaluate expression`
    - Value: `<2.1> >= <_Timeout>`
    - Match action: `Raise a violation`
    - Violation severity: Choose an appropriate severity level
    - Custom violation message: `<1.1> has a timeout of <2.1>m and <2.2>s, which is greater than the desired value: <_Timeout>m`
    - Does not match action: `Do not raise a violation`
      > :information_source: *Note: If condition #2 does not match, a violation will be raised for a missing `exec-timeout` command however, condition #3 will still run.  This will result in a second violation being raised for the timeout value, even though the configuration is not present: `line aux 0 has a timeout of <2.1>m and <2.2>s, which is greater than the desired value: 30m`  This is a quirk caused by condition #2's `Raise a violation and continue` setting, which is required for condition #3 to run if any previous violations are found.*

Example Result:
> *Alert output is invalid in this example, only for demonstration purposes.*

![exec_timeout_value_violation.png](/assets/exec_timeout_value_violation.png)

[⤴️ ToC](#table-of-contents)

---