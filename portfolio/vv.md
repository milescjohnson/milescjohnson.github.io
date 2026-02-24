# Building a Production-Ready Custom Serial Sensor Module with Viam

## Overview

In this tutorial, you will build a reusable, production-quality custom
Viam module for easy integration of unsupported serial sensor components.
The module:

-   Implements a generic `SerialSensor` base class
-   Cleanly separates transport logic from protocol parsing
-   Runs a background polling loop for deterministic `get_readings()`
    latency
-   Implements a concrete `PMS5003` particulate matter sensor subclass
-   Registers and runs locally within the Viam runtime
-   Exposes live sensor data in the Viam app via the standard Sensor API

------------------------------------------------------------------------

## Prerequisites

-   Python 3.11
-   Viam CLI installed
-   A Viam account
-   A machine registered in Viam
-   Optional: PMS5003 particulate sensor

------------------------------------------------------------------------

# Project Setup

Generate the module scaffold:

    viam module generate --language python --model-name usb-serial-sensor       --name serial_sensor --resource-subtype=sensor

Your project structure:

    usb-serial-sensor
    ├── src
    │   ├── models
    │   │   ├── serial_sensor.py
    │   │   └── pms5003_sensor.py
    │   └── main.py
    ├── build.sh
    ├── meta.json
    ├── requirements.txt
    ├── run.sh
    └── setup.sh

Create a virtual environment:

    python3 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip

Add dependencies in `requirements.txt`:

    viam-sdk
    pyserial

Install:

    pip install -r requirements.txt

------------------------------------------------------------------------

# Implement the Generic SerialSensor Base Class

File: `src/models/serial_sensor.py`

This class handles:

-   Opening the serial connection
-   Background polling
-   Thread-safe storage of latest readings
-   Lifecycle management
-   Delegation of protocol parsing to subclasses

This structure makes it trivial to support additional serial devices by
subclassing `SerialSensor` and implementing new `parse_logic()` methods.

``` python
import threading
import serial
from abc import ABC, abstractmethod
from typing import Mapping, Any, Self

from viam.components.sensor import Sensor
from viam.proto.common import ResourceName
from viam.resource.base import ResourceBase

class SerialSensor(Sensor, ABC):

    @classmethod
    def new(cls, config: Any, dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
        sensor = cls(config.name)
        sensor.reconfigure(config, dependencies)
        return sensor

    def reconfigure(self, config: Any, dependencies: Mapping[ResourceName, ResourceBase]):
        # Stop existing thread if reconfiguring
        self._running = False
        if hasattr(self, "_thread"):
            self._thread.join()

        # Extract configuration
        port = config.attributes.fields["port"].string_value
        baudrate_field = config.attributes.fields.get("baudrate")
        baudrate = baudrate_field.number_value if baudrate_field else 9600

        # Open serial connection
        self._serial = serial.Serial(port, baudrate=baudrate, timeout=1)

        # Thread-safe storage
        self._lock = threading.Lock()
        self._latest_readings = {}

        # Start polling loop
        self._running = True
        self._thread = threading.Thread(target=self._poll_loop, daemon=True)
        self._thread.start()

    def _poll_loop(self):
        while self._running:
            try:
                parsed = self.parse_logic(self._serial)
                if parsed:
                    with self._lock:
                        self._latest_readings = parsed
            except Exception:
                # In production, log or implement retry logic
                continue

    @abstractmethod
    def parse_logic(self, ser: serial.Serial) -> dict:
        pass

    async def get_readings(self, **kwargs) -> dict:
        with self._lock:
            return self._latest_readings

    def close(self):
        self._running = False
        if hasattr(self, "_thread"):
            self._thread.join()
        if hasattr(self, "_serial"):
            self._serial.close()
```

------------------------------------------------------------------------

# Why Use a Background Thread?

If `get_readings()` performed blocking serial reads:

-   RPC latency would depend on hardware response time
-   Calls could block indefinitely
-   Multiple callers could cause contention

Instead:

-   The thread continuously polls hardware
-   Latest data is cached
-   `get_readings()` returns immediately

------------------------------------------------------------------------

# Implement a concrete model for custom hardware

By inheriting from SerialSensor, all that is required from the subclass is 
protocol parsing logic specific to the device. In this example we will be working
with a PMS5003 particulate sensor.

File: `src/models/pms5003_sensor.py`

The PMS5003 outputs 32-byte frames:

-   Start bytes: 0x42 0x4D
-   30-byte payload
-   Big-endian unsigned 16-bit integers

``` python
import struct
from typing import Mapping, Any, Self

from viam.proto.common import ResourceName
from viam.resource.base import ResourceBase
from viam.resource.types import Model, ModelFamily

from .serial_sensor import SerialSensor

class PMS5003(SerialSensor):

    MODEL = Model(ModelFamily("your-org-name", "usb-sensors"), "pms5003")

    @classmethod
    def new(cls, config: Any, dependencies: Mapping[ResourceName, ResourceBase]) -> Self:
        sensor = cls(config.name)
        sensor.reconfigure(config, dependencies)
        return sensor

    def parse_logic(self, ser) -> dict:
        if ser.read(1) == b'\x42' and ser.read(1) == b'\x4d':
            payload = ser.read(30)

            if len(payload) != 30:
                return {}

            values = struct.unpack('>HHHHHHHHHHHHHHH', payload)

            return {
                "pm1_0": values[3],
                "pm2_5": values[4],
                "pm10": values[5]
            }

        return {}
```

## Testing Without Hardware

Because transport logic (`SerialSensor`) is separated from protocol parsing logic (`parse_logic()`) 
you can test parsing using a simulated serial stream.
Instead of connecting to a real serial.Serial device:

1. Create a mock object that behaves like a serial port
2. Feed it known-good PMS5003 frame bytes
3. Verify that parse_logic() returns the expected dictionary

This keeps tests deterministic, fast, hardware-independent, and CI-friendly

You can simulate serial.Serial with a minimal class:

```python
class FakeSerial:
    def __init__(self, data: bytes):
        self._buffer = data

    def read(self, n: int) -> bytes:
        chunk = self._buffer[:n]
        self._buffer = self._buffer[n:]
        return chunk
```

And then, using the PMS5003 output format outlined above, create a simple unit test

```python
from models.pms5003_sensor import PMS5003
import struct

def test_pms5003_parsing():
    # Create known test values
    values = [0] * 15
    values[3] = 12
    values[4] = 25
    values[5] = 40

    payload = struct.pack('>HHHHHHHHHHHHHHH', *values)
    frame = b'\x42\x4d' + payload

    fake_serial = FakeSerial(frame)

    sensor = PMS5003("test")
    result = sensor.parse_logic(fake_serial)

    assert result["pm1_0"] == 12
    assert result["pm2_5"] == 25
    assert result["pm10"] == 40
```

------------------------------------------------------------------------

# Register the Model

File: `src/main.py`

``` python
from viam.module.module import Module
from viam.components.sensor import Sensor

from models.pms5003_sensor import PMS5003

async def main():
    module = Module.from_args()

    module.add_model_from_registry(Sensor.API, PMS5003.MODEL, PMS5003)

    await module.start()

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

------------------------------------------------------------------------

# Update meta.json

``` json
{
  "module_id": "your-org-name:usb-sensors",
  "visibility": "private",
  "models": [
    {
      "api": "rdk:component:sensor",
      "model": "your-org-name:usb-sensors:pms5003"
    }
  ],
  "entrypoint": "python3 src/main.py"
}
```

------------------------------------------------------------------------

# Run Locally

    source venv/bin/activate
    python3 src/main.py --local

This starts the module and waits for Viam to connect via socket.

------------------------------------------------------------------------

# Add the Sensor in the Viam App

In the Viam app:

1.  Open your machine
2.  Add component → Sensor
3.  Choose model: your-org-name:usb-sensors:pms5003
4.  Add attributes:

```
    {
      "port": "/dev/ttyUSB0",
      "baudrate": 9600
    }
```

Save and refresh.

You should now see live particulate readings.

------------------------------------------------------------------------

# Extending to Other Serial Devices

To support a new device:

1.  Subclass `SerialSensor`
2.  Implement `parse_logic()`
3.  Register a new `MODEL`
4.  Add entry to `meta.json`

No changes to the transport layer required.

------------------------------------------------------------------------

# Key Takeaways

-   Viam modules cleanly separate hardware logic from application logic
-   Serial transport can be abstracted once and reused
-   Background polling ensures deterministic RPC performance
-   The model system allows clean extensibility
