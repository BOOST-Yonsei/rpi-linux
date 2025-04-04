# CAS4101 BOOST

컴종설 BOOST팀 라즈베리파이4B를 타겟한 RPi OS 커널 repository

## 개발환경

devcontainer로 우분투 개발환경을 세팅할 수 있다.
VSCode 등 devcontainer를 지원하는 IDE를 사용하면 편하다.

## 이 프로젝트 전용 경로

- .devcontainer/: devcontainer 개발환경 설정
- projboost/: 커널 소스가 아닌 파일 저장용
- mnt/: 빌드 후 커널 이미지를 구울 때 사용할 mount point

## USB UART 콘솔 사용

[picocom](https://github.com/npat-efault/picocom)을 추천한다.

라즈베리파이가 켜지기 이전에 먼저 USB를 연결하고 picocom으로 콘솔에 접속한 상태에서 부팅을 시작하면 된다.

```bash
ls /dev/tty* | grep -i usb # -> USB 시리얼 콘솔의 장치경로 확인
sudo picocom -b 11520 -f -h /dev/<ttyUSB0>
# -g <로그파일 경로> 옵션을 추가하면 콘솔 데이터를 파일로 저장할 수 있다
```

## 빌드 과정

[공식자료](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compiled-build)

기본 Raspberry Pi OS를 먼저 부팅 저장장치에 구워야 한다. 아래의 빌드 과정은 커널만 빌드한다.

### 커널 빌드 기본설정 (옵션 1)

```bash
KERNEL=kernel8
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
```

### 커널 빌드 상세설정 (옵션 2)

```bash
KERNEL=kernel8
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
# 방향키로 메뉴 이동
# ESC ESC로 상위 메뉴로 돌아가기
# H로 도움말 표시
# 마지막에 저장 후 종료하면 .config 파일에 저장된다
# -> 설정 공유는 .config 파일을 공유하고 다음 단계로 넘어가면 된다
```

### 커널 빌드

위의 옵션 1,2 중 하나를 진행한 다음에 다음 명령어 실행

```bash
make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
# -j 옵션은 logical core의 1.5배 권장
```

### 커널 모듈 굽기

`lsblk + sudo fdisk -l /dev/<device>`로 boot 파티션 (FAT32)과 root 파티션 (EXT4)를 찾아 mount한다.

```bash
# 예시
# sdc
# |--sdc1: FAT32 --> mnt/boot
# +--sdc2: EXT4  --> mnt/root
mkdir -p mnt/boot
mkdir -p mnt/root
sudo mount /dev/sdc1 mnt/boot
sudo mount /dev/sdc2 mnt/root
```

mount가 정상적으로 된 것을 확인한 다음 커널 이미지를 굽는다.

```bash
sudo env PATH=$PATH make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/root modules_install
# -j 옵션은 logical core의 1.5배 권장
```

### 커널 이미지 굽기

위에서 mount한 상태로 이어서 진행한다.

```bash
sudo cp mnt/boot/kernel8.img mnt/boot/kernel8-backup.img
# ^ 첫 명령어는 기존 커널 이미지를 백업하는 용도라 필요없다면 생략가능
sudo cp arch/arm64/boot/Image mnt/boot/kernel8.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/boot/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/boot/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/boot/overlays/
```

## 부팅 옵션 설정

부팅 과정의 로그를 USB 시리얼 콘솔로 출력하기 위해 부트로더의 설정을 변경한다.

`projboost/boot/config.txt`에 있는 변경사항들을 `mnt/boot/config.txt`에 반영한다.

추가로 디버깅용으로 `mnt/boot/cmdline.txt`에서 `plymouth.ignore-serial-consoles`를 제거하면 시리얼 콘솔에도 커널 로그가 출력된다. 여기에서 더 많은 커널 로그가 필요하면 `quiet` 옵션까지 제거하면 된다.

## 장치 제거

```bash
sudo umount mnt/boot
sudo umount mnt/root
```

# Original README.md

Linux kernel
============

There are several guides for kernel developers and users. These guides can
be rendered in a number of formats, like HTML and PDF. Please read
Documentation/admin-guide/README.rst first.

In order to build the documentation, use ``make htmldocs`` or
``make pdfdocs``.  The formatted documentation can also be read online at:

    https://www.kernel.org/doc/html/latest/

There are various text files in the Documentation/ subdirectory,
several of them using the Restructured Text markup notation.

Please read the Documentation/process/changes.rst file, as it contains the
requirements for building and running the kernel, and information about
the problems which may result by upgrading your kernel.

Build status for rpi-6.1.y:
[![Pi kernel build tests](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml/badge.svg?branch=rpi-6.1.y)](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml)
[![dtoverlaycheck](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml/badge.svg?branch=rpi-6.1.y)](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml)

Build status for rpi-6.6.y:
[![Pi kernel build tests](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml/badge.svg?branch=rpi-6.6.y)](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml)
[![dtoverlaycheck](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml/badge.svg?branch=rpi-6.6.y)](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml)

Build status for rpi-6.12.y:
[![Pi kernel build tests](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml/badge.svg?branch=rpi-6.12.y)](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml)
[![dtoverlaycheck](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml/badge.svg?branch=rpi-6.12.y)](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml)
