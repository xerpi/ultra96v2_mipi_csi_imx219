# Avnet Ultra96-V2 video pipeline for camera module with SONY IMX219 sensor using MIPI Adapter Mezzanine

## Import Block Design to Vivado

* File `design.tcl`

## Generate bitstream

1. _Generate Bitstream_ → _Include bitstream_

This will generate:
* `$PROJECT_NAME/$PROJECT_NAME.runs/impl_1/design_1_wrapper.bit`
* `$PROJECT_NAME/design_1_wrapper.xsa`

## Generate `boot.bin` and `boot_outer_shareable.bin`

0. `source /opt/Xilinx/Vivado/2022.1/settings64.sh`
1. `git clone https://github.com/ikwzm/ZynqMP-U-Boot-Ultra96-V2.git`
2. `cd ZynqMP-U-Boot-Ultra96-V2`
3. `cp $PROJECT_NAME/$PROJECT_NAME.runs/impl_1/design_1_wrapper.bit ./design_1_wrapper.bit`
4. `bootgen -arch zynqmp -image boot.bif -w -o boot.bin`
5. `bootgen -arch zynqmp -image boot_outer_shareable.bif -w -o boot_outer_shareable.bin`
6. `cp {boot.bin,boot_outer_shareable.bin} $SDCARD_BOOT_PARTITION`

## Generate Devicetree Source (DTS)

0. `source /opt/Xilinx/Vivado/2022.1/settings64.sh`
1. `git clone --branch xilinx_v2022.1_update2 https://github.com/Xilinx/device-tree-xlnx`
2. `cd $PROJECT_NAME`
3. `xsct`
    1. `hsi open_hw_design design_1_wrapper.xsa`
    2. `hsi set_repo_path ../device-tree-xlnx`
    3. `hsi create_sw_design device-tree -os device_tree -proc psu_cortexa53_0`
    4. `hsi generate_target -dir my_dts`
    5. `exit`

## Compile Linux and copy it to the boot partition

0. `source /opt/Xilinx/Vivado/2022.1/settings64.sh`
1. Follow [ZynqMP-FPGA-Linux's](https://github.com/ikwzm/ZynqMP-FPGA-Linux/blob/master/doc/build/linux-xlnx-v2021.1-zynqmp-fpga.md) guide:
  * Before building the kernel, make sure to enable the IMX219 driver:
    * `make ARCH=arm64 menuconfig`
    * _Device Drivers_ → _Multimedia support_ → _Media ancillary drivers_ → _Camera sensor devices_ → _Sony IMX219 sensor support_
  * Instead of `make deb-pkg`, you can just run:
        ```
        make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm64 -j 10 Image
        ```
2. `cp arch/arm64/boot/Image $SDCARD_BOOT_PARTITION/image-5.15.0-xlnx-v2022.1-zynqmp-fpga`

## Add a DTS node for the camera:

1. Create `linux-xlnx-v2021.1-zynqmp-fpga/arch/arm64/boot/dts/xilinx/ultra96v2_imx219.dtsi`
2. Add the following contents to `ultra96v2_imx219.dtsi`:
    ```dts
    #include <dt-bindings/gpio/gpio.h>

    / {
    	cam_clk: cam_clk {
    		status = "okay";
    		#clock-cells = <0>;
    		compatible = "fixed-clock";
    		clock-frequency = <24000000>;
    	};
    };

    &i2csw_2 {
    	imx219@10 {
    		compatible = "sony,imx219";
    		reg = <0x10>;
    		clocks = <&cam_clk>;
    		clock-names = "xclk";
    		reset-gpios = <&gpio 37 GPIO_ACTIVE_HIGH>;
    		status = "okay";

    		port {
    			imx219_to_mipi_csi2_rx_0: endpoint {
    				remote-endpoint = <&mipi_csi_invideo_pipeline_mipi_csi2_rx_subsyst_0>;
    				link-frequencies = /bits/ 64 <456000000>;
    				clock-noncontinuous;
    				clock-lanes = <0>;
    				data-lanes = <1 2>;
    			};
    		};
    	};
    };
    ```
3. `cp -rf $PROJECT_NAME/my_dts/* linux-xlnx-v2021.1-zynqmp-fpga/arch/arm64/boot/dts/xilinx/`
4. Modify `linux-xlnx-v2021.1-zynqmp-fpga/arch/arm64/boot/dts/xilinx/avnet-ultra96v2-rev1.dts` and add:
    ```dts
    #include <dt-bindings/media/xilinx-vip.h>
    #include "pl.dtsi"
    #include "ultra96v2_imx219.dtsi"

    &video_pipeline_mipi_csi2_rx_subsyst_0 {
    	mipi_csi_portsvideo_pipeline_mipi_csi2_rx_subsyst_0: ports {
    		mipi_csi_port0video_pipeline_mipi_csi2_rx_subsyst_0: port@0 {
    			mipi_csi_invideo_pipeline_mipi_csi2_rx_subsyst_0: endpoint {
    				remote-endpoint = <&imx219_to_mipi_csi2_rx_0>;
    			};
    		};
    	};
    };
    ```

## Compile the DTS and copy it to the boot partition

1. `make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm64 xilinx/avnet-ultra96v2-rev1.dtb`
2. `cp arch/arm64/boot/Image $SDCARD_BOOT_PARTITION/devicetree-5.15.0-xlnx-v2022.1-zynqmp-fpga-ultra96v2.dtb`

## Boot the system!

### Check that `/dev/video0` and `/dev/media0` exist
- `/dev/video0` and `/dev/media0` should exist. If not, check `dmesg` for errors.

  I recommend enabling debug logs in the media subsystem of Linux to help debugging:
  1. Modify `drivers/media/Makefile`
  2. Add:
        ```Makefile
        ccflags-y += -DDEBUG
        subdir-ccflags-y += -DDEBUG
        ```
  3. Recompile Linux and copy the `Image` to the boot partition


### Configure the video pipeline parameters

1. `media-ctl -v -d /dev/media0 -V '"imx219 3-0010":0 [fmt:SRGGB10_1X10/3280x2464]'`
2. `media-ctl -v -d /dev/media0 -V '"a0000000.mipi_csi2_rx_subsystem":1 [fmt:SRGGB10_1X10/3280x2464]'`
3. `media-ctl -v -d /dev/media0 -V '"a0020000.v_demosaic":1 [fmt:RBG888_1X24/3280x2464]'`
4. `media-ctl -v -d /dev/media0 -V '"a0080000.v_gamma_lut":1 [fmt:RBG888_1X24/3280x2464]'`
5. `media-ctl -v -d /dev/media0 -V '"a0040000.v_proc_ss":1 [fmt:RBG888_1X24/640x480]'`

You can change `[fmt:RBG888_1X24/640x480]` of `a0040000.v_proc_ss` to the desired format and dimensions.

`media-ctl -p` shows the video pipeline configuration.

`v4l2-ctl -d /dev/video0 --list-formats` to list the currently configured output formats.


### Take a picture!

- `v4l2-ctl --device /dev/video0 --set-fmt-video=width=640,height=480,pixelformat=RGB3 --stream-mmap --stream-to=frame.rgb --stream-count=1`

Make sure that the width, height and pixelformat match the ones configured at `"a0040000.v_proc_ss":1`!

You can `scp` the `frame.rgb` to the host and use a tool such as [YUView](http://ient.github.io/YUView/) to visualize it.
