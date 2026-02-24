# Create a Custom USB Air Quality Sensor with Viam Module

This tutorial demonstrates how to build, abstract, and register a custom sensor module for unsupported USB serial devices.
By the end of this guide, you'll have a functional, general module for USB air quality sensors with an example model specifically for the [PMS5003 particulate sensor](https://www.aqmd.gov/docs/default-source/aq-spec/resources-page/plantower-pms5003-manual_v2-3.pdf) that can be registered and queried with a Viam robot runtime.

---

## Prerequisites

* macOS, Linux, or Windows with Python 3.11.
* Viam SDK (Python).
* A USB serial device (or simulated).
* Basic understanding of Python and the Viam platform

---

## Project Structure

Use the Viam CLI to [generate template files for your module](https://docs.viam.com/operate/modules/support-hardware/#generate-stub-files).

`viam module generate`

Name the module `usb-air-sensor` and create two models:
* `serial_sensor.py`
* `pms5003_sensor.py`

Your project should look something like this:

```
usb-air-sensor
└── src
|   ├── models
|   |   └── serial_sensor.py
|   |   └── pms5003_sensor.py
|   └── main.py
└── build.sh
└── meta.json
└── requirements.txt
└── run.sh
└── setup.sh
```

---

## Implement the custom component

To integrate the PMS5003 sensor with the Viam ecosystem, its model needs to implement the `Sensor` component API. In particular, it needs to implement the `GetReadings` function.
That means reading the serial data from the device, parsing and decoding the data, and formatting the response. While parsing and decoding logic is likely to differ between different brands and models
of USB serial devices, the rest of the logic for reading data from the device is largely the same from one device to another. For that reason, we can create an abstract base class `serial_sensor` that will allow 
for code reuse if any additional models get packaged in the same module. 


**serial_sensor.py**

```python
import serial
from viam.components.sensor import Sensor

class SerialSensor(Sensor):
    def __init__(self, port, baudrate=9600):
        self._serial = serial.Serial(port, baudrate, timeout=2)

    def read_frame(self):
        """Read raw data frame from the serial device."""
        raise NotImplementedError("Subclasses must implement read_frame()")

    def parse_frame(self, frame):
        """Convert raw frame into dictionary readings."""
        raise NotImplementedError("Subclasses must implement parse_frame()")

    def get_readings(self):
        frame = self.read_frame()
        return self.parse_frame(frame)
```

**Explanation:**

* Handles serial port opening and timeouts.
* `read_frame()` and `parse_frame()` are implemented in subclasses.
* `get_readings()` provides a consistent interface for Viam.

---

## Step 3 — Implement PMS5003 Sensor Subclass

**pms5003_sensor.py**

```python
import struct
from serial_sensor import SerialSensor

class PMS5003Sensor(SerialSensor):
    def read_frame(self):
        while True:
            if self._serial.read(1) == b'\x42' and self._serial.read(1) == b'\x4d':
                return self._serial.read(30)  # remaining bytes

    def parse_frame(self, frame):
        data = struct.unpack(">HHHHHHHHHHHHHHH", frame[:30])
        return {
            "PM1.0": data[0],
            "PM2.5": data[1],
            "PM10": data[2]
        }
```

* `read_frame` waits for the PMS5003 header and reads the full frame.
* `parse_frame` unpacks the binary frame into PM readings.

---

## Step 4 — Create Module Entry Point

**main.py**

```python
import argparse
from viam.module.module import ModuleBase
from pms5003_sensor import PMS5003Sensor

class PMSModule(ModuleBase):
    def __init__(self, port):
        super().__init__()
        self.pms_sensor = PMS5003Sensor(port=port)
        self.add_component("pms5003_sensor", self.pms_sensor)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--local", action="store_true", help="Run locally")
    parser.add_argument("--socket_path", type=str, required=True, help="Path to robot gRPC socket")
    parser.add_argument("--port", type=str, default="/dev/ttyUSB0", help="Serial port for PMS5003")
    args = parser.parse_args()

    module = PMSModule(port=args.port)
    module.run(args.socket_path)
```

* `--socket_path` points to the local robot gRPC socket.
* `--port` specifies the serial device.
* The module registers `pms5003_sensor` with the robot runtime.

---

## Step 5 — Start Local Viam Robot Runtime

```bash
viam robot run --name my-laptop-robot
```

* Note the gRPC socket path printed (e.g., `/tmp/viam.sock`).

---

## Step 6 — Run the Module

```bash
python3 main.py --local --socket_path /tmp/viam.sock --port /dev/ttyUSB0
```

* The PMS5003 sensor is now registered as a component.
* Open the Viam app to view readings.

---

## Step 7 — Query via Python SDK

```python
from viam.robot.client import RobotClient

client = RobotClient.from_uri("local://my-laptop-robot")
sensor = client.get_sensor("pms5003_sensor")
print(sensor.get_readings())
```

Example output:

```json
{"PM1.0": 12, "PM2.5": 25, "PM10": 40}
```

---

## Benefits of this Architecture

* `SerialSensor` handles generic serial boilerplate.
* Subclasses implement device-specific parsing.
* Easy to add new serial devices with minimal code duplication.
* Clean, maintainable, and extensible for future hardware.

---

**End of Tutorial**
