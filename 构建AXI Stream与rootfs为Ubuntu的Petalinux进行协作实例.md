# 构建AXI Stream与rootfs为Ubuntu的Petalinux进行协作实例

第一步

Vitis HLS创建IP核，并在Vivado中生成XSA

[VIVADO HLS Training AXI Stream interface #07 - YouTube](https://www.youtube.com/watch?v=3So1DPe2_4s)

https://github.com/smosanu/axi_stream_tutorial

记得添加约束文件

```
set_property PACKAGE_PIN K16 [get_ports {GPIO_0_0_tri_io[0]}]
set_property PACKAGE_PIN J16 [get_ports {GPIO_0_0_tri_io[1]}]
set_property PACKAGE_PIN G18 [get_ports {GPIO_0_0_tri_io[2]}]
set_property PACKAGE_PIN G17 [get_ports {GPIO_0_0_tri_io[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {GPIO_0_0_tri_io[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {GPIO_0_0_tri_io[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {GPIO_0_0_tri_io[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {GPIO_0_0_tri_io[0]}]
set_property PACKAGE_PIN C20 [get_ports {RGMII_0_td[1]}]
set_property PACKAGE_PIN E19 [get_ports RGMII_0_tx_ctl]
set_property PACKAGE_PIN D20 [get_ports {RGMII_0_td[3]}]
set_property PACKAGE_PIN B20 [get_ports {RGMII_0_td[0]}]
set_property PACKAGE_PIN D18 [get_ports {RGMII_0_rd[2]}]
set_property PACKAGE_PIN F20 [get_ports MDIO_PHY_0_mdio_io]
set_property PACKAGE_PIN F19 [get_ports MDIO_PHY_0_mdc]
set_property PACKAGE_PIN D19 [get_ports {RGMII_0_td[2]}]
set_property PACKAGE_PIN B19 [get_ports {RGMII_0_rd[3]}]
set_property PACKAGE_PIN F17 [get_ports {RGMII_0_rd[0]}]
set_property PACKAGE_PIN A20 [get_ports RGMII_0_txc]
set_property PACKAGE_PIN J18 [get_ports RGMII_0_rxc]
set_property PACKAGE_PIN E17 [get_ports RGMII_0_rx_ctl]
set_property PACKAGE_PIN E18 [get_ports {RGMII_0_rd[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_td[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports RGMII_0_tx_ctl]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_td[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_td[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_rd[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports MDIO_PHY_0_mdio_io]
set_property IOSTANDARD LVCMOS33 [get_ports MDIO_PHY_0_mdc]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_td[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_rd[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_rd[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports RGMII_0_txc]
set_property IOSTANDARD LVCMOS33 [get_ports RGMII_0_rxc]
set_property IOSTANDARD LVCMOS33 [get_ports RGMII_0_rx_ctl]
set_property IOSTANDARD LVCMOS33 [get_ports {RGMII_0_rd[1]}]


create_clock -period 8.000 -name RGMII_0_rxc -waveform {0.000 4.000} [get_ports RGMII_0_rxc]
set_clock_groups -logically_exclusive -group [get_clocks -include_generated_clocks {gmii_clk_25m_out gmii_clk_2_5m_out}] -group [get_clocks -include_generated_clocks gmii_clk_125m_out]

set_property SLEW FAST [get_ports {RGMII_0_td[0]}]
set_property SLEW FAST [get_ports {RGMII_0_td[1]}]
set_property SLEW FAST [get_ports {RGMII_0_td[2]}]
set_property SLEW FAST [get_ports {RGMII_0_td[3]}]
set_property SLEW FAST [get_ports RGMII_0_tx_ctl]
set_property SLEW FAST [get_ports RGMII_0_txc]

set_property SLEW FAST [get_ports MDIO_PHY_0_mdc]
set_property SLEW FAST [get_ports MDIO_PHY_0_mdio_io]
```

注意事项：

https://github.com/bperez77/xilinx_axidma/issues/18#issuecomment-357294618

**AXI Lite在裸机程序中可用，在操作系统中不可用。**

所以需要添加

`#pragma HLS INTERFACE ap_ctrl_none port=return`

将ap_ctrl删除，仅使用AXI Stream

[Zynq linux加载axi_dma驱动报错 axidma: axidma_dma.c: axidma_request_channels: 651: Unable to get slave chan-CSDN博客](https://blog.csdn.net/m0_37545528/article/details/106300726)

不要忘记连接AXI DMA的中断信号！

第二步

Petalinux配置内核

[【精选】ZCU106开发（2）在Linux系统层上PL-PS通过AXI-DMA传输数据详细步骤及注意事项_zcu106 dma_aizaiyueye的博客-CSDN博客](https://blog.csdn.net/aizaiyueye/article/details/125032967)

Kernel Features---->Maximum count of the CMA areas 无需配置

dma-channel@后面的地址需要根据pl.dtsi修改

pl.dtsi和pcw.dtsi在一个目录里

pcw.dtsi修改内容

```
&gem0 {
	phy-handle = <&phy0>;
	phy-mode = "gmii";
	status = "okay";
	xlnx,ptp-enet-clock = <0x69f6bcb>;
	ps7_ethernet_0_mdio: mdio {
		#address-cells = <1>;
		#size-cells = <0>;
		phy0: phy@0 {
			device_type = "ethernet-phy";
			reg = <0>;
		};
		gmii_to_rgmii_0: gmii_to_rgmii_0@8 {
			compatible = "xlnx,gmii-to-rgmii-1.0";
			phy-handle = <&phy0>;
			reg = <8>;
		};
	};
};
```

system-user.dtsi内容

```
/include/ "system-conf.dtsi"
/ {
};
/{
  usb_phy0: usb_phy@0 {
  compatible = "ulpi-phy";
  #phy-cells = <0>;
  reg = <0xe0002000 0x1000>;
  view-port = <0x0170>;
  reset-gpios = <&gpio0 46 1>;
  drv-vbus;
 };
};
&usb0 {
 dr_mode = "host";
 usb-phy = <&usb_phy0>;
};
 &amba_pl{
    axidma_chrdev: axidma_chrdev@0 {
            compatible = "xlnx,axidma-chrdev";
            dmas = <&axi_dma_0 0 &axi_dma_0 1>;
            dma-names = "tx_channel", "rx_channel";
    };};
 &axi_dma_0{
    dma-channel@40400000 {
        xlnx,device-id = <0x0>;
    };
    dma-channel@40400030 {
        xlnx,device-id = <0x1>;
    };};
```

2.1 编译驱动

[Enable Linux 5.4 support by andrewvoznytsa · Pull Request #139 · bperez77/xilinx_axidma (github.com)](https://github.com/bperez77/xilinx_axidma/pull/139)

添加5.x内核驱动

编译完成后驱动文件在\build\tmp\sysroots-components\zynq_generic\xilinx-axidma\lib\modules\5.15.19-xilinx-v2022.1\extra里

每次petalinux-build后,需要重新复制新的驱动,否则不兼容

CMA大小可以通过\project-spec\configs\config中修改bootargs实现