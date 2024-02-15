- (C) 2024 Sean.Hung <https://github.com/seanhungtw/usb-wifi_monitor-mode_on_WSL2>
- Extraordinary is you improve yourself every day

# usb-wifi_monitor-mode_on_WSL2
- This document records how to attach USB WIFI dongle on WSL2 and running it in monitor mode so we can use it to capture WIFI packets.
- Why use WSL2 to capture wireless packets?
  - WSL2 is more efficient that virtualbox / VMware
  - Some times you only has 1 windows PC, it's convenient if you can finish your job with 1 PC.
- The chips I tested:

| Chips | Tested WSL2 kernel version | Driver | monitor mode on WSL2 |
| --- | --- | --- | --- |
| Mediatek MT7921au | wsl-5.15.146.1 (need porting driver from 5.18) | mt7921u | can work |
| Realtek rtl8192eu | wsl-5.15.146.1 | rtl8xxxu | can work |

# Step by Step
1. let's check your WIFI dongle is supported monitor mode on Linux kernel:
   - you can refer this page to check which dongle you can use:
     - https://github.com/morrownr/USB-WiFi/blob/main/home/USB_WiFi_Chipsets.md
2. You also need usbipd-win to mount USB devices on WSL2
   - Brilliant tool
     - https://github.com/dorssel/usbipd-win
3. Let's build the customized WSL2 kernel
   - Why? Because WSL2 kernel doesn't include any Wi-Fi drivers, We need to add it by ourself.
   - Let's get the code
    ```
      sudo apt install build-essential flex bison libssl-dev libelf-dev git dwarves
      git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
      cd WSL2-Linux-Kernel
      # copy default .config
      cp Microsoft/config-wsl .config
    ```
    - Then, you need to select your WIFI driver by `make menuconfig`
    ```
      # you need to select your WIFI driver
      make menuconfig
    ```
      - It's different for each WIFI chips, take rtl8192eu for example:
      - you need to eanble `CONFIG_MAC80211=m` `CONFIG_CFG80211=m` `CONFIG_RTL8XXXU=m`
      - And you need some trik to make WSL2 to load WIFI firmware
      - I downloaded rtl8192eu firmware here:
        - https://anduin.linuxfromscratch.org/sources/linux-firmware/rtlwifi/
      - And you can follow the steps in this comment to compile the kernel with your firmware
        - https://github.com/dorssel/usbipd-win/issues/390#issuecomment-1223615102
4. Let's build the code
   ```
     make modules
     make modules_install
     make -j $(expr $(nproc) - 1)
   ```
6. If no error, you shold got your kernel file, copy it to your Windows Host:
   ```
     cp arch/x86/boot/bzImage /mnt/c/Users/$user
   ```
7. Add a `.wslconfig` in your Users/$user directory, add `kernel` option which point to the kernel file you just build.
   ```
   [wsl2]
   kernel=C:\\Users\\$user\\bzImage
   ```
8. restart WSL2 (wsl --shutdown)
9. load WIFI driver module:
  - in my case I need to load rtl8xxxu driver module:
    ```
    sudo modprobe rtl8xxxu
    ```
  - And you can check everything is fine using lsmod:
    ```
    abcdefg@DESKTOP-123456:~$ lsmod
    Module                  Size  Used by
    rtl8xxxu              114688  0
    mac80211              888832  1 rtl8xxxu
    cfg80211              835584  2 mac80211,rtl8xxxu
    ```
  - you can also check if anything goes wrong with `sudo dmesg` :
    ```
      ~$ sudo dmesg
      [114750.039217] usb 1-4: RTL8192EU rev B (SMIC) 2T2R, TX queues 3, WiFi=1, BT=0, GPS=0, HI PA=0
      [114750.039220] usb 1-4: RTL8192EU MAC: 1c:bf:ce:94:bb:39
      [114750.039221] usb 1-4: rtl8xxxu: Loading firmware rtlwifi/rtl8192eu_nic.bin
      [114750.039279] usb 1-4: Firmware revision 19.0 (signature 0x92e1)
      [114757.126740] usb 1-4: rtl8192eu_rx_iqk_path_b: Path B RX IQK failed!
      [114758.694017] usbcore: registered new interface driver rtl8xxxu
      [114758.697056] rtl8xxxu 1-4:1.0 wlx1cbfce94bb39: renamed from wlan0
    ```

## How to change the WiFi interface to monitor mode and change channel
```
sudo iw dev
sudo ifconfig wlan0 down
sudo iw wlan0 set monitor none
sudo ifconfig wlan0 up

#set channel you want to monitor
sudo iw dev wlan0 set channel 6
```

### Running Wireshark
```
sudo wireshark
```
