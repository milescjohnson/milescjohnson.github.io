# Create a Custom USB Serial Sensor with Viam Module

This tutorial demonstrates how to build, abstract, and register a custom sensor module for unsupported USB serial devices.
By the end of this guide, you'll have a clean and extensible module for a variety of USB, serial, or modbus sensors with an example model specifically for the [PMS5003 particulate sensor](https://www.aqmd.gov/docs/default-source/aq-spec/resources-page/plantower-pms5003-manual_v2-3.pdf) that can be registered and queried with a Viam robot runtime.

---

## Prerequisites

* macOS, Linux, or Windows with Python 3.11.
* A free Viam account
* [Viam CLI](https://docs.viam.com/dev/tools/cli/#install)
* A machine connected to Viam.
* A USB serial device (or simulated).
* Basic familiarity with Python and the Viam platform

---

## Project Structure

Use the Viam CLI to [generate template files for your module](https://docs.viam.com/operate/modules/support-hardware/#generate-stub-files).

`viam module generate --language python --model-name usb-serial-sensor \
  --name serial_sensor --resource-subtype=sensor`

Your project should look something like this:

```
usb-serial-sensor
└── src
|   ├── models
|   |   └── serial_sensor.py
|   └── main.py
└── build.sh
└── meta.json
└── requirements.txt
└── run.sh
└── setup.sh
```

---

## Implement the custom component

Custom components need to implement one of Viam's standardized component APIs. In this case, you'll need to implement the [Sensor API](https://docs.viam.com/dev/reference/apis/components/sensor/). Most notably, you will need to add logic to `GetReadings` to read serialized data from the device, parse that data, and format the response. Parsing and formatting logic is specific and variable from one device to another, but a lot of boilerplate code for dealing with USB serials can be generalized by making `SerialSensor` an abstract base class. SerialSensor will not be registered but will make implementing different models and managing their dependencies much cleaner and easier.

The `SerialSensor` class handles the highlevel process of initializing serial port access and managing configs. It also spins up a background poller that repeatedly polls the serial connection for data ensuring get_readings is non-blocking with deterministic latency. This means that any models for specific hardware only need to worry about implementing `parse_logic`

**serial_sensor.py**

```python
import serial
import threading
from abc import ABC, abstractmethod
from viam.components.sensor import Sensor

class SerialSensor(Sensor):
    """
    Handles the boilerplate: connection, background threading, 
    and thread-safe data storage.
    """

    @classmethod
    def new(cls, config: Any, dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
        sensor = cls(config.name)
        sensor.reconfigure(config, dependencies)
        return sensor

    def reconfigure(self, config: Any, dependencies: Mapping[ResourceName, ResourceBase]):
        # 1. Stop existing serial thread if it's running
        self.stop_thread()

        # 2. Get config from Viam UI (e.g., {"port": "/dev/ttyUSB0"})
        port = config.attributes.fields["port"].string_value or "/dev/ttyUSB0"
        
        # 3. Initialize new serial connection
        import serial
        self.serial = serial.Serial(port, baudrate=9600, timeout=1)
        self._latest_readings = {}
        
        # 4. Restart the background poller
        self._running = True
        self._thread = threading.Thread(target=self._run_loop, daemon=True)
        self._thread.start()

    def parse_logic(self, ser_instance: serial.Serial) -> dict:
        """
        Subclasses implement this to find start bytes, 
        unpack the stream, and return a dict.
        """
        pass

    def _run_loop(self):
        while self._running:
            try:
                # Subclass handles the protocol specific 'read'
                new_data = self.parse_logic(self.serial)
                if new_data:
                    with self._lock:
                        self._latest_readings = new_data
            except Exception as e:
                # Log error or handle reconnect logic here
                continue

    async def get_readings(self, **kwargs) -> dict:
        with self._lock:
            return self._latest_readings

    def close(self):
        self._running = False
        self._thread.join()
        self.serial.close()
```

Now you can create a functional model. Create a file `/src/models/pms5003_sensor.py`. This will implement the logic for parsing the data from a [PMS5003 Air Quality Monitor](https://www.aqmd.gov/docs/default-source/aq-spec/resources-page/plantower-pms5003-manual_v2-3.pdf)

The `PMS5003` outputs air quality data in 32-byte binary data frames. The parser just needs to look for  `x42` and `x4d` denoting the beginning of a frame and unpack the following 30 bytes.  


**pms5003_sensor.py**

```python
import struct
from typing import Self, Mapping, Any
from viam.proto.common import ResourceName
from viam.resource.base import ResourceBase
from viam.resource.types import Model, ModelFamily
from .serial_sensor import SerialSensor

class PMS5003(SerialSensor):
    # This identifies your specific sensor model in the Viam Registry
    MODEL = Model(ModelFamily("my-org", "air-quality"), "pms5003")

    @classmethod
    def new(cls, config: Any, dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
        sensor = cls(config.name)
        sensor.reconfigure(config, dependencies)
        return sensor

    def parse_logic(self, ser) -> dict:
        # Implementation of the 32-byte PMS5003 protocol
        if ser.read(1) == b'\x42' and ser.read(1) == b'\x4d':
            payload = ser.read(30)
            res = struct.unpack('>HHHHHHHHHHHHHHH', payload)
            return {"pm2_5": res[6], "pm10": res[7]}
        return {}
```

---
## Update meta.json and requirements.txt


**requirements.txt**
```
viam-sdk
pyserial
```

Add python package dependencies to requirements.txt


**meta.json**

```
{
  "module_id": "your-org-name:air-quality-sensors",
  "visibility": "private",
  "models": [
    {
      "api": "rdk:component:sensor",
      "model": "your-org-name:air-quality-sensors:pms5003"
    }
  ],
  "entrypoint": "python3 src/main.py"
}
```

List all models the module is capable of handling. Note: Do not include SerialSensor in here, it is not a model.

---

