# gen_gmsl_dts tool

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [JSON Configuration Parameters](#json-configuration-parameters)
- [Example Configuration](#example-configuration)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## Overview

The `gen_gmsl_dts` tool automates the generation of Device Tree Source (DTS) overlays tailored for Gigabit Multimedia Serial Link (GMSL) camera configurations on NVIDIA Jetson Tegra194 platforms. It simplifies the integration of GMSL serializers, deserializers, and camera modules by translating JSON configuration files into corresponding DTS overlays.

## Prerequisites

- NVIDIA Jetson Tegra194-based board (e.g., AD-GMSL-522) running a compatible Linux kernel.
- Python 3.x with the Jinja2 package installed.
- Device Tree Compiler (`dtc`) and `fdtoverlay` installed on the host.

## JSON Configuration Parameters

The JSON configuration file defines the GMSL setup. Below are the primary parameters:

- **`name`**: Specifies the deserializer model. Can be:
  - `"max96724"`

- **`i2c_bus`**: The I2C bus label the deserializer is connected to:
  - `"gen2_i2c"`: General I2C bus on Tegra194 (`/i2c@c240000`)

- **`i2c_clock_frequency`** *(optional)*: I2C bus clock frequency in Hz (e.g., `400000`).

- **`pool_addrs`**: I2C alias pool addresses for the deserializer (e.g., `["0x41", "0x42", "0x43", "0x44"]`).

- **`platform_cfg`**: Platform-specific configurations:
  - `name`: Platform template (e.g., `"ad-gmsl522"`)
  - `tegra_port`: NVCSI port index (e.g., `4`)
  - `tegra_sinterface`: Tegra serial interface name (e.g., `"serial_e"`)
  - `lane_polarity` *(optional)*: CSI lane polarity swap value (e.g., `"6"`)

- **`phys`**: List of MIPI PHY configurations:
  - `phy_idx`: Deserializer PHY index (e.g., `2`)
  - `num_lanes`: Number of MIPI CSI-2 data lanes (e.g., `4`)
  - `link_frequencies`: MIPI PHY lane rate. Because of double data rate on MIPI, the rate must be half of the lane rate (e.g., `[750000000]` for 1.5 Gbps lane rate)
  - `clock_lanes`: Clock lane indices (e.g., `[0]`)
  - `data_lanes`: Data lane indices (e.g., `[1, 2, 3, 4]` for 4 lanes)
  - `pipe_stream_autoselect` *(optional)*: Enable automatic pipe-stream selection on the PHY (default: `true`)

- **`links`**: List of GMSL links. One entry for every connected serializer:
  - `name`: Specifies the serializer model. Can be:
    - `"max96717"`
  - `cameras`: Camera connected to the serializer:
    - `name`: Camera model, can be `"imx219"`, `"imx477"`, or `"ox03a"`
    - `reset_pin` *(optional)*: GPIO pin for camera reset (IMX219 default: `0`, IMX477 default: `2`)
    - `pwdn_pin` *(optional)*: GPIO pin for camera power down (OX03A, default: `0`)
    - `rclk_pin` *(optional)*: GPIO pin for reference clock output (IMX477/OX03A, default: `4`)
    - `ad_gain` *(optional)*: Use analog+digital gain control instead of simple gain (OX03A only, default: `false`)
  - `pool_addrs`: I2C alias pool addresses for the serializer (e.g., `["0x50", "0x51"]`)

## Example Configuration

```json
[
    {
        "name": "max96724",
        "i2c_bus": "gen2_i2c",
        "i2c_clock_frequency": 400000,
        "pool_addrs": ["0x41", "0x42", "0x43", "0x44"],
        "platform_cfg": {
            "name": "ad-gmsl522",
            "tegra_port": 4,
            "tegra_sinterface": "serial_e"
        },
        "phys": [
            {
                "phy_idx": 2,
                "num_lanes": 4,
                "link_frequencies": [750000000],
                "clock_lanes": [0],
                "data_lanes": [1, 2, 3, 4],
                "pipe_stream_autoselect": false
            }
        ],
        "links": [
            {
                "name": "max96717",
                "cameras": [{"name": "imx219"}],
                "pool_addrs": ["0x50", "0x51"]
            },
            {
                "name": "max96717",
                "cameras": [{"name": "imx219"}],
                "pool_addrs": ["0x52", "0x53"]
            },
            {
                "name": "max96717",
                "cameras": [{"name": "imx219"}],
                "pool_addrs": ["0x54", "0x55"]
            },
            {
                "name": "max96717",
                "cameras": [{"name": "imx219"}],
                "pool_addrs": ["0x56", "0x57"]
            }
        ]
    }
]
```

## Usage

1. **Navigate to the Tool Directory:**

   ```bash
   cd gen_gmsl_dts
   ```

2. **Prepare Your JSON Configuration:**

   Pre-defined configuration JSON files are available for common setups. If none of these suit your use case, create a new JSON file. Refer to the [JSON Configuration Parameters](#json-configuration-parameters) section for details.

3. **Generate the DTS Overlay:**

   ```bash
   python3 gen_gmsl_dts.py --dtbo -o gmsl.dts max96724_4_max96717_imx219_tegra194.json
   ```

   This command will generate a DTS overlay file named `gmsl.dts` based on your configuration.

4. **Compile the DTS Overlay:**

   ```bash
   dtc -@ -I dts -O dtb -o gmsl.dtbo gmsl.dts
   ```

   Warnings from `dtc` are expected and harmless.

5. **Apply the Overlay to the Base DTB:**

   The base DTB must be a clean DTB with no pre-existing GMSL nodes. Apply the overlay using `fdtoverlay`:

   ```bash
   fdtoverlay -i tegra194-base.dtb -o tegra194-gmsl.dtb gmsl.dtbo
   ```

6. **Deploy to the Board:**

   Copy the merged DTB to the board:

   ```bash
   scp tegra194-gmsl.dtb <user>@<board>:/home/<user>/
   ```

   Example of `<user>`: `analog`

### **Note:** The following commands must be executed on the board

7. **Install on the Board:**

   Install the DTB on the board:

   ```bash
   sudo cp /home/<user>/tegra194-gmsl.dtb /boot/dtb/tegra194-gmsl.dtb
   ```

8. **Update the Boot Configuration:**

   Edit `/boot/extlinux/extlinux.conf` (using `vim` or `mousepad`) and update the `FDT` line to point to the new DTB:

   ```
   FDT /boot/dtb/tegra194-gmsl.dtb
   ```

9. **Reboot the Board:**

   ```bash
   sudo reboot
   ```

    **Note:** Due to the fact that the on-board registers might cache the previous configuration, a cold boot might be necessary in order to clear them:
    > 1. Run `sudo poweroff`
    > 2. Turn off and unplug the board
    > 3. Wait a few seconds, then plug the board back in
    > 4. Turn on the board

10. **Configure the Media Pipeline:**

    Before streaming video, the media pipeline must be configured with the correct format for your camera. Run the corresponding media configuration script:

    ```bash
    cd ~/Workspace
    ./media_config_imx219.sh   # for IMX219/IMX477 cameras
    ./media_config_ovx03a.sh   # for OX03A cameras
    ```

11. **Start the Video Stream:**

    ```bash
    cd ~/Workspace
    ./run_demo.sh
    ```

## Troubleshooting

- **Driver Probe:**

  Check that the deserializer and serializer drivers probed successfully:

  ```bash
  sudo dmesg | grep max
  ```

  The output shows the I2C bus and address (e.g., `max96724 1-0027` means bus 1, address 0x27). Use these bus numbers to scan for devices:

  ```bash
  sudo i2cdetect -y -r 1
  ```

- **Video Device Verification:**

  Check if the video devices are recognized:

  ```bash
  v4l2-ctl --list-devices
  ```

- **Media Device Verification:**

  Check if the media devices are recognized:

  ```bash
  media-ctl -p
  ```

  Your GMSL parts should appear in the media devices tree. There may be multiple devices created in `/dev/media*`. Select the correct one with the `-d` flag.

## References

- Analog Devices GMSL Linux Kernel Repository: [https://github.com/analogdevicesinc/linux](https://github.com/analogdevicesinc/linux)
- GMSL support landing page: [Gigabit Multimedia Serial Link (GMSL) technology from Analog Devices Inc.](https://github.com/analogdevicesinc/gmsl)
