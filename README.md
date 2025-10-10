# Thiết lập môi trường PetaLinux, tạo Image khởi động và rootfs cho Linux trên SoC FPGA trên kit ZCU104
⚠️ **Lưu ý**: 
- Hướng dẫn này được hỗ trợ từ anh Phan Thành Luân - AISeQ Lab.
- Đường dẫn gốc: https://github.com/datnduit/Level_1_KV260_FPGA/tree/main
---
### Thiết lập môi trường PetaLinux và tạo driver

Sau khi hoàn tất thiết kế phần cứng và tạo Block Design trong Vivado, bước tiếp theo là **xuất file phần cứng (`.xsa`)** để sử dụng trong PetaLinux nhằm tạo hệ điều hành và driver phù hợp cho hệ thống.

#### 1. Xuất file phần cứng (`.xsa`) từ Vivado

- Trong Vivado, sau khi **Generate Bitstream** thành công:
  - Vào menu: **File → Export → Export Hardware**
  - Chọn:Include bitstream
  - File `.xsa` sẽ được tạo ra (ví dụ: `SoC_wrapper.xsa`)

#### 2. Cài đặt PetaLinux

- Tải bộ cài **PetaLinux 2022.2** từ trang chính thức Xilinx:
    🔗 https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/archive.html


##### Cài đặt các gói phụ thuộc (Ubuntu/Debian)

```bash
sudo apt-get install tofrodos gawk xvfb git libncurses5-dev tftpd zlib1g-dev zlib1g-dev:i386 \
libssl-dev flex bison chrpath socat autoconf libtool texinfo gcc-multilib \
libsdl1.2-dev libglib2.0-dev screen pax libtinfo5 xterm build-essential net-tools
```
	
##### Cấp quyền thực thi cho file `.run`

```bash
chmod +x petalinux-v2022.2-*.run
```

#####  Chạy trình cài đặt

```bash
./petalinux-v2022.2-*.run
```

- Trong quá trình cài đặt, trình cài đặt sẽ hiển thị các thỏa thuận bản quyền:
	- Dùng PgUp / PgDn để đọc
	- Nhấn q để thoát khỏi phần hiển thị
	- Nhấn y để đồng ý và tiếp tục

#### 3. Xây dựng môi trường phần cứng

##### Thiết lập môi trường làm việc Petalinux

##### **Source** đến thư mục cài đặt Petalinux để sử dụng được các lệnh `petalinux-*`:
```bash
source <đường_dẫn_cài_petalinux>/2022.2/settings.sh
```

##### Tải bộ cài BSP cho KV260 FPGA từ trang chính thức Xilinx:
    🔗 https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/archive.html

##### Tạo project PetaLinux từ BSP
```bash	
petalinux-create -t project -s <đường_dẫn_tới_file_BSP>.bsp --name ZCU104_Linux
cd ZCU104_Linux
 ```
 
##### Import phần cứng (.xsa) vào project Sau khi bạn export file .xsa từ Vivado (có chứa bitstream), hãy dùng lệnh sau để tích hợp phần cứng vào project:
```bash
petalinux-config --get-hw-description=<path_to_the_hw_description_file> 
 ```
##### Cấu hình kernel bootargs thủ công Sau khi chạy petalinux-config, hệ thống sẽ mở giao diện curses để bạn cấu hình sâu hơn. Điều chỉnh cấu hình kernel bootargs Trong cửa sổ cấu hình, thực hiện các bước sau:
 1) Thiết lập bootargs
 ```text
Subsystem AUTO Hardware Settings  --->
    DTG Settings  --->
        Kernel Bootargs  --->
            [ ] generate boot args automatically
            (user-defined) user set kernel bootargs
 ```
 
Dán đoạn bootargs dưới đây vào phần user set kernel bootargs:
```bash
earlycon clk_ignore_unused cpuidle.off=1 root=/dev/mmcblk0p2 rw rootwait uio_pdrv_genirq.of_id=generic-uio
 ```
📌 Cấu hình này giúp khởi động đúng thiết bị, bật driver UIO, cấp vùng bộ nhớ CMA, và giữ clock cho các IP tự thiết kế trong PL.
2) Thiết lập SD card
```text
Image Packaing Configuration --->
    Root filesystem type (INITRD) --->
        (X) EXT4  (SD/eMMC/SATA/USB)
```
##### Chỉnh sửa Device Tree (system-user.dtsi)

Để hệ điều hành Linux có thể sử dụng **IP tự thiết kế trong PL** thông qua driver `uio`, bạn cần chỉnh sửa file **Device Tree Overlay**.
Trong file ở đường dẫn `ZCU104_Linux/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`, chỉnh lại file thành: 
```dts
/include/ "system-conf.dtsi"
/ {
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;
        reserved: buffer@0 {
                no-map;
                reg = <0x8 0x0 0x0 0x80000000>;
        };
    };

    amba: axi {
        /* GDMA */
        fpd_dma_chan1: dma-controller@fd500000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan2: dma-controller@fd510000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan3: dma-controller@fd520000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan4: dma-controller@fd530000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan5: dma-controller@fd540000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan6: dma-controller@fd550000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan7: dma-controller@fd560000 {
            compatible = "generic-uio";
        };

        fpd_dma_chan8: dma-controller@fd570000 {
            compatible = "generic-uio";
        };
    };

    amba_pl@0 {
        MY_IP@a0000000 {
                compatible = "generic-uio";
        };
    };

    ddr_high@000800000000 {
        compatible = "generic-uio";
        reg = <0x8 0x0 0x0 0x80000000>;
    };
};

```
##### Sau đó tiến hành build project

```bash
petalinux-build
```
⚠️ **Lưu ý**:  Build sẽ tốn tầm 30 phút đến 1 tiếng.

### Tạo image khởi động và rootfs cho Linux trên SoC FPGA

Sau khi build project thành công, gõ lệnh này để đóng gói file khởi động BOOT.BIN cùng với U-Boot phù hợp cho hệ thống.

```bash
petalinux-package --boot --force --u-boot
```
⚠️ **Lưu ý**: Vẫn thực hiện trong folder ZCU104 hoặc các folder tương ứng mọi người đang dùng để lưu.

Sau đó cắm SD card vào PC, tiến hàn phân vùng và định dạng thẻ nhớ SD.

📥 [Tải file Debian rootfs tại đây](https://drive.google.com/file/d/1ZcJYuVHpn8ER11nLCjwCUjfc5ykqP0tM/view?usp=sharing)

> File rootfs này chứa hệ điều hành Debian đã được cấu hình sẵn cho kiến trúc ARM64, hỗ trợ giao diện XFCE và dễ dàng cài đặt thêm ứng dụng bằng `apt`.
> Giải nén file zip để được file tar

Kiểm tra phân vùng thẻ sd card đã cắm
```bash
sudo fdisk -l
```
> Thông thường có dạng như kiểu `/dev/sda` hoặc `/dev/sdd`. Ví dụ ở đây của mình là `/dev/sda`.

Cấp quyền cho folder `image/`
```bash
chmod 777 image/
```

Tiến hành phân vùng cho sd card: 
1) Nhập lệnh:
   ``` bash
   sudo fdisk /dev/sda
   ```
2) Với từng tùy chọn ta chọn như sau: 
  ```text
  Command (m for help) : [Điền n]

  ...

  Select (default p): [Bấm Enter]

  ...

  Partition number (1-4, default 1): [Bấm Enter]

  ...

  Last sector, +/-sectors or +/-size{K,M,G,T,P} (...): [Điền +1GB]

  ...

  Do you want to remove the signature [Y]es/[N]o: [Điền y]

  ...

  Command (m for help) : [Điền n]

  ...

  Select (default p): [Bấm Enter]

  ...

  First sector (...): [Bấm Enter]

  ...

  Last sector, +/-sectors or +/-size{K,M,G,T,P} (...): [Bấm Enter]

  ...

  Do you want to remove the signature [Y]es/[N]o: [Điền y]

  ...

  Command (m for help) : [Điền w]
  ```

Format để định dạng sd card
```bash
sudo mkfs.vfat -F 32 -n boot /dev/sda1
sudo mkfs.ext4 -L root /dev/sda2
