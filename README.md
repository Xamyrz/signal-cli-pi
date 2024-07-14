# signal-cli for Raspberry Pi guide

## Compilation
1. Download the latest linux tar signal-cli from [Asamk repository](https://github.com/AsamK/signal-cli/releases/latest)
2. Extract the tar
3. navigate to `signal-cli-x.x.x/lib` and find `libsignal-client-x.x.x.jar`
4. Replace `libsignal_jni.so` in `libsignal-client-x.x.x.jar` with a corresponding `libsignal_jni.so` version from [exquo repository](https://github.com/exquo/signal-libs-build/releases/). aarch64 for 64bit raspbian and armv7 for 32 bit raspbian
5. Archive `signal-cli-x.x.x` folder back into tar

## Installation

```ssh
sudo tar xf signal-cli-x.x.x-Linux.tar -C /opt
sudo ln -sf /opt/signal-cli-x.x.x/bin/signal-cli /usr/local/bin/
```

## Compilation/Installation bash script

```ssh
#!/bin/bash

echo 'Downloading Signal-CLI'
VER=$(curl --silent -qI  https://github.com/AsamK/signal-cli/releases/latest | awk -F '/' '/^location/ {print  substr($NF, 1, length($NF)-1)}'); \
wget https://github.com/AsamK/signal-cli/releases//download/$VER/signal-cli-${VER#v}.tar.gz

echo 'creating directory signal-${VER#v}'
mkdir signal-${VER#v}

echo 'extracting signal-cli-${VER#v}.tar.gz to signal-${VER#v} directory'
tar xf signal-cli-${VER#v}.tar.gz -C signal-${VER#v}/

echo 'Getting libsignal-client version'
LIBVER=$(find signal-${VER#v}/signal-cli-${VER#v}/lib -name 'libsignal-client-*.jar' | grep -Eo '[0-9]+\.[0-9]+\.[0.9]')

echo downlaoding cooresponding libsignal_v$LIBVER
wget https://github.com/exquo/signal-libs-build/releases/download/libsignal_v$LIBVER/libsignal_jni.so-v$LIBVER-aarch64-unknown-linux-gnu.tar.gz

echo extracting libsignal_jni.so-v$LIBVER-aarch64-unknown-linux-gnu.tar.gz to signal-${VER#v} directory
tar xf libsignal_jni.so-v$LIBVER-aarch64-unknown-linux-gnu.tar.gz -C signal-${VER#v}/

echo updating libsignal_jni.so file in libsignal-client-$LIBVER.jar
jar -uf signal-${VER#v}/signal-cli-${VER#v}/lib/libsignal-client-$LIBVER.jar -C signal-${VER#v} libsignal_jni.so

echo creating signal-cli-${VER#v}.tar from signal-${VER#v}/signal-cli-${VER#v} directory
tar -C signal-${VER#v} -cvf signal-cli-${VER#v}.tar signal-cli-${VER#v}

echo extracting signal-cli-${VER#v}.tar to /opt dir
sudo tar xf signal-cli-${VER#v}.tar -C /opt

echo linking /opt/signal-cli-${VER#v}/bin/signal-cli to /usr/local/bin/ directory
sudo ln -sf /opt/signal-cli-${VER#v}/bin/signal-cli /usr/local/bin/


echo removing signal-${VER#v} directory
rm -rf signal-${VER#v}

echo removing signal-cli-${VER#v}.tar.gz
rm -rf signal-cli-${VER#v}.tar.gz

echo removing libsignal_jni.so-v$LIBVER-aarch64-unknown-linux-gnu.tar.gz
rm -rf libsignal_jni.so-v$LIBVER-aarch64-unknown-linux-gnu.tar.gz

echo signal-CLI version $VER

```

## Usage

For a complete usage overview please read
the [man page](https://github.com/AsamK/signal-cli/blob/master/man/signal-cli.1.adoc) and
the [wiki](https://github.com/AsamK/signal-cli/wiki).

Important: The ACCOUNT is your phone number in international format and must include the country calling code. Hence it
should start with a "+" sign. (See [Wikipedia](https://en.wikipedia.org/wiki/List_of_country_calling_codes) for a list
of all country codes.)

* Register a number (with SMS verification)

        signal-cli -a ACCOUNT register

  You can register Signal using a landline number. In this case you can skip SMS verification process and jump directly
  to the voice call verification by adding the `--voice` switch at the end of above register command.

  Registering may require solving a CAPTCHA
  challenge: [Registration with captcha](https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha)

* Verify the number using the code received via SMS or voice, optionally add `--pin PIN_CODE` if you've added a pin code
  to your account

        signal-cli -a ACCOUNT verify CODE

* Send a message

        signal-cli -a ACCOUNT send -m "This is a message" RECIPIENT

* Pipe the message content from another process.

        uname -a | signal-cli -a ACCOUNT send --message-from-stdin RECIPIENT

* Receive messages

        signal-cli -a ACCOUNT receive

**Hint**: The Signal protocol expects that incoming messages are regularly received (using `daemon` or `receive`
command). This is required for the encryption to work efficiently and for getting updates to groups, expiration timer
and other features.

## Storage

The password and cryptographic keys are created when registering and stored in the current users home directory:

        $XDG_DATA_HOME/signal-cli/data/
        $HOME/.local/share/signal-cli/data/

## Dbus setup
Run this script with sudo
```python
#!/usr/bin/env python
#coding: utf-8
import os
version = "x.x.x"
number = "+48xxxxxxxxx"

os.system("cd /tmp ; git clone https://github.com/AsamK/signal-cli.git")
os.system("cd /tmp/signal-cli/data ; cp org.asamk.Signal.conf /etc/dbus-1/system.d/ ; cp org.asamk.Signal.service /usr/share/dbus-1/system-services/ ; cp signal-cli.service /etc/systemd/system/")
os.system("""sed -i -e 's|policy user="signal-cli"|policy user="pi"|' /etc/dbus-1/system.d/org.asamk.Signal.conf""")
os.system("""sed -i -e 's|User=signal-cli|User=pi|' /etc/systemd/system/signal-cli.service""")
os.system("""sed -i -e "s|ExecStart=%dir%/bin/signal-cli --config /var/lib/signal-cli daemon --system|ExecStart=/opt/signal-cli-""" + version + """//bin/signal-cli -u """ + number + """ daemon --system|" /etc/systemd/system/signal-cli.service""")
```

etc/systemd/system/signal-cli.service should look like below

```nano
[Unit]
Description=Send secure messages to Signal clients
Requires=dbus.socket
After=dbus.socket
Wants=network-online.target
After=network-online.target

[Service]
Type=dbus
Environment="SIGNAL_CLI_OPTS=-Xms2m"
ExecStart=/opt/signal-cli-0.11.7//bin/signal-cli -u +48xxxxxxxxx daemon --system
User=pi
BusName=org.asamk.Signal
# JVM always exits with 143 in reaction to SIGTERM signal
SuccessExitStatus=143

[Install]
Alias=dbus-org.asamk.Signal.service
```


try sending a message using dbus

```ssh
dbus-send --system --type=method_call --print-reply --dest="org.asamk.Signal" /org/asamk/Signal org.asamk.Signal.sendMessage string:'hey' array:string: string:'+353xxxxxxxxx'
```

if it doesn't work, reboot the raspberry pi

## sample python code
```python
def msgRcv (timestamp, source, groupID, message, attachments):
    if not groupID:
        print ("Message", message, "received from", source)
    else:
        print ("Message", message, "received in group", signal.getGroupName (groupID))
    return

from pydbus import SystemBus
from gi.repository import GLib

bus = SystemBus()
loop = GLib.MainLoop()

signal = bus.get('org.asamk.Signal')

signal.sendMessage("hi", [], "+353xxxxxxxx")
signal.onMessageReceived = msgRcv
loop.run()
```
