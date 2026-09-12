# sensor-control

This provides the core sensor functionality (other than acoustic and visual) for WildlifeSystems nodes. A default sensing device, `onboard` is provided which provides useful diagnostic information about the Raspberry Pi.

## Installing the software

[Add the WildlifeSystems APT repository to your system](https://wildlife.systems/apt-configuration.html)

Install sensor-control.

```bash
sudo apt update
sudo apt install sensor-control
```


## sr: sensor read

The `sr` command reads sensors and outputs their data as JSON. It runs multiple sensors in parallel for efficiency.

To read a specific sensor use `sr <device> <sensor>`, e.g. to read the onboard CPU temperature run:

```bash
sr onboard onboard_cpu
```

The second argument is passed to the device's driver as given, so it can be anything the driver accepts. To see what a driver measures:

```bash
sr dht11 list
```

To read all sensors from a device (the default when no sensor is named):

```bash
sr onboard all
```

To select readings across devices, filter the output instead: `sr all --sensor dht11_temperature` keeps readings whose `sensor` field matches, `--sensor_id` does the same for `sensor_id`, and `--with-errors` keeps only failed readings.

To list all installed sensor devices:

```bash
sr list
```

### Checking expected sensors

`sr check` verifies that a node is reporting all of the sensors it is supposed
to. It reads `/etc/ws/expected.sensors`, which lists the `sensor_id` of each
expected sensor, one per line (blank lines and `#` comments are ignored):

```
{{serial}}_onboard_cpu
{{serial}}_storage_used
28-0000075a1b2c
```

`{{serial}}` is replaced with this node's serial number (the output of
`pi-data serial`), so the same list can be deployed to every node. Sensor IDs
that do not embed the node serial, such as a 1-Wire address, are written out in
full.

```bash
sr check
```

It exits `0` when the file is present and every listed `sensor_id` appears in
the output of `sr all`. If sensors are missing they are listed on stderr and
`sr check` exits `23`; if the file is missing, unreadable or empty it exits
`22`. This makes it suitable for cron or monitoring checks:

```bash
sr check || logger -t sensor-control "expected sensors missing"
```

The file location can be overridden with the `EXPECTED_SENSORS_FILE`
environment variable.

### Deployment identity

If `/etc/ws/node.json` exists and carries a `deployment_id`, `sr` copies it into
every reading alongside `node_id` and `timestamp`:

```json
{ "deployment_id": "unp-pond-01" }
```

The file is shipped and documented by `ws-node`; `sr` only reads it. It is
optional — without it, or without that key, the `deployment_id` field stays
null, which is what it has always been. A malformed file is ignored rather than
treated as an error, so a bad edit cannot stop sensors being read.

The location can be overridden with the `NODE_JSON_FILE` environment variable.

### Concurrency Control

By default, `sr` runs sensors in parallel, up to one per CPU core, or four if the number of cores cannot be determined. The limit is set with the `CONCURRENCY` environment variable:

```bash
# Run up to 8 sensors concurrently
CONCURRENCY=8 sr all

# Run sensors sequentially (one at a time)
CONCURRENCY=1 sr all
```

### sr output

`sr` outputs an array of JSON objects, one per reading. Each has the fields of the `sc-prototype` template.

| key           | value |
|---------------|-------|
| sensor        | The name of the sensor, without whitespace (e.g. `dht11_temperature`, `ds18b20`, `onboard_cpu`) |
| device        | The physical device model (e.g. `dht11`, `raspberry_pi`), or null for a pseudo-sensor such as `storage_used` |
| measures      | The property measured (e.g. `temperature`) |
| value         | The reading, or null if the reading failed |
| unit          | The unit of measurement (e.g. `Celsius`, `percentage`, `hPa`, `Ohms`) |
| node_id       | The serial number of the node, filled in by `sr` |
| sensor_id     | The identifier of the sensor, or null if it could not be determined |
| sensor_name   | A human-readable name from the driver's configuration, or null |
| location      | Where the sensor is: a GeoJSON Point or Feature, the token `{{node}}` or `{{none}}`, or null if no location was declared |
| deployment_id | The deployment identifier from `/etc/ws/node.json`, or null; filled in by `sr` |
| timestamp     | The Unix time at which the sensor was read |
| config        | Driver-specific details of the sensor, as a JSON object, or null |
| internal      | Whether the sensor is inside the enclosure |
| error         | The reason the reading failed, or null. A reading never has both a value and an error |

### Return Codes

`sr` uses standard return codes:

| Code | Meaning |
|------|---------|
| 0    | Success |
| 1    | Missing required commands or system error |
| 2    | Invalid arguments or sensor name |
| 20   | Unknown device |
| 21   | Unknown sensor or sensor not found |
| 22   | (`sr check`) Expected sensors file missing, unreadable, or empty |
| 23   | (`sr check`) One or more expected sensors are not reporting |

## Installing a new sensing device

Each sensing device is its own Debian package, `sensor-<device>`, from the WildlifeSystems APT repository; install it with `apt` and `sr list` will find it.

## Adding a new sensing device

A new sensing device is a separate program, `sensor-<device>`, that fills in the reading template generated by `sc-prototype` and prints a JSON array of readings to `stdout`. The `node_id` and `deployment_id` fields, and `timestamp` if the driver has not set it, are filled in by `sr`.

A driver written in C should use `libwildlifesystems`, which provides the template handling, configuration parsing and standard commands; `sensor-dht11`, `sensor-bme680` and `sensor-w1therm` are the reference implementations. A driver written in shell should describe its readings to `ws-emit`, from the `wildlifesystems-tools` package, which produces the same JSON using the same library code; `sensor-onboard`, installed with this package, is the reference implementation.

### Sensor Script Requirements

1. **Executable**: Must be executable and named `sensor-<devicename>` in `/usr/bin/`
2. **Identify command**: Must exit with code 60 when called with `identify` argument
3. **List command**: Must list available sensors when called with `list` argument
4. **JSON output**: Must output valid JSON array to stdout
5. **Exit codes**: 60 for `identify`, 20 for an argument the driver does not accept, 0 otherwise; there is no 21, which is `sr`'s own code for a driver that does not exist
6. **Mock**: Must emit fixed readings in the real output format when called with `mock`, so `sr --mock` and `sr check --mock` can exercise the node without hardware
7. **Timeout**: Scripts must complete within 10 seconds (enforced by `sr`, which sends SIGTERM at 10 seconds and SIGKILL 5 seconds later); a script killed by the timeout contributes nothing, so retries of a failing sensor must fit inside it

There are no restrictions on the scripting/programming language(s) that may be used, however it should be kept in mind that the scripts will likely be running on connected, autonomous nodes. For this reason it is recommended that minimizing the installation of additional packages, and the number of scripting environments overall, should be priorities (there is a reason that `sensor-onboard` is written in bash).

## Development

- Development of this script was done as part of the Urban Nature Project at the Natural History Museum, London.
