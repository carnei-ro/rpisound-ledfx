# Raspberry PI Sound System with AirPlay, Spotify, Snapcast, and LEDFX

```bash
sudo apt update
sudo apt install --no-install-recommends -y \
      alsa-utils autoconf automake avahi-daemon build-essential cmake curl git \
      libasound2-dev libavahi-client-dev libavcodec-dev libavformat-dev libavutil-dev \
      libboost-program-options-dev libboost-system-dev libboost-test-dev libboost-thread-dev \
      libconfig-dev libexpat1-dev libflac-dev libgcrypt-dev libglib2.0-dev libmosquitto-dev \
      libopus-dev libplist-dev libpopt-dev libpulse-dev libsndfile1-dev libsodium-dev libsoxr-dev \
      libssl-dev libtool libvorbis-dev libvorbisidec-dev pulseaudio pulsemixer uuid-dev vim xxd

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

# testing if it is still working
# sudo reboot
# speaker-test -c2 --test=wav -w /usr/share/sounds/alsa/Front_Center.wav
# if it is not working, try to run 'mv /home/pi/.asoundrc /home/pi/.asoundrc-bkp'

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

curl -sL https://dtcooper.github.io/raspotify/install.sh | sh
sudo journalctl -xeu raspotify

curl -sSL https://install.ledfx.app | bash

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
cd ..

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

echo '# Start the client, used only by the init.d script
START_SNAPCLIENT=true

# Additional command line options that will be passed to snapclient
# note that user/group should be configured in the init.d script or the systemd unit file
# For a list of available options, invoke "snapclient --help" 
SNAPCLIENT_OPTS=""' | sudo tee /etc/default/snapclient

sudo useradd -r -s /usr/sbin/nologin -d /var/lib/snapclient -c "Snapcast Client" snapclient
sudo usermod -a -G audio snapclient
sudo mkdir /var/lib/snapclient
sudo chown snapclient:snapclient /var/lib/snapclient

###
sudo systemctl disable --now shairport-sync
sudo sed -i -e '/\[stream\]/a\source = airplay:///shairport-sync?name=AirPlay&devicename=RPiSound-AirPlay' /etc/snapserver.conf

sudo systemctl disable --now raspotify
sudo sed -i -e '/\[stream\]/a\source = spotify:///librespot?name=Spotify&devicename=RPiSound-Spotify' /etc/snapserver.conf

sudo sed -i -e 's,name=default,name=pipe,g' /etc/snapserver.conf
sudo sed -i -e '/name=pipe/a\source = meta:///AirPlay/Spotify/pipe?name=default' /etc/snapserver.conf


####

echo 'load-module module-native-protocol-unix auth-anonymous=1 socket=/tmp/pulse-socket' | sudo tee -a /etc/pulse/default.pa

sudo sed -i -e '/ExecStart=/iExecStartPre=mkdir -p /var/lib/snapclient/.config/pulse' /etc/systemd/system/snapclient.service
sudo sed -i -e '/ExecStart=/iExecStartPre=echo "default-server = unix:/tmp/pulse-socket" > /var/lib/snapclient/.config/pulse/client.conf' /etc/systemd/system/snapclient.service

sudo systemctl daemon-reload
sudo sed -i -e 's/SNAPCLIENT_OPTS=""/SNAPCLIENT_OPTS="--player pulse"/g' /etc/default/snapclient

sudo systemctl --global disable pulseaudio.socket pulseaudio.service


echo '[Unit]
Description=Sound Service
Requires=pulseaudio.socket

[Service]
ExecStart=/usr/bin/pulseaudio --daemonize=no --log-target=journal --system
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

sudo systemctl daemon-reload
sudo systemctl --system enable pulseaudio.socket
sudo systemctl --system enable pulseaudio.service
sudo systemctl --system start pulseaudio.socket
sudo systemctl --system start pulseaudio.service

sudo cp /etc/pulse/system.pa /etc/pulse/system.pa-bkp
sudo cp /etc/pulse/default.pa /etc/pulse/system.pa

sudo usermod -a -G audio pulse
sudo usermod -a -G bluetooth pulse
sudo usermod -a -G pulse-access pi
sudo usermod -a -G pulse-access snapclient
sudo usermod -a -G pulse-access snapserver
sudo usermod -a -G pulse-access root

mkdir -p /home/pi/.config/pulse
echo "default-server = unix:/tmp/pulse-socket" > /home/pi/.config/pulse/client.conf

sudo su - snapserver -s /bin/bash -c 'mkdir -p /var/lib/snapserver/.config/pulse; echo "default-server = unix:/tmp/pulse-socket" > /var/lib/snapserver/.config/pulse/client.conf'

sudo sed -i -e 's,After=.*,& snapserver.service pulseaudio.service,g' /etc/systemd/system/snapclient.service

sudo sed -i -e 's,After=.*,& snapserver.service pulseaudio.service snapclient.service,g' /etc/systemd/system/ledfx.service

sudo systemctl daemon-reload

sudo systemctl start snapserver
sudo systemctl enable snapserver
sudo systemctl start snapclient
sudo systemctl enable snapclient

sudo reboot
```

## Bluetooth

This part is not fully tested - some bugs occur not letting the raspberry to shutdown - needs to pull the plug. (or wait for some minutes)

```bash
sudo apt-get install pulseaudio pulseaudio-module-bluetooth bluez-tools
sudo usermod -a -G bluetooth pi
sudo sed -i -e 's/#DiscoverableTimeout = 0/DiscoverableTimeout = 0/g' /etc/bluetooth/main.conf
sudo sed -i -e 's/#IdleTimeout.*/IdleTimeout = 0/g' /etc/bluetooth/input.conf
sudo sed -i -e 's/#Class = 0.*/Class = 0x41C/g' /etc/bluetooth/main.conf # Do we really need it? https://www.bluetooth.com/specifications/assigned-numbers/baseband
sudo sed -i -e 's/StopWhenUnneeded=yes/StopWhenUnneeded=yes/g' /lib/systemd/system/bluetooth.service
sudo systemctl daemon-reload
sudo systemctl restart bluetooth
echo '[Unit]
Description=Bluetooth Auth Agent
After=bluetooth.service
PartOf=bluetooth.service

[Service]
Type=simple
ExecStart=/usr/bin/bt-agent -c NoInputNoOutput
ExecStartPost=hciconfig hci0 piscan

[Install]
WantedBy=bluetooth.target' | sudo tee /etc/systemd/system/bt-agent.service
sudo systemctl enable bt-agent
sudo systemctl start bt-agent

echo '.ifexists module-switch-on-connect.so
load-module module-switch-on-connect
.endif' | sudo tee -a /etc/pulse/system.pa

sudo sed -i -e 's/^load-module module-suspend-on-idle/#load-module module-suspend-on-idle/g' /etc/pulse/system.pa

sudo usermod -a -G bluetooth pulse

sudo systemctl restart dbus
sudo systemctl restart pulseaudio
sudo systemctl status pulseaudio

sudo sed -i -e 's/ExecStart.*/& --exit-idle-time=-1/g' /etc/systemd/system/pulseaudio.service
sudo systemctl daemon-reload
sudo systemctl restart pulseaudio
sudo systemctl status pulseaudio

sudo usermod -a -G bluetooth snapserver
sudo systemctl restart snapserver
sudo reboot
```
