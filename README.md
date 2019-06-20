# AutoAPI L11


The message types are now unified into 2 types.

* Get commands use `0x00`.  
* Set commands use `0x01`.  

Get commands allow a *state* to be queried (requested) from a connected device.  

Set commands allow *setting* of values, but also are used to transfer the *state* to the other device.  
It should be thought of as one device setting the data in the other one. What the receiving device then does with this data is up to the client / vehicle to decide.

The `.yml` spec files defines the following *values* and *syntax* for each capability.  


### `getters`

If not defined in a capability – there are *no getters* for that capability.  
Otherwise, it's a dictionary (hash) with 2 possible keys.

* `default` marks the capability having *default getters*
* `static` defines a getter that requests *specific properties*

#### `default`

Requires 2 getters to be automatically synthesised.

```
get_[name]_state()                  --> [id.msb, id.lsb, 0x00]
get_[name]_properties(property_IDs) --> [id.msb, id.lsb, 0x00] + property_IDs
```

The `_state` getter takes no input and requests **all** properties in the capability.  
The `_properties` getter takes in *property IDs* as arguments and requests **only** the specific properties.  

There are additional customisation options available for getters:

* `name_override: string` used to override the `[name]` in the getters generation
* `omit_state_text: bool` used to remove the `_state` text from the *state-getter*
* `skip_properties_getter: bool` used to skip the `_properties` getter generation
	* If `false` (default), the `_properties` getter is **only** generated when there are **more than 1** property in the capability

Examples:

```
getters:
    default

getters:
    default:
        name_override: vehicle_location
        omit_state_text: true

getters:
    default:
        omit_state_text: true
        skip_properties_getter: true
```

#### `static`

Defines *static* getters, that always request *specific properties*, to be automatically synthesised.  

```
get_[name]()                        --> [id.msb, id.lsb, 0x00] + property_IDs
```

Contains an array of *static* getters that have the following keys-values:

* `name: string` as the name of the getter
* `properties: [property_IDs]` defines what properties the getter requests

Examples:  

```
getters:
    static:
      - name: get_control_mode
        properties: [0x01]
      - name: some_other_getter
        properties: [0x02, 0x03]
```


### `commands`

If not defined in a capability – there are *no commands* for that capability.  
Otherwise, it's an array of dictionaries (hashes) as elements, with the dictionary having the following keys.  

The keys are divided into 2 categories: *required* and *optional* ones.  

Required keys:  

* `name: string` as the name of the setter

Optional keys (at least 1 has to be included; can be combined):  

* `mandatory: [property_IDs]` defines what properties are required as input (and what are actually sent in the command)
* `optional: [property_IDs]` list of *optional* properties allowed as input, if only this is defined (no *mandatory* or *constants*) - **at least 1** input is required
* `constants` is an array of constant values defined by the following keys:
    * `property_id: property_ID` defines the constant property
    * `value: [bytes]` lists the constant value of the property

Examples:

```
commands:
  - name: start_stop_ionising
    mandatory: [0x08]
  - name: set_temperature_settings
    optional: [0x03, 0x04, 0x0c]

commands:
  - name: control_command
    optional: [0x02, 0x03]
  - name: start_control
    constants:
      - property_id: 0x01
        value: [0x02]
  - name: stop_control
    constants:
      - property_id: 0x01
        value: [0x05]
```


### `state`

If not defined in a capability – there are *no states* for that capability.  
Defines what *properties* are exposed to the developer (client). This message is sent over the `set` message type.  

```
[id.msb, id.lsb, 0x01] + state_properties
```

Defines an array of *property IDs* (or keys).

* `all` noting that **all** properties in a state are exposed / included
* `[property_IDs]` defines what properties are exposed

Examples:  

```
state: all

state: [0x01, 0x02]
```


### `properties`

**Required** for every capability.  
Follows the same syntax as before, with some additional options for *enums*.  

* `id: integer` identifier <sup>required (if a *property* or an *enum value*)</sup>
* `name: string` name of the property / type <sup>required</sup>
* `type: string` type of the property / item <sup>required</sup>
* `size: integer` size of the item <sup>required (if the type isn't *string* or * * doesn't have *dynamic* values)</sup>
* `[flags]` define additional options for the property (i.e. `multiple: true`) <sup>optional</sup>
* `[texts]` means other *text* related things (i.e. `pretty_name: string`, * `description: string`) <sup>optional</sup>
* `values/items` array of values / items that define the data structure in the property (same as previous versions) <sup>required</sup>

*New* keys for `enum` types.  

* `command_allowed_values: [enum_values]` defines what values are allowed to use in a *command* (that uses that property)
* `verb: string` used to express an action in a *command* or used as another name (i.e. *active - activate*, *triggered - trigger* )
* `verb_pretty_name: string` defines the "pretty" name of the *verb*

Examples:  

```
properties:
  - id: 0x03
    name: convertible_roof_state
    type: enum
    size: 1
    command_allowed_values: [0x00, 0x01]
    values:
      - id: 0x00
        name: closed
        verb: close
      - id: 0x01
        name: open
      - id: 0x02
        name: emergency_locked
      - id: 0x03
        name: closed_secured
  - id: 0x04
    name: sunroof_tilt_state
    type: enum
    size: 1
    values:
      - id: 0x00
        name: closed
        verb: close
      - id: 0x01
        name: tilted
      - id: 0x02
        name: half_tilted

properties:
  - id: 0x1c
    name: wheel_rpms
    type: custom
    size: 3
    multiple: true
    pretty_name: Wheel RPMs
    items:
      - name: location
        type: enum
        size: 1
        values:
          - id: 0x00
            name: front_left
          - id: 0x01
            name: front_right
          - id: 0x02
            name: rear_right
          - id: 0x03
            name: rear_left
      - name: rpm
        type: integer
        size: 2
        pretty_name: RPM
        description: The RPM measured at this wheel

properties:
  - id: 0x12
    name: price_tariffs
    type: custom
    multiple: true
    items:
      - name: pricing_type
        type: enum
        size: 1
        values:
          - id: 0x00
            name: starting_fee
          - id: 0x01
            name: per_minute
          - id: 0x02
            name: per_kwh
            pretty_name: Per kWh
      - name: price
        type: float
        size: 4
        description: The price in 4-bytes per IEEE 754
      - name: currency_size
        type: integer
        size: 1
        description: Size of the currency string
      - name: currency
        type: string
        description: The currency alphabetic code per ISO 4217 or crypto currency symbol
```


### `miscellaneous`

New *types* for properties.  

* `datetime` is used in all our *timestamp* / *date* related properties / types (used to be `integer: 8`)
* `signal` used to convey a single action that doesn't have any value (empty *property data component*; i.e. *wake_up*, *clear_notification*)
* `custom` used when the property contains `items` (meaning it's a custom structure defined by us)
* `capability_state` represents a state of a capability (i.e. *states* in *vehicle status*)
