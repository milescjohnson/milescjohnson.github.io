# Building a Custom Serial Sensor Module with Viam

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

    viam module generate --language python --model-name usb-serial-sensor --name serial_sensor --resource-subtype=sensor

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

Create an abstract base class `SerialSensor`. `SerialSensor` is not a function model on its own,
but organizing our module this way has several advantages:

-   Code reuse: transport logic is the same accross all serial devices, now it lives in one place
-   Extensibility: Adding new serial sensors is trivial. Just create a new subclass of `SerialSensor` and implement `parse_data()` and for that sensor
-   Dependency management: This module uses the `pyserial` for connecting to packages. If you need to update to a new version, you can do that in one place
-   Organization: Having a single module for all `Sensor` sub-types that use a serial connection is clean
-   Flexibility: If you want to support different a different transport protocol in the future, you can do so with minimal changes to sensor logic

File: `src/models/serial_sensor.py`

This class handles:

-   Opening the serial connection
-   Background polling
-   Thread-safe storage of latest readings
-   Lifecycle management
-   Delegation of protocol parsing to subclasses

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

    @classmethod
    def validate_config(
        cls, config: ComponentConfig
    ) -> Tuple[Sequence[str], Sequence[str]]:
        # Check that a path to get an image was configured
        fields = config.attributes.fields
        if "port" not in fields:
            raise Exception("Missing port attribute.")
        elif not fields["port"].HasField("string_value"):
            raise Exception("port must be a string.")
        if "baudrate" not in fields or not fields["baudrate"].HasField("number_value"):
            raise Exception("baudrate must be a number.")

        return [], []

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
                parsed = self.parse_data(self._serial)
                if parsed:
                    with self._lock:
                        self._latest_readings = parsed
            except Exception:
                # In production, log or implement retry logic
                continue

    @abstractmethod
    def parse_data(self, ser: serial.Serial) -> dict:
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

# Gracefully handle `reconfigure`

`reconfigure()` is called on startup and anytime the configuration is changed from the Viam UI.

>   `reconfigure()` is not just a configuration setter. It is the lifecycle boundary between robot configuration and hardware control. It must safely transition the component from one configuration state to another without leaking resources or blocking RPC calls


On reconfigure you must stop running threads and close serial ports to avoid:

-   Multiple polling threads running
-   Serial port collisions
-   OS-level file descriptor accumulation
-   Memory leaks

------------------------------------------------------------------------

# Implement a concrete model for custom hardware

By inheriting from SerialSensor, all that is required from the subclass is 
protocol parsing logic specific to the device. In this example we will be working
with a PMS5003 particulate sensor.

File: `src/models/pms5003_sensor.py`

## PMS5003 Output Data Frame Structure (32 Bytes)
The sensor sends a 32-byte payload of Big-endian unsigned 16-bit integers: 

-   Header: 0x42, 0x4D.
-   Frame Length: 2 bytes (30 bytes after check-code).
-   Data 1-12: Concentration data (PM1.0, PM2.5, PM10 in standard & environmental units).
-   Data 13-18: Particle count (>  per 0.1L air).
-   Reserved: 2 bytes.
-   Checksum: 2 bytes (sum of bytes 1-30).

For this example, we will only be interested in indexes 3-5 (PM1, PM2.5, PM10 in standard units)

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

    def parse_data(self, ser) -> dict:
        if ser.read(1) == b'\x42' and ser.read(1) == b'\x4d':
            payload = ser.read(30)

            if len(payload) != 30:
                return {}

            # and add all bytes of the payload except the last two.
            calc_checksum = 0x42 + 0x4d + sum(payload[:-2])

            values = struct.unpack('>HHHHHHHHHHHHHHH', payload)

            sent_checksum = values[14]
            if calc_checksum != sent_checksum:
                return {}

            return {
                "pm1_0": values[3],
                "pm2_5": values[4],
                "pm10": values[5]
            }

        return {}
```

## Testing Without Hardware

Because transport logic (`SerialSensor`) is separated from protocol parsing logic (`parse_data()`) 
you can test parsing using a simulated serial stream.
Instead of connecting to a real serial.Serial device:

1. Create a mock object that behaves like a serial port
2. Feed it known-good PMS5003 frame bytes
3. Verify that parse_data() returns the expected dictionary

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
    result = sensor.parse_data(fake_serial)

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
2.  Implement `parse_data()`
3.  Register a new `MODEL`
4.  Add entry to `meta.json`

No changes to the transport layer required.

------------------------------------------------------------------------

# Key Takeaways

-   Viam modules cleanly separate hardware logic from application logic
-   Serial transport can be abstracted once and reused
-   Background polling ensures deterministic RPC performance
-   The model system allows clean extensibility

------------------------------------------------------------------------

# Potential Next Steps

## Robust Connection Handling

-   Serial connections are notoriously fickle. If a cable is jiggled or there’s a momentary power dip, the serial port may "ghost" the OS.
-   Add a connection watchdog in your BaseSerialSensor. If parse_data fails X times in a row, the base class should attempt to close the port, wait 5 seconds, and re-initialize the connection automatically.
-   This is ensures your robot doesn't require a manual restart just because a USB cable was loose for a split second.

## Support for "Passive Mode"

Many serial devices have two modes: Active (pushes data constantly) and Passive (waits for a request).Active mode wears out the internal laser and fan faster. Passive mode allows you to sample the sensor less frequently to extend the lifespan of your hardware.

-   Add a `read_mode` attribute to your configuration.
-   In "Passive" mode, your `get_readings` method would send a "Request Data" command to the sensor over serial, wait for the response, and then invoke `parse_data()`.

## Semantic Data Mapping (Viam Tags)

Viam's get_readings returns a dictionary, but different brands use different keys (e.g., pm25 vs pm2_5 vs particulate_matter_2_5). If you swap a PMS5003 for a Honeywell HPMA115S0, your downstream logic (like a dashboard or an air purifier trigger) shouldn't have to change its code because the keys changed.

-   Define a standard schema for the return payload to be used across all models in your module.
-   Ensure every model returns the same keys for the same physical phenomena. You can also include metadata like sensor_model or firmware_version in the dictionary.

## Improve logging and error handling

To really make your module production ready, you should use Viam logging.

    from viam.logging import getLogger

    LOGGER = getLogger(__name__)

Then, when you catch exceptions in your code, you can send logs to the Viam dashboard

    except Exception as e:
        LOGGER.error(f"Error in serial loop: {e}")
