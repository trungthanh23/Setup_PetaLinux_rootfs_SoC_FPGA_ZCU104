# Thiết lập môi trường PetaLinux, tạo Image khởi động và rootfs cho Linux trên SoC FPGA trên kit ZCU104
⚠️ **Lưu ý**: 
- Hướng dẫn này được hỗ trợ từ anh Phan Thành Luân - AISeQ Lab.
- Đường dẫn gốc: https://github.com/datnduit/Level_1_KV260_FPGA/tree/main
---
### I. Thiết lập môi trường PetaLinux và tạo driver

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

##### Tải bộ cài BSP cho ZCU104 FPGA từ trang chính thức Xilinx:
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
Sau đó chọn `Save` => `OK` => `Exit` => `Exit`

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

### II. Tạo image khởi động và rootfs cho Linux trên SoC FPGA

#### Sau khi build project thành công, gõ lệnh này để đóng gói file khởi động BOOT.BIN cùng với U-Boot phù hợp cho hệ thống.

```bash
petalinux-package --boot --force --u-boot
```
⚠️ **Lưu ý**: Vẫn thực hiện trong folder ZCU104 hoặc các folder tương ứng mọi người đang dùng để lưu.

#### Sau đó cắm SD card vào PC, tiến hàn phân vùng và định dạng thẻ nhớ SD, mọi người nên sử dụng thẻ nhớ 8GB trở nên.

#### Kiểm tra phân vùng thẻ sd card đã cắm
```bash
sudo fdisk -l
```
> Thông thường có dạng như kiểu `/dev/sda` hoặc `/dev/sdd`. Ví dụ ở đây của mình là `/dev/sda`.

#### Cấp quyền cho folder `image/`
```bash
chmod 777 image/
```

##### Tiến hành phân vùng cho sd card: 
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

#### Format để định dạng sd card
```bash
sudo mkfs.vfat -F 32 -n boot /dev/sda1
sudo mkfs.ext4 -L root /dev/sda2
```

#### Tiến hành mount thẻ SD card
1) Tạo folder
   ```bash
   sudo mkdir /mnt/boot
   sudo mkdir /mnt/root
   ```
2) Thực hiện mount thẻ SD card
   ```bash
   sudo mount /dev/sda1 /mnt/boot
   sudo mount /dev/sda2 /mnt/root
   ```
⚠️ **Lưu ý**: Tuy nhiêu, hệ điều hành sẽ tự động mount đến một vị trí khác vì vậy mọi người cần kiểm tra lại
	Kiểm tra lại ổ vị trí mount:
		```bash
		lsblk
		```
	Trên màn log sẽ hiện ví dụ như:
	```text
	NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
	loop0    7:0    0     4K  1 loop /snap/bare/5
	loop1    7:1    0 329,6M  1 loop /snap/code/209
	loop2    7:2    0  63,8M  1 loop /snap/core20/2669
	loop3    7:3    0  73,9M  1 loop /snap/core22/2111
	loop4    7:4    0  73,9M  1 loop /snap/core22/2133
	loop5    7:5    0 247,1M  1 loop /snap/firefox/6836
	loop6    7:6    0 247,6M  1 loop /snap/firefox/6966
	loop7    7:7    0 516,2M  1 loop /snap/gnome-42-2204/226
	loop8    7:8    0   516M  1 loop /snap/gnome-42-2204/202
	loop9    7:9    0  91,7M  1 loop /snap/gtk-common-themes/1535
	loop10   7:10   0  12,9M  1 loop /snap/snap-store/1113
	loop11   7:11   0  12,2M  1 loop /snap/snap-store/1216
	loop12   7:12   0  49,3M  1 loop /snap/snapd/24792
	loop13   7:13   0  50,8M  1 loop /snap/snapd/25202
	loop14   7:14   0   568K  1 loop /snap/snapd-desktop-integration/253
	loop15   7:15   0   576K  1 loop /snap/snapd-desktop-integration/315
	loop16   7:16   0 330,2M  1 loop /snap/code/210
	sda      8:0    0 465,8G  0 disk 
	├─sda1   8:1    0   512M  0 part /boot/efi
	└─sda2   8:2    0 465,3G  0 part /
	sdb      8:16   0 931,5G  0 disk 
	├─sdb1   8:17   0 931,5G  0 part /media/edabk/boot
	└─sda2   8:2    0 465,3G  0 part /media/edabk/root
	```
	Từ đây mọi người sử dụng 2 đường dẫn mới là `/media/edabk/boot` và `/media/edabk/root` thực hiện cho các bước tiếp theo.

3) Dịch chuyển file config vào boot
   ```bash
   	cd 
	sudo cp BOOT.BIN boot.scr image.ub /media/edabk/boot/
   ```
4) Giải nén deian linux
   📥 [Tải file Debian rootfs tại đây](https://drive.google.com/file/d/1ZcJYuVHpn8ER11nLCjwCUjfc5ykqP0tM/view?usp=sharing)

	> File rootfs này chứa hệ điều hành Debian đã được cấu hình sẵn cho kiến trúc ARM64, hỗ trợ giao diện XFCE và dễ dàng cài đặt thêm ứng dụng bằng `apt`.
	> Giải nén file zip để được file tar

	Giải nén file tar đến folder root
	```bash
 	sudo tar xfvp arm64-rootfs-debian-bullseye.tar -C /media/edabk/root/
 	```
 	⚠️ **Lưu ý**: file tar mọi người nếu cần phải trỏ đường dẫn đến đúng file này
5) Tạo ramdisk
   ```
