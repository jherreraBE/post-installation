# fedora_postinstall
Making the Quectel EM120R-GL LTE modem work with Linux (x-posted in /r/lenovo)
Tips and Tricks
Greetings traveler,

join me at my campfire while I regale you with the tall tale of making an LTE modem work with Linux on my Thinkpad T14 Gen 2 AMD.

So here's the device:

# lspci
[...]
05:00.0 Unassigned class [ff00]: Quectel Wireless Solutions Co., Ltd. EM120R-GL LTE Modem
The modem has 2 SIM Slots, one for eSIM and one for a physical SIM.

It defaults to the eSIM slot, and it refuses to work with mmcli to switch the slots like this:

# mmcli -v -m 0 --set-primary-sim-slot=0
So I figured out with intense googling that I had to switch the slot with:

# qmicli -d qrtr://0 --uim-sim-power-on=0
I've disable the SIM's PIN with:

# mmcli -v -m 0 --disable-pin --pin xxxx --sim 0
Before attempting a connection with ModemManager/NetworkManager, it has to be FCC unlocked (see MM's docs).

Googling this, I stumbled upon this, an unlocking procedure using a lib from a snap package provided by Lenovo. That's not necessary on a new enough ModemManager, all you have to do is:

# ln -snf /usr/share/ModemManager/fcc-unlock.available.d/1eac:1001 /etc/ModemManager/fcc-unlock.d/
Finally, the modem makes a successful connection with my provider's data. ModemManager-GUI crashes once the modem is unlocked, but that's due to an incompatibility with NetworkManager. Deleting the NM plugin fixes the crash, but removes NM integration:

# rm -f /usr/lib/modem-manager-gui/modules/libmodcm_nm09.so
The connection drops after a few minutes. EVERY TIME. The Lenovo Forums tell me about a firmware update that would fix the issue.

Great, fwupd will take care of that I thought.

Nope:

# fwupdmgr get-updates
Devices with no available firmware updates: 
 • UEFI Device Firmware
 • UEFI Device Firmware
 • UEFI Device Firmware
 • UEFI Device Firmware
 • EM120R GL
 • Integrated Camera
sigh Further investigations shows that most users updated their drivers in Windows, installing the firmware update. There's no Windows on my device. The device isn't USB, so I can't plug it into another device.

Quectel does provide a firmware flasher here.

$ git clone https://github.com/nippynetworks/qfirehose
$ cd qfirehose
$ make
The download Lenovo offers is a Windows Installer (groan)

$ innoextract nz7wj05w.exe
Extracting "1.0.0.30 (NZ7WJ05W)" - setup data version 5.5.7 (unicode)
 - "code$GetExtractPath$/Setup.bat" - overwritten
 - "code$GetExtractPath$/Setup.bat"
 - "code$GetExtractPath$/fw/firmware.zip"
 - "code$GetExtractPath$/fw/FWUpgradeTool_PCIe.exe"
 - "code$GetExtractPath$/fw/ImageInfo.xml"
 - "code$GetExtractPath$/fw/mbfwdriver.cat"
 - "code$GetExtractPath$/fw/QService.exe"
 - "code$GetExtractPath$/fw/QuectelFwUpdateDriver.dll"
 - "code$GetExtractPath$/fw/QuectelFwUpdateDriver.inf"
 - "code$GetExtractPath$/fw/WUProgressCtl.exe"
 - "code$GetExtractPath$/pcie/mhihost.cat"
 - "code$GetExtractPath$/pcie/MhiHost.inf"
 - "code$GetExtractPath$/pcie/ModemAuthService.exe"
 - "code$GetExtractPath$/pcie/qcude.cat"
 - "code$GetExtractPath$/pcie/qcude.inf"
 - "code$GetExtractPath$/pcie/mhi/amd64/MhiHost.sys"
 - "code$GetExtractPath$/pcie/qcude/amd64/qcude.sys"
 - "code$GetExtractPath$/usb/qcgnss.cat"
 - "code$GetExtractPath$/usb/qcgnss.inf"
 - "code$GetExtractPath$/usb/qcser.cat"
 - "code$GetExtractPath$/usb/qcser.inf"
 - "code$GetExtractPath$/usb/qmuxmdm.cat"
 - "code$GetExtractPath$/usb/QmuxMdm.inf"
 - "code$GetExtractPath$/usb/qusbccgp.cat"
 - "code$GetExtractPath$/usb/qusbccgp.inf"
 - "code$GetExtractPath$/usb/qcgnss/amd64/qcgnss.dll"
 - "code$GetExtractPath$/usb/qcmux/amd64/qcqmux.sys"
 - "code$GetExtractPath$/usb/qcmux/amd64/qcqmuxusb.sys"
 - "code$GetExtractPath$/usb/serial/amd64/qcusbser.sys"
$ unzip code\$GetExtractPath\$/fw/firmware.zip
$ unzip EM120RGLAPR02A09M4G.zip -d A09
Now I have firmware and flasher, let's get this show on the road. The firmware flasher complains about being unable to find the device. Huh? It wants to flash USB devices. Checking the source code that it indeed supports PCIe, I find a reference to /dev/mhi0 that doesn't exist on my system. WTF? More googling reveals that Quectel has a USB driver this firmware flasher depends on.

I acquired the zip file from a forum post from Quectel's forum.

So, let's use it... Of course it didn't compile on the latest available kernel, so I had to use the LTS kernel:

$ unzip Quectel_Linux_PCIE_MHI_Driver_V1.3.1.zip
$ cd pci_mhi
$ make KDIR=/lib/modules/5.15.82-1-lts/build
make ARCH=x86_64 CROSS_COMPILE= -C /lib/modules/5.15.82-1-lts/build M=/home/fuero/modem/pcie_mhi clean
make[1]: Entering directory '/usr/lib/modules/5.15.82-1-lts/build'
  CLEAN   /home/fuero/modem/pcie_mhi/Module.symvers
make[1]: Leaving directory '/usr/lib/modules/5.15.82-1-lts/build'
find . -name *.o.ur-safe | xargs rm -f
make ARCH=x86_64 CROSS_COMPILE= -C /lib/modules/5.15.82-1-lts/build M=/home/fuero/modem/pcie_mhi modules
make[1]: Entering directory '/usr/lib/modules/5.15.82-1-lts/build'
  CC [M]  /home/fuero/modem/pcie_mhi/core/mhi_init.o
  CC [M]  /home/fuero/modem/pcie_mhi/core/mhi_main.o
  CC [M]  /home/fuero/modem/pcie_mhi/core/mhi_pm.o
  CC [M]  /home/fuero/modem/pcie_mhi/core/mhi_boot.o
  CC [M]  /home/fuero/modem/pcie_mhi/core/mhi_dtr.o
  CC [M]  /home/fuero/modem/pcie_mhi/controllers/mhi_qti.o
  CC [M]  /home/fuero/modem/pcie_mhi/devices/mhi_uci.o
  CC [M]  /home/fuero/modem/pcie_mhi/devices/mhi_netdev_quectel.o
  LD [M]  /home/fuero/modem/pcie_mhi/pcie_mhi.o
  MODPOST /home/fuero/modem/pcie_mhi/Module.symvers
  CC [M]  /home/fuero/modem/pcie_mhi/pcie_mhi.mod.o
  LD [M]  /home/fuero/modem/pcie_mhi/pcie_mhi.ko
  BTF [M] /home/fuero/modem/pcie_mhi/pcie_mhi.ko
make[1]: Leaving directory '/usr/lib/modules/5.15.82-1-lts/build'
Fine. Don't forget to blacklist all mhi modules the kernel ships before loading the module.

Add /etc/modprobe.d/blacklist_mhi.conf with the following content:

blacklist mhi
blacklist mhi_ep
blacklist mhi_net
blacklist mhi_pci_generic
blacklist mhi_wwan_ctrl
blacklist mhi_wwan_mbim
Regenerate your initrd (in my case with mkinitcpio -P) and boot into the LTS kernel.

Now insmod the newly minted driver:

insmod ./pcie_mhi.ko
Then call the firmware flasher:

# ./QFirehose -p /dev/mhi_BHI -f ../A09
The firmware's flashed successfully, and the modem performs as expected now.


Upvote
14

Downvote

6
Go to comments

Share
