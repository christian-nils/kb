# Puck.JS

## Set up SD Card

### Physical connections

Assuming that you are using a SD card reader breakout board, you need to connect
the `GND` and `3.3V` to your power supply `3.3V` and `GND`. If you are not using a power
supply, your will need to connect it to the Puck.js pins `3V` and `GND`.

You can then connect the `MISO`, `MOSI`, `SCK`, and `CS` pins to any of the pins of the
Puck.js (for instance `D28`, `D29`, `D30`, `D31`).

### Software configuration

Using the WebIDE, you can use the following code to mount the SD card storage
(note that you will need to have the "FILESYSTEM" module enable in your
firmware, see next section).

```javascript
// Wire up up MOSI, MISO, SCK and CS pins (along with 3.3v and GND)
SPI1.setup({ mosi: D30, miso: D28, sck: D29, baud: 8000000 });
// console.log(E);
E.connectSDCard(SPI1, D31 /*CS*/);
// see what's on the device
try {
  console.log(require("fs").readdirSync());
} catch (error) {
  console.log(error);
}
```

Here, we are using `SPI1` which corresponds to the Hardware SPI. If you want to
use the Software SPI (i.e. bit-banging techniques), you can create a SPI object
via `var spi = new SPI();`. Note that it will be way slower than the Hardware
connection.

## Compile Espruino for Puck.JS

First you need to GIT clone the official repository.

```bash
git clone https://github.com/espruino/Espruino.git
```

When the repository is cloned, you can open the project in VS Code for instance
and do your changes. For instance, here we will add the `filesystem` module and
remove the `neopixel` and `graphics` modules. Open `boards/PUCKJS.py` and modify the file so
that the following is set.

```python
info = {
 'name' : "Puck.js",
 'link' :  [ "http://www.espruino.com/PuckJS" ],
 'default_console' : "EV_SERIAL1",
 'default_console_tx' : "D28",
 'default_console_rx' : "D29",
 'default_console_baudrate' : "9600",
 'variables' : 2250, # How many variables are allocated for Espruino to use. RAM will be overflowed if this number is too high and code won't compile.
 'bootloader' : 1,
 'binary_name' : 'espruino_%v_puckjs.hex',
 'build' : {
   'optimizeflags' : '-Os',
   'libraries' : [
     'BLUETOOTH',
     'NET',
     #'GRAPHICS',
     'CRYPTO','SHA256','SHA512',
     'AES',
     'NFC',
     #'NEOPIXEL',
     'FILESYSTEM',
     #'TLS'
   ],
   'makefile' : [
     'DEFINES+=-DHAL_NFC_ENGINEERING_BC_FTPAN_WORKAROUND=1', # Looks like proper production nRF52s had this issue
     # 'DEFINES+=-DCONFIG_GPIO_AS_PINRESET', # reset isn't being used, so let's just have an extra IO (needed for Puck.js V2)
     'DEFINES+=-DESPR_DCDC_ENABLE', # Ensure DCDC converter is enabled
     'DEFINES+=-DBLUETOOTH_NAME_PREFIX=\'"Puck.js"\'',
     'DEFINES+=-DCUSTOM_GETBATTERY=jswrap_puck_getBattery',
     'DEFINES+=-DNFC_DEFAULT_URL=\'"https://puck-js.com/go"\'',
     'DFU_PRIVATE_KEY=targets/nrf5x_dfu/dfu_private_key.pem',
     'DFU_SETTINGS=--application-version 0xff --hw-version 52 --sd-req 0x8C',
     'INCLUDE += -I$(ROOT)/libs/puckjs',
     'WRAPPERSOURCES += libs/puckjs/jswrap_puck.c'
   ]
 }
};
```

When you are satisfied with the changes, you can simply
follow the [official
instructions](https://github.com/espruino/Espruino/blob/master/README_Building.md#under-linux).
Which would then be:

```bash
cd Espruino
source scripts/provision.sh PUCKJS
make clean && DFU_UPDATE_BUILD=1 BOARD=PUCKJS RELEASE=1 make
```

**Note**: At the time of writing this page, Python 3.9 was not supported by NRFUtil
(>v6) which is required for the compilation. So make sure to have a version
below 3.9 (e.g. 3.7 should work). Also, install `nrfutil` manually without
`sudo` if you are using a Conda environment with Python 3.x. If you install
NRFUtil manually, do not forget to source `provision.sh` afterwards.

You will find a `*puckjs*.zip` file that you will use to flash your Puck.JS
board using the [official instructions](http://www.espruino.com/Puck.js#firmware-updates).
