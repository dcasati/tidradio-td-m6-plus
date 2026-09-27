# TIDRADIO TD-M6 Plus boot sound removal

Disable the spoken `TIDRADIO` startup sound on the TD-M6 Plus by modifying the Android `system` partition through Qualcomm EDL.

Tested on:

```text
TIDRADIO TD-M6 Plus
Android 7.1.2
Qualcomm MSM8909
Build: TIDM6plusGEPXTTZ1.0.0
USB EDL: 05c6:9008
```

The radio exposes ADB and Qualcomm EDL. The stock boot animation binary directly plays:

```text
/system/media/boot.wav
```

Replacing that file with a silent WAV removes the startup voice without rooting Android.

## Requirements

Linux computer with:

```text
adb
git
python3
python3-venv
SoX
```

Install the required packages:

```bash
sudo apt install \
  adb \
  git \
  python3-full \
  python3-venv \
  libusb-1.0-0-dev \
  liblzma-dev \
  sox
```

Enable USB debugging on the radio and verify ADB:

```bash
adb devices -l
```

Example:

```text
0 device usb:3-1.3 product:msm8909 model:TIDM6plus device:msm8909
```

## Confirm the boot sound

The stock file is:

```text
/system/media/boot.wav
```

It is a standard PCM WAV:

```text
RIFF
16-bit
stereo
44100 Hz
```

Boot logs confirm that the vendor boot animation process plays it directly:

```text
BootAnimation: barnett bootMusic
BootAnimation: barnett index 7
BootAnimation: barnett playing /system/media/boot.wav
```

Disabling Android apps such as `com.android.poweronalert` does not disable this sound.

## Install EDL

Clone `bkerler/edl`:

```bash
git clone https://github.com/bkerler/edl.git
cd edl
git submodule update --init --recursive
```

Create a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If ModemManager is running:

```bash
sudo systemctl stop ModemManager
```

## Enter Qualcomm EDL

From Android:

```bash
adb -s 0 reboot edl
```

Verify:

```bash
lsusb
```

Expected:

```text
05c6:9008 Qualcomm, Inc. Gobi Wireless Modem (QDL mode)
```

## Verify Firehose access

Run:

```bash
python edl.py printgpt
```

On this radio:

```text
HWID:         0x009600e100000000
CPU detected: MSM8909
MemoryName:   eMMC
```

The generic MSM8909 Firehose loader was accepted:

```text
Possibly unfused device detected, so any loader should be fine...
Loader successfully uploaded.
```

Important partitions:

```text
boot      32 MB
recovery  32 MB
system    ~1.5 GB
persist   32 MB
```

## Back up the radio

Do this before writing anything.

```bash
mkdir -p ~/tidm6plus-backup
```

Back up GPT metadata:

```bash
python edl.py gpt ~/tidm6plus-backup --genxml
```

Back up the important partitions:

```bash
python edl.py r boot ~/tidm6plus-backup/boot.img
python edl.py r recovery ~/tidm6plus-backup/recovery.img
python edl.py r system ~/tidm6plus-backup/system.img
python edl.py r aboot ~/tidm6plus-backup/aboot.img
python edl.py r abootbak ~/tidm6plus-backup/abootbak.img
python edl.py r devinfo ~/tidm6plus-backup/devinfo.img
python edl.py r config ~/tidm6plus-backup/config.img
```

Also preserve the radio-specific modem and persistent data:

```bash
python edl.py r modemst1 ~/tidm6plus-backup/modemst1.img
python edl.py r modemst2 ~/tidm6plus-backup/modemst2.img
python edl.py r fsg ~/tidm6plus-backup/fsg.img
python edl.py r persist ~/tidm6plus-backup/persist.img
```

Optional full backup excluding userdata:

```bash
python edl.py rl ~/tidm6plus-backup/all --skip=userdata --genxml
```

Generate checksums:

```bash
sha256sum ~/tidm6plus-backup/*.img \
  > ~/tidm6plus-backup/SHA256SUMS
```

Keep this backup.

## Inspect system.img

The system partition is normal ext4:

```bash
file ~/tidm6plus-backup/system.img
```

Expected:

```text
Linux rev 1.0 ext4 filesystem data
volume name "system"
```

Mount it read-only:

```bash
sudo mkdir -p /mnt/tidm6-system

sudo mount -o loop,ro \
  ~/tidm6plus-backup/system.img \
  /mnt/tidm6-system
```

Check the media files:

```bash
ls -lh /mnt/tidm6-system/media/
file /mnt/tidm6-system/media/boot.wav
```

Stock contents include:

```text
boot.wav
bootanimation.zip
shutdown.wav
shutdownanimation.zip
```

Unmount:

```bash
sudo umount /mnt/tidm6-system
```

## Create a silent boot sound

Create one second of silence using the same audio format as the factory file:

```bash
sox -n \
  -r 44100 \
  -c 2 \
  -b 16 \
  ~/silent-boot.wav \
  trim 0 1
```

Verify:

```bash
file ~/silent-boot.wav
```

Expected:

```text
RIFF (little-endian) data
WAVE audio
Microsoft PCM
16 bit
stereo
44100 Hz
```

## Create the modified system image

Always work from the original backup:

```bash
cp \
  ~/tidm6plus-backup/system.img \
  ~/tidm6plus-system-silent.img
```

Mount the copy read-write:

```bash
sudo mount -o loop,rw \
  ~/tidm6plus-system-silent.img \
  /mnt/tidm6-system
```

Replace the startup sound:

```bash
sudo cp \
  ~/silent-boot.wav \
  /mnt/tidm6-system/media/boot.wav

sudo chown root:root \
  /mnt/tidm6-system/media/boot.wav

sudo chmod 644 \
  /mnt/tidm6-system/media/boot.wav
```

Flush and unmount:

```bash
sync
sudo umount /mnt/tidm6-system
```

Check the filesystem:

```bash
sudo e2fsck -f ~/tidm6plus-system-silent.img
```

If asked:

```text
Padding at end of inode bitmap is not set. Fix<y>?
```

answer:

```text
y
```

Run it again:

```bash
sudo e2fsck -f ~/tidm6plus-system-silent.img
```

The second pass should complete without repairs.

## Verify the image

Confirm the modified image has the same exact size as the original:

```bash
stat -c '%s %n' \
  ~/tidm6plus-backup/system.img \
  ~/tidm6plus-system-silent.img
```

On this radio both are:

```text
1610613760
```

Mount the modified image read-only:

```bash
sudo mount -o loop,ro \
  ~/tidm6plus-system-silent.img \
  /mnt/tidm6-system
```

Verify:

```bash
file /mnt/tidm6-system/media/boot.wav
ls -lh /mnt/tidm6-system/media/boot.wav
```

Unmount:

```bash
sudo umount /mnt/tidm6-system
```

## Flash the modified system partition

Return the radio to EDL:

```bash
adb -s 0 reboot edl
```

Verify:

```bash
lsusb | grep 05c6:9008
```

Activate the EDL environment:

```bash
cd ~/edl
source .venv/bin/activate
```

Write only the `system` partition:

```bash
python edl.py w system \
  ~/tidm6plus-system-silent.img
```

Example successful completion:

```text
Progress: |██████████| 100.0%
Wrote /home/user/tidm6plus-system-silent.img to sector 592992.
```

Do not disconnect USB or power during the write.

## Verify the flash

Read the partition back:

```bash
python edl.py r system \
  ~/tidm6plus-system-silent-verify.img
```

Compare hashes:

```bash
sha256sum \
  ~/tidm6plus-system-silent.img \
  ~/tidm6plus-system-silent-verify.img
```

The hashes must match exactly.

Example:

```text
4057b5c0b76815df7ccc2c06d28e60033c13c9e2daf522f65dc751cfb7a63567
4057b5c0b76815df7ccc2c06d28e60033c13c9e2daf522f65dc751cfb7a63567
```

Do not reboot until the read-back verification matches.

## Reboot

ADB is unavailable while the radio is in EDL mode.

Reset it through Firehose:

```bash
python edl.py reset
```

The radio should boot normally without the spoken `TIDRADIO` startup sound.

## Restore the factory system image

If required, enter EDL again and restore the original image:

```bash
python edl.py w system \
  ~/tidm6plus-backup/system.img
```

Read it back and verify the hash before rebooting.

## Notes

The TD-M6 Plus has a very limited Qualcomm fastboot implementation. Commands such as:

```bash
fastboot reboot
```

may return:

```text
unknown command
```

EDL is more useful on this device.

The tested unit accepted the generic MSM8909 Firehose loader and reported itself as possibly unfused.

Do not use recovery images, boot images or partition layouts from unrelated MSM8909 devices.

Do not erase or overwrite:

```text
modemst1
modemst2
fsg
persist
sbl1
tz
rpm
```

unless you know exactly why you are doing it.

The startup voice is not controlled by a normal Android application. It is played directly by the vendor-modified `bootanimation` binary from:

```text
/system/media/boot.wav
```

Replacing the file with silence is preferable to deleting it. The boot animation process still finds a valid audio file and follows its normal playback path.
