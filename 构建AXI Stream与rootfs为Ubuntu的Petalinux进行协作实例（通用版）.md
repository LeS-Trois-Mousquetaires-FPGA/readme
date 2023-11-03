# 构建AXI Stream与rootfs为Ubuntu的Petalinux进行协作实例（通用版）

### 第一步 Vitis HLS创建IP核，并在Vivado中生成XSA

[VIVADO HLS Training AXI Stream interface #07 - YouTube](https://www.youtube.com/watch?v=3So1DPe2_4s)

https://github.com/smosanu/axi_stream_tutorial

注意事项：

https://github.com/bperez77/xilinx_axidma/issues/18#issuecomment-357294618

**AXI Lite在裸机程序中可用，在操作系统中不可用。**

所以需要添加

`#pragma HLS INTERFACE ap_ctrl_none port=return`

将ap_ctrl删除，仅使用AXI Stream

[Zynq linux加载axi_dma驱动报错 axidma: axidma_dma.c: axidma_request_channels: 651: Unable to get slave chan-CSDN博客](https://blog.csdn.net/m0_37545528/article/details/106300726)

不要忘记连接AXI DMA的中断信号！

### 第二步 构建Petalinux

#### 2.0 配置内核

[【精选】ZCU106开发（2）在Linux系统层上PL-PS通过AXI-DMA传输数据详细步骤及注意事项_zcu106 dma_aizaiyueye的博客-CSDN博客](https://blog.csdn.net/aizaiyueye/article/details/125032967)

Kernel Features---->Maximum count of the CMA areas 无需配置

dma-channel@后面的地址需要根据pl.dtsi修改

pl.dtsi和pcw.dtsi在一个目录里

#### 2.1 编译驱动

[Enable Linux 5.4 support by andrewvoznytsa · Pull Request #139 · bperez77/xilinx_axidma (github.com)](https://github.com/bperez77/xilinx_axidma/pull/139)

添加5.x内核驱动(也可以直接用我们的GitHub仓库)

编译完成后驱动文件在`\build\tmp\sysroots-components\zynq_generic\xilinx-axidma\lib\modules\5.15.19-xilinx-v2022.1\extra`里

每次petalinux-build后,需要重新复制新的驱动,否则不兼容

CMA大小可以通过`\project-spec\configs\config`中修改bootargs实现

#### 2.2 构建Ubuntu根文件系统

[【精选】制作适用于ZYNQ（ARM平台）的Ubuntu系统_xczu跑ubuntu系统_MarkusXu的博客-CSDN博客](https://blog.csdn.net/Markus_xu/article/details/117020452)

Certificate verification failed: The certificate is NOT trusted. The certificate chain uses expired certificate.  Could not handshake: Error in the certificate verification. [IP: 2402:f000:1:400::2 443]

```text
# https改为http
$ vim /etc/apt/sources.list

# 执行apt更新和重新安装证书
$ sudo apt-get update
$ sudo apt-get install --reinstall ca-certificates

# http改为https
$ vim /etc/apt/sources.list
```

ifupdown resolvconf

http://ports.ubuntu.com/ubuntu-ports/pool/main/i/ifupdown/ifupdown_0.8.35ubuntu1_armhf.deb

http://ports.ubuntu.com/ubuntu-ports/pool/main/r/resolvconf/resolvconf_1.78ubuntu7_all.deb

chroot: failed to run command ‘/bin/bash’: Exec format error

```
sudo update-binfmts --enable qemu-arm
```

[(31条消息) 打包开发板根文件系统，并制作成img镜像_歌舞丶升平的博客-CSDN博客](https://blog.csdn.net/zc21463071/article/details/106751361)

dns解析失败

```
nameserver 8.8.8.8
```

 `/etc/resolvconf/resolv.conf.d/base` 

```
nameserver 8.8.4.4
```

 `/etc/resolvconf/resolv.conf.d/head`

创建swap

```
sudo dd if=/dev/zero of=/opt/swap bs=1M count=1024
sudo mkswap /opt/swap
sudo swapon /opt/swap
sudo vim /etc/fstab
/opt/swap         swap          swap      defaults                             0      0
```

gpio permission denied

```
sudo groupadd gpio
sudo usermod -aG gpio <myusername>
su <myusername>
sudo chgrp gpio /sys/class/gpio/export
sudo chgrp gpio /sys/class/gpio/unexport
sudo chmod 775 /sys/class/gpio/export
sudo chmod 775 /sys/class/gpio/unexport
sudo chgrp gpio -HR /sys/class/gpio/gpio960 && sudo chmod -R 775 /sys/class/gpio/gpio960
```

制卡时，如果需要多分区，则启动分区应当为FAT16而非FAT32

### 第三步 Zynq板卡启动镜像

启动完成后使用`sudo insmod xilinx-axidma.ko`加载驱动到内核

使用`dmesg | grep dma`查看驱动是否加载成功，是否识别到ip核中的通道

正确加载驱动后再编写程序测试ip核功能是否正常