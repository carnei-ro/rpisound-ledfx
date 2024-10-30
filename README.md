# Raspberry PI Sound System with AirPlay, Spotify, Snapcast, and LEDFX

Installed on 2024-10-22-raspios-bullseye-arm64.img

## Problems

- [ ] Snapcast doesn't work with Bluetooth (Try to sink the Bluetooth audio to the Snapcast server)
- [ ] If a device disconnects from the Bluetooth, need to do `bluetoothctl devices` and `bluetoothctl remove <MAC>` to remove the device
- [ ] It takes too long to shutdown the Raspberry PI (Something related to the bluetooth)
- [ ] Volume buttons for bluetooth devices don't work

## Installing

```bash
sudo apt update
sudo apt install --no-install-recommends -y \
      alsa-utils autoconf automake avahi-daemon bluez-tools build-essential cmake curl gcc git \
      libasound2-dev libatlas3-base libavahi-client-dev libavcodec-dev libavformat-dev libavformat58 \
      libavutil-dev libboost-program-options-dev libboost-system-dev libboost-test-dev libboost-thread-dev \
      libconfig-dev libexpat1-dev libflac-dev libgcrypt-dev libglib2.0-dev libmosquitto-dev libopus-dev \
      libplist-dev libpopt-dev libpulse-dev libsndfile1-dev libsodium-dev libsoxr-dev libssl-dev libtool \
      libvorbis-dev libvorbisidec-dev portaudio19-dev pulseaudio pulseaudio-module-bluetooth pulsemixer \
      python3-pip uuid-dev vim xxd

## Dependencies for AirPlay
git clone https://github.com/mikebrady/nqptp.git
cd nqptp
autoreconf -fi
./configure --with-systemd-startup
make -j $(nproc)
sudo make install
sudo systemctl enable nqptp
sudo systemctl start nqptp
cd ..

git clone https://github.com/mikebrady/alac.git
cd alac
autoreconf -fi
./configure
make -j $(nproc)
sudo make install
cd ..

## AirPlay (shairport-sync)
git clone https://github.com/mikebrady/shairport-sync.git
cd shairport-sync
autoreconf -fi
./configure --sysconfdir=/etc \
  --with-airplay-2 --with-alsa --with-apple-alac --with-avahi \
  --with-convolution --with-dbus-interface --with-dummy \
  --with-metadata --with-mpris-interface --with-mqtt-client \
  --with-pipe --with-soxr --with-ssl=openssl --with-stdout --with-systemd
make -j $(nproc)
sudo make install
cd ..

## Raspotify
curl -sL https://dtcooper.github.io/raspotify/install.sh | sh
sudo journalctl -xeu raspotify

## LedFX
python3 -m pip install --upgrade pip wheel setuptools
python3 -m pip install ledfx
sed -i -e '/Not a compatible WLED brand/,+1s/raise/# raise/g' $HOME/.local/lib/python3.*/site-packages/ledfx/utils.py
sudo usermod -a -G audio pi
echo "[Unit]
Description=LedFx Music Visualizer
After=network.target sound.target snapserver.service pulseaudio.service snapclient.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=pi
Group=audio
ExecStart=/usr/bin/python3 /home/pi/.local/bin/ledfx
Environment=XDG_RUNTIME_DIR=/run/user/"$UID"

[Install]
WantedBy=multi-user.target
" | sudo tee /etc/systemd/system/ledfx.service

## SnapCast
git clone https://github.com/badaix/snapcast.git
cd snapcast
sudo cmake .
sudo make -j $(nproc)
sudo make install
sed -i -e 's,/usr/bin,/usr/local/bin,g' ./extras/package/debian/snapclient.service ./extras/package/debian/snapserver.service 
sed -i -e 's,/usr/share,/usr/local/share,g' ./server/etc/snapserver.conf
sudo cp ./extras/package/debian/snapclient.service /etc/systemd/system/
sudo cp ./extras/package/debian/snapserver.service /etc/systemd/system/
sudo cp ./server/etc/snapserver.conf /etc/
sudo systemctl disable --now shairport-sync
sudo sed -i -e '/\[stream\]/a\source = airplay:///shairport-sync?name=AirPlay&devicename=RPiSound-AirPlay' /etc/snapserver.conf
sudo systemctl disable --now raspotify
sudo sed -i -e '/\[stream\]/a\source = spotify:///librespot?name=Spotify&devicename=RPiSound-Spotify' /etc/snapserver.conf
sudo sed -i -e 's,name=default,name=pipe,g' /etc/snapserver.conf
sudo sed -i -e '/name=pipe/a\source = meta:///AirPlay/Spotify/pipe?name=default' /etc/snapserver.conf
sudo sed -i -e '/ExecStart=/iExecStartPre=mkdir -p /var/lib/snapclient/.config/pulse' /etc/systemd/system/snapclient.service
sudo sed -i -e '/ExecStart=/iExecStartPre=echo "default-server = unix:/tmp/pulse-socket" > /var/lib/snapclient/.config/pulse/client.conf' /etc/systemd/system/snapclient.service
sudo sed -i -e 's,After=.*,& snapserver.service pulseaudio.service,g' /etc/systemd/system/snapclient.service

wget https://github.com/badaix/snapweb/releases/download/v0.8.0/snapweb_0.8.0-1_all.deb
sudo sed -i -e 's;/usr/local/share/snapserver/snapweb;/usr/share/snapweb;g' /etc/snapserver.conf
sudo dpkg -i snapweb_0.8.0-1_all.deb

echo '# Start the server, used only by the init.d script
START_SNAPSERVER=true

# Additional command line options that will be passed to snapserver
# note that user/group should be configured in the init.d script or the systemd unit file
# For a list of available options, invoke "snapserver --help" 
SNAPSERVER_OPTS=""' | sudo tee /etc/default/snapserver

sudo useradd -r -s /usr/sbin/nologin -d /var/lib/snapserver -c "Snapcast Server" snapserver
sudo usermod -a -G audio snapserver
sudo mkdir /var/lib/snapserver
sudo chown snapserver:snapserver /var/lib/snapserver

sudo su - snapserver -s /bin/bash -c 'mkdir -p /var/lib/snapserver/.config/pulse; echo "default-server = unix:/tmp/pulse-socket" > /var/lib/snapserver/.config/pulse/client.conf'

echo '# Start the client, used only by the init.d script
START_SNAPCLIENT=true

# Additional command line options that will be passed to snapclient
# note that user/group should be configured in the init.d script or the systemd unit file
# For a list of available options, invoke "snapclient --help" 
SNAPCLIENT_OPTS="--player pulse"' | sudo tee /etc/default/snapclient

sudo useradd -r -s /usr/sbin/nologin -d /var/lib/snapclient -c "Snapcast Client" snapclient
sudo usermod -a -G audio snapclient
sudo mkdir /var/lib/snapclient
sudo chown snapclient:snapclient /var/lib/snapclient

cd ..

## PulseAudio
sudo cp /etc/pulse/default.pa /etc/pulse/system.pa
echo 'load-module module-native-protocol-unix auth-anonymous=1 socket=/tmp/pulse-socket' | sudo tee -a /etc/pulse/system.pa

echo '.ifexists module-switch-on-connect.so
load-module module-switch-on-connect
.endif' | sudo tee -a /etc/pulse/system.pa

sudo sed -i -e 's/^load-module module-suspend-on-idle/#load-module module-suspend-on-idle/g' /etc/pulse/system.pa

sudo systemctl --global disable pulseaudio.socket pulseaudio.service

echo '[Unit]
Description=Sound Service
Requires=pulseaudio.socket

[Service]
ExecStart=/usr/bin/pulseaudio --daemonize=no --log-target=journal --system --exit-idle-time=-1
Restart=on-failure
SystemCallArchitectures=native
Type=notify
UMask=0077

[Install]
Also=pulseaudio.socket
WantedBy=multi-user.target' | sudo tee /etc/systemd/system/pulseaudio.service

echo '[Unit]
Description=Sound System

[Socket]
Priority=6
Backlog=5
ListenStream=%t/pulse/native

[Install]
WantedBy=sockets.target' | sudo tee /etc/systemd/system/pulseaudio.socket

mkdir -p /home/pi/.config/pulse
echo "default-server = unix:/tmp/pulse-socket" > /home/pi/.config/pulse/client.conf

## Bluetooth
sudo sed -i -e 's/#DiscoverableTimeout = 0/DiscoverableTimeout = 0/g' /etc/bluetooth/main.conf
sudo sed -i -e 's/#IdleTimeout.*/IdleTimeout = 0/g' /etc/bluetooth/input.conf
sudo sed -i -e 's/#Class = 0.*/Class = 0x41C/g' /etc/bluetooth/main.conf
sudo sed -i -e 's/StopWhenUnneeded=yes/StopWhenUnneeded=yes/g' /lib/systemd/system/bluetooth.service
sudo sed -i -e 's,/bluetooth/bluetoothd,& --noplugin=sap,g' /lib/systemd/system/bluetooth.service

echo '[Unit]
Description=Bluetooth Auth Agent
After=bluetooth.service
PartOf=bluetooth.service

[Service]
Type=simple
ExecStart=/usr/bin/bt-agent -c NoInputNoOutput
ExecStartPost=/usr/bin/hciconfig hci0 piscan

[Install]
WantedBy=bluetooth.target' | sudo tee /etc/systemd/system/bt-agent.service


## Configuring Operating System
sudo hciconfig hci0 piscan

sudo usermod -a -G audio pulse
sudo usermod -a -G bluetooth pulse
sudo usermod -a -G pulse-access pi
sudo usermod -a -G pulse-access snapclient
sudo usermod -a -G pulse-access snapserver
sudo usermod -a -G pulse-access root
sudo usermod -a -G bluetooth pi
sudo usermod -a -G bluetooth pulse
sudo usermod -a -G bluetooth snapserver
sudo usermod -a -G snapserver pi


sudo systemctl daemon-reload

sudo systemctl enable ledfx
sudo systemctl enable snapserver
sudo systemctl enable snapclient
sudo systemctl --system enable pulseaudio.socket
sudo systemctl --system enable pulseaudio.service
sudo systemctl enable bt-agent


sudo reboot
```

## Configuring

Now open the browser at `http://<ip-of-your-raspberry-pi>:8888` and configure the LED strip at `Devices`. At `Settings` change the `Audio Device` to **pulse**.

At `http://<ip-of-your-raspberry-pi>:1780` configure the Pi to listen to the "default" Snapcast stream.

Try to pair your phone via Bluetooth and play some music. TODO: Bluetooth is not "multi-room" yet, need to configure snapcast server to listen to bluetooth (or pipe the bluetooth to somewhere).
