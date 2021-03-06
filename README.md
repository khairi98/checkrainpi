# CheckRainPi -- a rainwater gauge data collection script

This is an easy script for collecting results off the serial port and
sending them out to Amazon SimpleDB.

## How it works

The script sends the memory dump command to a rainwater gauge attached
to the serial port.

It collects the results and stores them in a file.

It then posts just the latest results to a database in the Amazon
cloud.

The results can be retrieved at any time by accessing the Amazon
SimpleDB.

This script is ideal for running on a Raspberry Pi.


## Installation

First get python3 and git.

    sudo apt-get install python python-virtualenv git

The following commands will create a python virtualenv and install the
script into it.

    git clone https://github.com/rvl/checkrainpi.git
    cd checkrainpi
    ./install.sh


## Configuration

Find the file `raingauge.conf` and edit it to your tastes.

In particular, you will need to go to http://aws.amazon.com/ and sign
up for a free account, so you can enter your access key into the
config file.


## Invocation

    ./venv/bin/checkrain -v --conf=raingauge.conf


## Getting results

This will print out all the samples collected so far.

    ./venv/bin/getrain --conf=raingauge.conf


## Running automatically

Put the following in a file `/etc/cron.d/checkrainpi`.

It will run the script every day at 4PM.

    SHELL=/bin/sh
    PATH=/bin:/usr/bin

    0 16 * * *  pi /home/pi/checkrainpi/venv/bin/checkrain --conf=/home/pi/checkrainpi/raingauge.conf

## Complete setup starting from clean Raspbian image

Do the following steps as the `pi` user. Don't use `sudo` unless it
says.

### Initial raspberry pi setup

Do the following steps with this command.

    sudo raspi-config

1. Change password to a good long secret password
2. Change language to en-AU.UTF-8
3. Change timezone to Australia/Perth
4. Change keyboard layout to 104 key English (US)
5. Enable the SSH server

Install vim editor because nano sucks.

    apt-get update && apt-get install vim

### Mobile Internet connection (Telstra)

Telstra 4G dongle config is quite easy. The attached dongle appears as
a normal ethernet interface, using the ``cdc_ether`` driver.

First check for USB device. The attached dongle will appear as one of
the following two vendor strings.

    lsusb
    Bus 001 Device 007: ID 19d2:1405 ZTE WCDMA Technologies MSM
    Bus 001 Device 005: ID 12d1:14db Huawei Technologies Co., Ltd.

The ZTE is well supported by Raspbian and appears as the ``usb0``
interface. The Huawei is supported by newer versions of Linux than
Raspbian, so needs an updated config file. When it works, it comes up
as ``eth1`` (``eth0`` is the Raspberry Pi's ethernet port).

Get the updated file:

    wget https://sources.debian.net/data/main/u/usb-modeswitch-data/20150115-1/40-usb_modeswitch.rules
    sudo chown root:root 40-usb_modeswitch.rules
    sudo mv 40-usb_modeswitch.rules /lib/udev/rules.d/40-usb_modeswitch.rules
    sudo service udev reload

Edit the Debian networking file.

    sudo vim /etc/network/interfaces
    
Add the following lines:

    allow-hotplug usb0
    auto usb0
    iface usb0 inet dhcp

    allow-hotplug eth1
    auto eth1
    iface eth1 inet dhcp

Bring up connection (choose interface based on attached dongle).

    sudo ifup usb0
    sudo ifup eth1

### SSH Reverse Proxy

For this, we use the `autossh` software.

    sudo apt-get install autossh

Running as the pi user, create a passwordless ssh keypair.

     ssh-keygen

Now, you will need a gateway host somewhere on the Internet that you
have access SSH to. It is also nice to have the setting
`GatewayPorts yes` in `/etc/ssh/sshd_config`, but not required.

On the gateway host, append the created file
`/home/pi/.ssh/id_rsa.pub` to its `authorized_keys` file.

Add the following to `/etc/rc.local` (before exit 0 line). Replace
`myuser@mygateway.host` with the correct username and hostname.
   
    autossh -N -f -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -o "TCPKeepAlive=yes" -o "ServerAliveInterval=90" -i /home/pi/.ssh/id_rsa -R "*:6666:localhost:22" myuser@mygateway.host &

Test the SSH connection and accept the host key:

    sudo ssh myuser@mygateway.host

Then run it the init script.

    chmod +x /etc/rc.local
    /etc/rc.local

Check the reverse proxy by trying to login to the raspberry pi from
another machine.

    ssh -p 6666 -l pi mygateway.host

The above assumes that the gateway has `GatewayPorts yes`. If not,
then you need to login with two hops.

    ssh myuser@mygateway.host
    ssh -p 6666 -l pi localhost

### Create your own gateway host

If you don't have access to a gateway host, one can be created under
the Amazon EC2 12 month free usage tier. You would then need to use an
dynamic DNS client to maintain a constant hostname. Amazon have
instructions in their EC2 documentation.

### USB Serial setup

The device will be `/dev/ttyUSB0`. Just check that the user has
permission to open it.

    sudo adduser pi dialout

### Raingauge setup

    sudo mkdir /data
    sudo chown pi:pi /data
    
    sudo apt-get install python python-virtualenv git

    cd
    git clone https://github.com/rvl/checkrainpi.git
    cd checkrainpi
    ./install.sh

    cp raingauge.conf site.conf

Edit `site.conf` according to your settings.

### Automatic running with cron

This will run the script every day at 7AM. It will also auto-update
the checkrainpi script every week. Run the crontab editor for the pi
user:

    crontab -e

Put the following lines down the bottom:

     0 7 * * * /home/pi/checkrainpi/venv/bin/checkrain --conf=/home/pi/checkrainpi/site.conf
     0 6 * * * cd /home/pi/checkrainpi && git pull -q
     */5 * * * * /home/pi/checkrainpi/scripts/connect4g.sh
     0 5 * * * /home/pi/checkrainpi/scripts/reboot-huawei.sh
