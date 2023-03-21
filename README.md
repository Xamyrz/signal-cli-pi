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