# sensor-control
This provides the core sensor functionality (other than acoustic and visual) for WildlifeSystems nodes. A default sensing device, `onboard` is provided which provides useful diagnostic information about the Raspberry Pi.

## Installing this package
`wget -O - https://raw.githubusercontent.com/Wildlife-Systems/sensor-control/main/install | sudo bash`

## sr: sensor read
To read a sensor use the sr script `sr <device> <sensor>`, e.g. to read the onboard CPU temperature run:

`sr onboard cpu`

### sr output
`sr` provides a JSON response that describes the sensor and the measurement.

key         | value
------------|------
sensor      | Short name of the sensor (no whitespace)
measures    | The property being measured (e.g. temperature)
value       | The value currently measured by the sensor
unit        | The unit of measurement of value
node_id     | The serial number of the aao node 
sensor_id   | Serial number of sensor (if available)
timestamp   | Timestamp of when the sensor was read
config      | (Optional) Additional configuration details about the sensor. May be JSON array.

## Adding a new sensing device
New sensing devices should be added a seperate scripts/programs that take the prototype JSON response generate by `sc-prototype` and populate the values `sensor`, `measures`, `value`, `unit` and, where appropriate, `config`. The modified JSON should then be printed to `stdout`.

The fields `node_id` and `timestamp` are populated by the sensor read command, `sr`.

The script/program must define one or more sensors on the device, and accept these as the first and only parameter. When `list` is provided as the first parameter the available sensors should be listed, one per line. The `sensor-onboard` script installed with this package provides a reference implementaation in bash using the `jq` command to process JSON.

There are no restrictons on the scripting/programming language(s) that may be used, however it should be kept in mind that the scripts will likely be running on connected, autonomous nodes. For this reason it is reccomeneded that minimising the installation of additional packages, and the nuber of scripting environments overall, should be priorities (there is a reason that `sensor-onboard` is written in bash).

## Development
* Development of this script was done as part of the Urban Nature Project at the Natural History Museum, London.
