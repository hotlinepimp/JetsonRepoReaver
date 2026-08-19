# JetsonRepoReaver

r2d2 — Jetson Orin Nano Super + Waveshare UGV Rover PT

Build notes and operational reference for a Waveshare UGV Rover PT running an NVIDIA Jetson Orin Nano Super 8GB on JetPack 7.2.1.

Why this document exists: the vendor documentation for this rover assumes JetPack 6 / Ubuntu 22.04 and a Waveshare-supplied disk image. This build uses JetPack 7.2.1 / Ubuntu 24.04 with a self-installed OS. Everything below records what actually worked, and where reality diverged from the docs.

Status
Component	State
Jetson OS	✅ JetPack 7.2.1 / L4T 39.2.1 / Ubuntu 24.04.4
Boot device	✅ NVMe SSD (/dev/nvme0n1p1, 466 GB)
System firmware	✅ 39.2.1 (QSPI, A/B slots healthy)
CUDA stack	✅ CUDA 13.2, cuDNN 9.20, TensorRT 10.16, VPI 4.1, OpenCV 4.8
Power mode	✅ MAXN_SUPER available
Network	✅ Onboard wifi, SSH
Jetson ↔ ESP32 serial	✅ Verified — /dev/ttyTHS1 @ 115200
Motion control	✅ Verified — wheels respond to JSON commands
ROS 2	⬜ Not started
Camera / LiDAR	⬜ Not started
Hardware

Compute

NVIDIA Jetson Orin Nano Super Developer Kit, 8GB module (P3767-0005)
NVMe SSD in M.2 Key-M 2280 slot — OS boots from here, not microSD
Wifi/BT module pre-populated in M.2 Key-E slot (the kit's spare wifi card is not needed)

Rover — Waveshare UGV Rover PT (6-wheel, 4WD)

Waveshare General Driver for Robots board (ESP32 sub-controller)
2-axis pan-tilt with ST3215 servos (30 kg·cm)
3S UPS module, 3 × 18650 cells (not included with kit)
D500 360° LiDAR, OAK-D Lite depth camera, 5MP 160° USB camera

Power

Supply	Output	Goes to
NVIDIA brick	19V	Jetson barrel jack (bench use)
Waveshare RXW-0698	12.6V 2A — battery charger	Rover UPS charging port only
3S 18650 pack	~9.0–12.6V	Rover, and Jetson via base lead

⚠️ The Jetson's DC jack accepts 9–20V, max 3.5A. The 12.6V unit is a battery charger — it belongs in the rover's charging port, never directly on the Jetson. The 40-pin header's 5V rail cannot power an Orin Nano (it's intended for Raspberry Pi and the old 5V Jetson Nano).

Quick start
bash
ssh r2d2@192.168.1.75          # or r2d2.local when mDNS resolves

Check health:

bash
sudo nvpmodel -q                       # power mode
sudo tegrastats                        # live stats (Ctrl+C to stop)
jtop                                   # nicer dashboard, if the git build works
sudo nvbootctrl dump-slots-info        # firmware version and A/B slot state

Test the rover link (read-only, nothing moves):

bash
python3 ~/esp32_link_test.py
The serial link

The Jetson talks to the ESP32 over the 40-pin header hardware UART, not USB.

Setting	Value
Device	/dev/ttyTHS1
Baud	115200
Pins	40-pin header 8 (TX) / 10 (RX)
Kernel device	3100000.serial

Deviation from vendor docs. Waveshare's code uses /dev/ttyTHS0, which does not exist on the Orin Nano — this board exposes ttyTHS1 and ttyTHS2 only. This is the same breakage reported in ugv_jetson issue #3 for JetPack 6. Change every /dev/ttyTHS0 to /dev/ttyTHS1.

Helpful: the debug console lives on ttyTCU0 (see /proc/cmdline), so no nvgetty service is squatting on the UART. Nothing needs disabling.

Your user must be in the dialout group:

bash
sudo usermod -aG dialout $USER    # then log out and back in
Protocol

Newline-delimited JSON, both directions.

Send — motion (L/R are left/right speed, roughly -0.5 … 0.5):

json
{"T":1,"L":0.2,"R":0.2}
{"T":1,"L":0,"R":0}

Receive — telemetry (T:1001, streamed continuously):

Field	Meaning
L, R	commanded wheel speeds
ax,ay,az	accelerometer (az ≈ 8450 when level)
gx,gy,gz	gyroscope (≈ 0 at rest)
mx,my,mz	magnetometer
odl, odr	wheel odometry counts
v	battery voltage ×100 — 1058 = 10.58V

v is your fuel gauge. 3S range is 1260 (full) down to about 900 (empty).

Gotchas encountered

Recorded because every one of these cost real time.

Installation
JetPack 7.2 has no SD card image for the Orin Nano. It ships a unified installer ISO (jetsoninstaller-r39.2.0-*-arm64.iso) that the Jetson boots from a USB stick. No host PC, no recovery jumper, no SDK Manager.
The installer skips the account setup when you install to NVMe, leaving a login prompt with no valid credentials (reported here). In this build the real cause was a staged firmware update that hadn't run yet — after the QSPI capsule update (36.4.3 → 39.2.x) completed, the first-boot wizard appeared normally.
The installed console does not render on a 4K display. You get a blinking cursor and nothing else. The UEFI screens render fine. Workaround: use the USB serial gadget (/dev/ttyACM0 @ 115200 on the host) or a 1080p display.
SSH fails out of the box — the image ships without host keys, so sshd -t fails and the service never starts:
bash
  sudo ssh-keygen -A && sudo systemctl restart ssh
Flashing media
dd progress output on some hosts reports bytes buffered, not written — showing impossible speeds (3+ GB/s) then appearing to hang during the real flush. pv is fooled the same way. The only honest source is the kernel:
bash
  watch -n5 'awk "{print \$7*512/1048576 \" MB\"}" /sys/block/sdX/stat'
Wait for the command to actually exit. Interrupting during conv=fsync leaves a partial image.
Post-install
MAXN_SUPER was missing after installing 7.2 — only 15W and 7W existed. A full apt upgrade replaced the stale nvpmodel configs and restored 15W / 25W / MAXN_SUPER.
jetson-stats does not support L4T 39.2.1 in the PyPI release. Install from git; the JetPack name still shows as "NOT DETECTED" (cosmetic).
bash
  sudo pip3 install -U git+https://github.com/rbonghi/jetson_stats.git --break-system-packages
  sudo jtop --install-service
No swap by default. On an 8GB board, add some:
bash
  sudo fallocate -l 16G /swapfile && sudo chmod 600 /swapfile
  sudo mkswap /swapfile && sudo swapon /swapfile
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
No RTC battery. The clock reads 1970 until NTP syncs, which can break TLS and build tooling in the first moments after boot.
The desktop costs ~1 GB. For headless use:
bash
  sudo systemctl set-default multi-user.target
Power
MAXN_SUPER draws well over 25W. On battery, prefer 15W (sudo nvpmodel -m 0). A boot loop under battery power is starvation, not a software fault.
The 12.6V 2A charger provides ~25W total — for the Jetson and motors and servos. The 18650s do the real work.
Roadmap
 Charge the pack (watch v in telemetry; target ~1260)
 ROS 2 — start with Waveshare's Humble container, passing the device through: --device=/dev/ttyTHS1
 Decide: stay on Humble-in-Docker, or port their nodes to native Jazzy
 Pan-tilt servo control
 USB camera → ROS 2 image topic
 D500 LiDAR → SLAM
 OAK-D Lite depth
 Autonomous navigation (Nav2)

Run rviz2 and other visualisation on the desktop machine rather than the Jetson — ROS 2 is distributed by design, and the Jetson's 8GB is better spent on inference.

References
Orin Nano Devkit User Guide
JetPack 6.x firmware update path
Carrier board specification — DC jack 9–20V, J14 pinout
Waveshare UGV Rover ROS2 wiki
Jetson Python chassis motion control
ugv_jetson · ugv_ws · ugv_base_general
Appendix — J14 button header (Orin Nano devkit)

1×12 single-row, 2.54 mm pitch, near the fan connector and power jack.

Pin	Signal		Pin	Signal
1	PC_LED−		7	GND
2	PC_LED+		8	SYS_RESET*
3	UART2_RXD (debug)		9	GND
4	UART2_TXD (debug)		10	FORCE_RECOVERY*
5	AC OK		11	GND
6	Auto power-on disable		12	SLEEP/WAKE*

Recovery mode: jumper pin 10 to pin 9 (or 11 — both are GND) with power disconnected, then apply power. Confirm on the host with lsusb | grep 0955:7523. Do not bridge pins 8–9 — that holds the board in reset.

Not needed for the ISO installer route, but required for SDK Manager flashing.
