---
title: "Emulating RP on Pc"
date: 2021-02-22T10:51:44Z
draft: true
categories: ["IoT"]
---

# Setup
One of the usecases possible with emulating RPi on a pc is to create RP images. Although Docker can be used to create images, Packer is a very interesting alternative. By using packer, the configuration for Docker and Packer can be harmonized. This gives flexibility for deploying the images,
because configuration/image creation can also be used without Docker.
It makes 'infrastructure as code' possible. Start with an empty Pi and apply the configuration programmaticaly.
By combining this with the network boot and NFS feature, custom RP images can be defined and instaniated, just like custom AMI images can be defined and deployed on AWS.

## Installing qemu on arch linux

Install packages qemu and qemu-arch-extra (https://wiki.archlinux.org/index.php/QEMU)
Clone git repro from https://github.com/dhruvvyas90/qemu-rpi-kernel
Download the raspbian image from http://downloads.raspberrypi.org/
The image that was downloaded (buster 2019-09-26 raspbian-lite) was modified such that /etc/fstab refers to /dev/sda1 and /dev/sda3

The process is also described in https://azeria-labs.com/emulate-raspberry-pi-with-qemu/

The command shown below resulted in a proper booting RPi, with network connection, and accessible via ssh on the host on port 5022

$ qemu-system-arm -kernel qemu-rpi-kernel/kernel-qemu-4.19.50-buster -cpu arm1176 -m 256 -M versatilepb -append 'root=/dev/sda2 rootfstype=ext4 rw panic=1' -hda 2019-09-26-raspbian-buster-lite.img -nic user,hostfwd=tcp::5022-:22 -dtb qemu-rpi-kernel/versatile-pb-buster.dtb -serial stdio

Interesting for further investigation
Mapping serial port of emulator to serial ports on host: https://stackoverflow.com/questions/39373236/redirect-multiple-uarts-in-qemu

Related
- packer(-arm)
- docker
- ansible
- kubernetes
