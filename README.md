# a603-flash-image

[a603-build-image](https://github.com/JunghanShin/a603-build-image)로 만든 플래시 이미지를 **Seeed A603 + Jetson Orin NX에 쓰는** 스크립트입니다. 이미지를 만들지는 않습니다.

보드를 USB로 연결하는 단계는 이것 하나입니다. 이미지는 미리 만들어 두었으므로 연결 시간이 짧아 USB write 에러가 나지 않습니다.

```text
a603-build-image.sh   (보드 없이, 오래 걸림)
        │ 작업 폴더/Linux_for_Tegra/tools/kernel_flash/images/
        ▼
a603-flash-image.sh   (보드 REC 모드, USB 연결)   ← 이 저장소
  보드가 initrd로 부팅 → SSD(nvme0n1) 찾기 → 파티션 생성 → NVMe와 QSPI 부트로더에 쓰기 → 결과 확인
```

| 항목 | 값 |
|---|---|
| 모듈 | Jetson Orin NX |
| 캐리어 보드 | Seeed A603 |
| 소프트웨어 | JetPack 6.2 (L4T 36.4.3) |
| 설치 위치 | NVMe SSD, 부트로더는 QSPI |

## 준비

| 항목 | 내용 |
|---|---|
| Linux | x86_64 Ubuntu 22.04 PC 또는 가상 머신. **이미지를 만든 환경과 같은 곳** (작업 폴더가 있어야 함) |
| 권한 | `sudo` (스크립트가 스스로 `sudo`로 다시 실행) |
| 이미지 | `a603-build-image.sh`로 만든 작업 폴더 |
| SSD | M.2 Key M 슬롯의 NVMe SSD (SATA M.2는 안 됨). NVIDIA 기본 파티션 구성은 64GB 이상 기준 |
| 케이블 | PC와 A603의 micro-USB 포트를 잇는 **데이터** 케이블 |

가상 머신에서 플래시하려면 NVIDIA 장치(Vendor ID `0955`)가 항상 가상 머신으로 자동 연결되게 설정해야 합니다. 방법은 [holybro-build-image README의 가상 머신 절](https://github.com/JunghanShin/holybro-build-image#가상-머신-virtualbox-vmware)과 같습니다. WSL2는 쓰지 않습니다.

## 설치

```bash
git clone https://github.com/JunghanShin/a603-flash-image.git
```

## 사용법

처음부터 첫 부팅까지 따라 하는 전체 순서는 [a603-build-image README의 빠른 시작](https://github.com/JunghanShin/a603-build-image#빠른-시작-처음부터-첫-부팅까지)에 있습니다. 아래는 쓰기 단계만 적은 것입니다.

1. 이미지를 먼저 만듭니다(a603-build-image README 참고).
   ```bash
   ./a603-build-image/a603-build-image.sh -w ~/a603-work
   ```
2. 보드 전원을 끄고 **NVMe SSD가 M.2 Key M 슬롯에 꽂혀 있는지** 확인합니다. SSD는 전원을 켜기 전에 꽂아야 합니다(핫플러그 안 됨).
3. 헤더 **W7의 3번(FORCE_RECOVERY)과 4번(GND)을 쇼트**합니다. A603에는 리커버리 버튼이 없습니다([Seeed 문서](https://wiki.seeedstudio.com/headless_setup_and_recovery_for_a603/)). 쇼트하기 전에 보드 실크 인쇄의 핀 번호를 확인합니다.
4. PC와 보드를 micro-USB로 연결하고 전원을 켭니다. 2초쯤 지나면 쇼트를 빼도 됩니다.
5. 가상 머신이면 VMware 메뉴 **VM → Removable Devices → NVIDIA APX → Connect**로 장치를 넘깁니다.
6. 복구 장치가 **480Mbps**로 잡혔는지 확인합니다. NVIDIA 장치(Orin NX 16GB `0955:7323`, 8GB `0955:7423`)가 `480M`으로 보여야 합니다. Seeed 문서의 `0955:7523`은 Orin Nano 모듈 값입니다.
   ```bash
   lsusb | grep 0955
   ```
   ```bash
   lsusb -t
   ```
7. 이미지를 만들 때와 **같은 작업 폴더**를 지정해 씁니다. 쓰는 도중 보드가 다시 부팅하며 USB 장치가 새로 잡히면 **가상 머신**에 연결합니다.
   ```bash
   ./a603-flash-image/a603-flash-image.sh -w ~/a603-work
   ```
8. `Flash finished`가 나오면 전원을 끄고 **W7 쇼트를 뺀 뒤** 전원을 켭니다.

## SSD는 어떻게 찾고 포맷하나

SSD는 Jetson의 PCIe에 붙어 있어서 PC에서는 보이지 않습니다. 그래서 PC 쪽에서 SSD를 찾거나 포맷하는 명령은 없고, `l4t_initrd_flash.sh`가 다음 순서로 처리합니다.

1. 보드를 작은 리눅스(initrd)로 부팅합니다.
2. **보드가 직접** `nvme0n1`을 찾고, `flash_l4t_t234_nvme.xml`대로 GPT 파티션을 만들어 이미지를 씁니다. 파티션이 없는 새 SSD도 그대로 씁니다.
3. 그 과정이 `--showlogs`로 PC 화면에 나옵니다.

이 스크립트는 끝난 뒤 로그로 결과를 확인합니다. SSD를 찾지 못했으면 원인을 출력하고 실패로 끝납니다. 다른 시스템이 들어 있던 SSD를 다시 쓸 때는 `--erase`로 SSD 전체와 QSPI를 먼저 지웁니다.

```bash
./a603-flash-image/a603-flash-image.sh -w ~/a603-work --erase
```

## 작업 폴더

- `-w 폴더`로 지정합니다.
- 지정하지 않으면 환경 변수 `WORK_DIR`, 그것도 없으면 **명령을 실행한 현재 폴더**를 씁니다.
- 작업 폴더에 `Linux_for_Tegra/tools/kernel_flash/images/external/system.img`가 없으면 아무것도 쓰지 않고 끝납니다.

## 옵션

| 옵션 | 설명 |
|---|---|
| `-w 폴더` | `a603-build-image.sh`가 쓴 작업 폴더 (기본: `WORK_DIR`, 없으면 현재 폴더) |
| `--erase` | 쓰기 전에 보드의 저장 장치 전체(SSD 전체, QSPI)를 지움 (`l4t_initrd_flash.sh --erase-all`) |
| `-h` | 사용법 출력 |

## 스크립트가 하는 일

1. 작업 폴더에 이미지가 있는지 확인합니다.
2. USB 버퍼를 키우고(`usbfs_memory_mb=2048`) USB 절전을 끕니다.
3. NVIDIA 복구 장치(`0955:7323`/`7423`/`7523`/`7623`)를 찾고 속도가 480Mbps인지 확인합니다.
4. `l4t_initrd_flash.sh --flash-only`로 씁니다. 보드가 SSD를 찾고 파티션을 만들어 씁니다. `--erase`면 먼저 저장 장치 전체를 지웁니다.
5. 결과를 확인합니다. 실패하면 SSD를 못 찾은 경우를 골라 안내하고, 성공했는데 성공 메시지가 로그에 없으면 실패로 처리합니다.

전체 출력은 작업 폴더의 `flash-image.log`, 보드 쪽 로그는 `Linux_for_Tegra/initrdlog/`에 남습니다.

## 문제 해결

| 증상 | 원인과 조치 |
|---|---|
| `Flash images are missing` | 작업 폴더가 다르거나 이미지를 아직 만들지 않았습니다. `-w`로 이미지를 만든 폴더를 지정합니다. |
| `Need a 480 Mbps recovery device` (`speed=missing`) | 보드가 리커버리 모드가 아닙니다. 전원을 끄고 W7 3번–4번을 쇼트한 뒤 다시 켭니다. 가상 머신이면 장치를 가상 머신에 연결했는지 확인합니다. 충전 전용 케이블이면 장치가 안 보입니다. |
| `Need a 480 Mbps recovery device` (`speed=5000` 등) | USB 3.x 속도로 잡혔습니다. PC의 다른 USB 포트(USB 2.0)나 USB 2.0 허브를 씁니다. |
| `The board did not find the NVMe SSD` | SSD가 덜 꽂혔거나 SATA M.2입니다. 전원을 끄고 NVMe SSD를 다시 꽂은 뒤 REC 모드로 켭니다. |
| `no success message was found` | 오류 없이 끝났지만 로그에서 성공을 확인하지 못했습니다. `flash-image.log`와 보드 로그를 확인한 뒤 부팅합니다. |
| 쓰는 도중 USB write 에러 | 짧은 데이터 케이블로 바꾸고 PC 본체 뒤쪽 포트에 직접 연결합니다. |

## 상태

- 2026-11-09 작성. **아직 실행하지 않았습니다.** 플래시 명령은 Seeed 문서(JP6.2)와 같고, holybro-flash-image에서 확인한 흐름을 그대로 씁니다.
- 첫 실행 때 확인할 것: `--flash-only`와 `--erase-all`을 함께 쓰는 동작, 성공·실패 메시지 문구(`Flash is successful` 등), `initrdlog/` 로그 위치.
