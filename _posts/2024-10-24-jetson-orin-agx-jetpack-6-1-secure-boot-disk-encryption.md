---
layout: post
title: Configuring NVidia Jetson Orin AGX to securely boot Jetpack 6.1 from an encrypted NVMe drive
---

Perusing the NVidia forums for this information is quite time consuming so I thought this might come in handy for others. Also, you might just make a small mistake and spend the next day looking for it, much like [I did](https://forums.developer.nvidia.com/t/uefi-secureboot-and-disk-encryption-in-36-4/310592/6).

When I was halfway through doing this for Jetpack 5.1.2 I was very lucky to run into [this post by Chris](https://medium.com/@monoclechris/secureboot-and-encrypted-rootfs-nvme-on-jetson-orin-41d3356d7922). When you're fine using the Ubuntu 20-based Jetpack 5 then that post is all you need.

In case you want to use Ubuntu-22 then the following instructions should be useful.

I'm using the same structure as Chris, as I based my notes on his. In some cases I'll also blatantly include entire paragraphs, I hope he does not mind. I'll also try to point out when there are changes compared to the workflow for Jetpack 5.


# Setup

I tested this on an Ubuntu 20 host, I believe 22 should also work now (though for flashing Jetpack 5 I ran into issues as for some flashing scripts ‘python3’ had to link to a python version <=3.8)

You can use the following to create a conda environment with the appropriate python version and install the required libraries:
```
conda create --name tegra
conda activate tegra
conda install python=3.8
apt-get install dislocker cryptsetup libcryptsetup-dev libcryptsetup12 cryptmount overlayroot qemu-user-static openssl device-tree-compiler efitools uuid-runtime
pip install cryptography
pip install pycrypto
conda install -c anaconda pyyaml
```
You could try skipping the lines starting with conda but this is what worked for me for both Jetpack 5 and 6.

Just in case, also disable the USB autosuspend :
```
sudo systemctl stop udisks2
sudo -s echo -1 > /sys/module/usbcore/parameters/autosuspend
sudo ufw disable
```

## Download the tarballs

The first thing to do is download and unpack the tarballs for Jetson Linux 36.4. Make sure they are all exactly the same version, otherwise you are very likely to run into issues.
```
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.0/release/Jetson_Linux_R36.4.0_aarch64.tbz2
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.0/release/Tegra_Linux_Sample-Root-Filesystem_R36.4.0_aarch64.tbz2
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v4.0/sources/public_sources.tbz2

tar xvf Jetson_Linux_R36.4.0_aarch64.tbz2
sudo tar xvf Tegra_Linux_Sample-Root-Filesystem_R36.4.0_aarch64.tbz2 -C Linux_for_Tegra/rootfs/
tar xvf public_sources.tbz2
cd Linux_for_Tegra/source
tar xvf nvidia-jetson-optee-source.tbz2
cd ../..
sudo tools/l4t_flash_prerequisites.sh
sudo ./apply_binaries.sh
```
_(there is a slight change compared to Jetpack 5.1.2 where the optee sources were in source/public earlier)_

## (Possibly redundant) Perform a 'simple' flash first

This step might not be necessary for you, but I've run into an issue with some Orin AGX-s where it was necessary to update the bootloader on the internal eMMC as older versions don’t support booting from NVME (UEFI firmware version was 4.1x after flashing with L4T 35.4.1, and 1.1x with the base factory image).

Also if you don't require disk encryption and secure boot, then this is what you would use.

To run the flashing scripts, connect the device to the host via the USB-C port and enter force recovery mode as shown [here](https://developer.nvidia.com/embedded/learn/jetson-agx-orin-devkit-user-guide/howto.html#force-recovery-mode).

Make sure you see this:
```
$ lsusb
…
Bus 003 Device 015: ID 0955:7023 NVIDIA Corp. APX
```
Sometimes (perhaps when not holding down the middle button long enough) it does not boot into force recovery mode, and instead you see `“NVIDIA Corp. L4T (Linux for Tegra) running on Tegra”`

NB: for me the “when powered ON” this is the correct sequence to enter force recovery mode (NB: important to hold recovery pressed for 3s): 
- wait for boot LED to light up
- press the reset & recovery buttons together
- release the reset button
- release the recovery button after 3 seconds later. This will set it to Recovery mode

Then, in /Linux_for_Tegra, run:
```
sudo ./nvsdkmanager_flash.sh internal
```
 
Now you have the Orin flashed with Jetpack 6.1 without secure boot/disk encryption. If you need those (why else are you here?), read on.

## Generating Keys

You will need a private key and its hash, a SBK(Secure Boot Key) and a KEK(Key Encryption Key) in a variety of formats. You will need to keep these safe and you will need them again if you want to reflash your device.
```
openssl genrsa -out rsa.pem 3072
PKCS_KEY_XML_HASH=$(./bootloader/tegrasign_v3.py --pubkeyhash rsa.pubkey rsa.hash --key rsa.pem | grep "tegra-fuse format" | awk '{print $NF}')
SBK_0=$(openssl rand -hex 4)
SBK_1=$(openssl rand -hex 4)
SBK_2=$(openssl rand -hex 4)
SBK_3=$(openssl rand -hex 4)
SBK_4=$(openssl rand -hex 4)
SBK_5=$(openssl rand -hex 4)
SBK_6=$(openssl rand -hex 4)
SBK_7=$(openssl rand -hex 4)
SBK_KEY=$(echo "0x${SBK_0} 0x${SBK_1} 0x${SBK_2} 0x${SBK_3} 0x${SBK_4} 0x${SBK_5} 0x${SBK_6} 0x${SBK_7}")
echo "${SBK_KEY}" > sbk.key
SBK_KEY_XML="0x${SBK_0}${SBK_1}${SBK_2}${SBK_3}${SBK_4}${SBK_5}${SBK_6}${SBK_7}"
echo "${SBK_KEY_XML}" > sbk_xml.key
KEK_2_0=$(openssl rand -hex 4)
KEK_2_1=$(openssl rand -hex 4)
KEK_2_2=$(openssl rand -hex 4)
KEK_2_3=$(openssl rand -hex 4)
KEK_2_4=$(openssl rand -hex 4)
KEK_2_5=$(openssl rand -hex 4)
KEK_2_6=$(openssl rand -hex 4)
KEK_2_7=$(openssl rand -hex 4)
KEK_2_KEY=$(echo "0x${KEK_2_0} 0x${KEK_2_1} 0x${KEK_2_2} 0x${KEK_2_3} 0x${KEK_2_4} 0x${KEK_2_5} 0x${KEK_2_6} 0x${KEK_2_7}")
echo "${KEK_2_KEY}" > kek.key
KEK_2_KEY_XML="0x${KEK_2_0}${KEK_2_1}${KEK_2_2}${KEK_2_3}${KEK_2_4}${KEK_2_5}${KEK_2_6}${KEK_2_7}"
echo "${KEK_2_KEY_XML}" > kek_xml.key
KEK_2_KEY_OPTEE="${KEK_2_0}${KEK_2_1}${KEK_2_2}${KEK_2_3}${KEK_2_4}${KEK_2_5}${KEK_2_6}${KEK_2_7}"
echo "${KEK_2_KEY_OPTEE}" > kek_optee.key

openssl rand -rand /dev/urandom -hex 32 > sym_t234.key
openssl rand -rand /dev/urandom -hex 16 > sym2_t234.key 
openssl rand -rand /dev/urandom -hex 16 > auth_t234.key
```

The following is copied pretty much verbatim from [The NVidia Secure Boot docs](https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/SD/Security/SecureBoot.html#prepare-the-pk-kek-db-keys):
```
mkdir uefi_keys
cd uefi_keys
GUID=$(uuidgen)
```
Generate PK RSA Key Pair, Certificate, and EFI Signature List File
```
openssl req -newkey rsa:2048 -nodes -keyout PK.key  -new -x509 -sha256 -days 3650 -subj "/CN=my Platform Key/" -out PK.crt
cert-to-efi-sig-list -g "${GUID}" PK.crt PK.esl
```

Generate KEK RSA Key Pair, Certificate, and EFI Signature List File
```
openssl req -newkey rsa:2048 -nodes -keyout KEK.key  -new -x509 -sha256 -days 3650 -subj "/CN=my Key Exchange Key/" -out KEK.crt
cert-to-efi-sig-list -g "${GUID}" KEK.crt KEK.esl
```
Generate db_1 RSA Key Pair, Certificate, and EFI Signature List File
```
openssl req -newkey rsa:2048 -nodes -keyout db_1.key  -new -x509 -sha256 -days 3650 -subj "/CN=my Signature Database key/" -out db_1.crt
cert-to-efi-sig-list -g "${GUID}" db_1.crt db_1.esl
```
Generate db_2 RSA Key Pair, Certificate, and EFI Signature List File

```
openssl req -newkey rsa:2048 -nodes -keyout db_2.key  -new -x509 -sha256 -days 3650 -subj "/CN=my another Signature Database key/" -out db_2.crt
cert-to-efi-sig-list -g "${GUID}" db_2.crt db_2.esl
```
Create a uefi_keys.conf file and insert the following:
```
UEFI_DB_1_KEY_FILE="db_1.key";  # UEFI payload signing key
UEFI_DB_1_CERT_FILE="db_1.crt"; # UEFI payload signing key certificate

UEFI_DEFAULT_PK_ESL="PK.esl"
UEFI_DEFAULT_KEK_ESL_0="KEK.esl"

UEFI_DEFAULT_DB_ESL_0="db_1.esl"
UEFI_DEFAULT_DB_ESL_1="db_2.esl"
```
```
cd ..
sudo tools/gen_uefi_keys_dts.sh uefi_keys/uefi_keys.conf
```

# Generating the OP-TEE image
Note: when re-flashing with the same keys for any reason, you can start from here.

Flashing the encrypted rootfs on the host with flash.sh for the Jetson AGX Orin requires an update of the `Linux_for_Tegra/bootloader/eks_<platform>.img`.

Use the following command to create that file:
```
python3 ./source/optee/samples/hwkey-agent/host/tool/gen_ekb/gen_ekb.py -chip t234 -oem_k1_key kek_optee.key -in_sym_key sym_t234.key -in_sym_key2 sym2_t234.key -in_auth_key auth_t234.key -out bootloader/eks_t234.img
```
Some notes: 
- in the linked guide OEM_K2 key was used, I switched to using OEM_K1 as I read somewhere in the docs that was the recommendation. Possibly OEM_K2 still works as well)
- change compared to Jetpack 5.1.2: the `fv` argument is not required anymore, [as per this](https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/SD/Security/OpTee.html#tool-for-ekb-generation)
- change compared to Jetpack 5.1.2: a new argument `in_auth_key` is required, for the UEFI variable authentication key

# Fusing
Fuses on the device are secure ROM that can be only written to once in an irreversible process. This is the part where you need to be very careful. The BootSecurityInfo is a bitmap that can be calculated from the Fuse specification linked above.

```
echo "<genericfuse MagicId=\"0x45535546\" version=\"1.0.0\">" > fuse.xml
echo "  <fuse name=\"PublicKeyHash\" size=\"64\" value=\"${PKCS_KEY_XML_HASH}\"/>" >> fuse.xml
echo "  <fuse name=\"SecureBootKey\" size=\"32\" value=\"${SBK_KEY_XML}\"/>" >> fuse.xml
echo "  <fuse name=\"OemK1\" size=\"32\" value=\"${KEK_2_KEY_XML}\"/>" >> fuse.xml
echo "  <fuse name=\"BootSecurityInfo\" size=\"4\" value=\"0x209\"/>" >> fuse.xml
echo "  <fuse name=\"SecurityMode\" size=\"4\" value=\"0x1\"/>" >> fuse.xml
echo "</genericfuse>" >> fuse.xml
```
_(Note: again, using OemK1 instead of OemK2 here)_

It's recommended to run with `--test` first just in case:
```
sudo ./odmfuse.sh -i 0x23 -k rsa.pem -S sbk.key -X fuse.xml --test jetson-agx-orin-devkit
``` 
If everything is good, remove `--test` and re-run. NB: after the `SecurityMode` (also known as `odm_production_mode`) fuse is burned with a value of `0x1`, all additional fuse write requests will be blocked ([source](https://docs.nvidia.com/jetson/archives/r36.4/DeveloperGuide/SD/Security/SecureBoot.html#fuses-and-security)).

NB if you get an error like:
```
[   0.1437 ] ERROR: might be timeout in USB write.
```
the USB autosuspend is to blame. Run the commands listed in the beginning (search for 'autosuspend'), pull out the USB cable, put Jetson back into recovery mode again and plug the USB cable back in.

# QSPI Setup
Before continuing, you must replace the `“NUM_SECTORS”` text inside `“tools/kernel_flash/flash_l4t_t234_nvme_rootfs_enc.xml”`. This is because the host scripts are not able to determine the size of the disk in the Jetson, so you have to calculate it manually. 
Once set, then this script below will create an image for the QSPI, but not flash it, then copy it to a location where it will be flashed later. The device must be plugged in during this phase, because it will be queried by the scripts.
For 1TB disks the NUM_SECTORS value I used is “1953525168”. The following commands also assume a 1TB disk is used.
```
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --network usb0 -u ./rsa.pem -v ./sbk.key --uefi-keys uefi_keys/uefi_keys.conf --uefi-enc sym_t234.key --no-flash --showlogs -p "-c bootloader/generic/cfg/flash_t234_qspi.xml" jetson-agx-orin-devkit internal
```
Note:
- the location of `flash_t234_qspi.xml` has changed compared to Jetpack 5.1.2
- The `--uefi-keys` and `--uefi-keys` arguments are added

```
sudo cp bootloader/eks_t234_sigheader_encrypt.img.signed ./tools/kernel_flash/images/internal/
```
# RootFs Setup
Due to the same issue described above, you need to replace the “-S 900Gib” argument with a size that will fit for your own hardware. This must be below the size set above, but also have enough room for all the other partitions on the disk. The device must be plugged in during this phase, because it will be queried by the scripts.

```
sudo ROOTFS_ENC=1 ./tools/kernel_flash/l4t_initrd_flash.sh --showlogs -u ./rsa.pem -v ./sbk.key --no-flash --external-device nvme0n1p1 -i ./sym2_t234.key --uefi-keys uefi_keys/uefi_keys.conf --uefi-enc sym_t234.key -c ./tools/kernel_flash/flash_l4t_t234_nvme_rootfs_enc.xml -S 900GiB --external-only --append --network usb0 jetson-agx-orin-devkit external
```
Note:
- Again, the `--uefi-keys` and `--uefi-keys` arguments are added compared to the Jetpack 5.1.2 instructions.

# Flash
This will flash the device. Although not necessary, you should have the UART plugged in to see progress from the devices perspective. If there are any issues, particularly if you need help with them, this will be very important.
```
sudo ./tools/kernel_flash/l4t_initrd_flash.sh --showlogs -u rsa.pem -v sbk.key --uefi-keys uefi_keys/uefi_keys.conf --uefi-enc sym_t234.key --network usb0 --flash-only
```
Note:
- Even if the flashing is successful, it's possible to run into problems on booting, and then you might also need to use the UART debugging (this is using the microUSB port). See Chris' guide for more instructions.

If all is well, then after the flashing succeeds, you'll see `“NVIDIA Corp. L4T (Linux for Tegra) running on Tegra”` listed from `lsusb`.

Good luck!