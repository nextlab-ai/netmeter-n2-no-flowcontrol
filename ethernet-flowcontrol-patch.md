# Raspberry Pi 5 이더넷 흐름제어(Flow Control) 비활성화

## 개요

Raspberry Pi 5의 이더넷 인터페이스(`eth0`)에서 TX pause frame(흐름제어)이 기본적으로
활성화되어 있어 네트워크 성능 측정에 영향을 줍니다. `ethtool`로는 제어가 불가능하며,
커널 드라이버를 직접 수정하고 재빌드해야 합니다.

---

## 시스템 환경

| 항목 | 값 |
|------|----|
| 장비 | Raspberry Pi 5 (BCM2712 / RP1) |
| OS | Raspberry Pi OS Bookworm (Debian 12) |
| 원본 커널 | `6.12.47+rpt-rpi-2712` |
| 적용 후 커널 | `6.12.87-v8-16k-rpi-2712-nopause` |
| 드라이버 | `macb` (Cadence GEM, `raspberrypi,rp1-gem`) |
| PHY 칩 | Broadcom BCM54213PE |
| 인터페이스 | `eth0` |

---

## 문제 분석

### 증상

```
# dmesg (수정 전)
macb 1f00100000.ethernet eth0: Link is Up - 1Gbps/Full - flow control tx

# ethtool eth0 (수정 전)
Supported pause frame use: Transmit-only
Advertised pause frame use: Transmit-only
Link partner advertised pause frame use: Symmetric Receive-only
```

Pi가 상대방에게 TX pause 기능을 광고하고, 802.3 협상 결과 TX pause가 활성화됩니다.
즉, Pi의 RX 버퍼가 찰 때 pause frame을 전송하여 상대방의 전송을 일시 정지시킵니다.

### ethtool로 제어 불가능한 이유

```bash
$ sudo ethtool -A eth0 tx off rx off
netlink error: Operation not supported
```

`macb` 드라이버가 `set_pauseparam` ethtool operation을 구현하지 않아
런타임에서 흐름제어를 변경할 수 없습니다.

### mii-tool로 제어 불가능한 이유

```bash
$ sudo mii-tool -A 1000baseT-FD eth0
# 재협상 후에도 flow control tx 유지
```

`macb` 드라이버가 phylink 서브시스템을 사용합니다. phylink가 PHY 레지스터를
직접 관리하므로, `mii-tool`로 PHY 레지스터를 수동으로 변경해도 링크 이벤트
발생 시 즉시 덮어씌워집니다.

### 근본 원인

파일: `drivers/net/ethernet/cadence/macb_main.c`, 함수: `macb_mii_probe()`

```c
// 문제 코드 (line ~983)
bp->phylink_config.mac_capabilities = MAC_ASYM_PAUSE |
    MAC_10 | MAC_100;
```

`MAC_ASYM_PAUSE`를 MAC 기능으로 선언하면 phylink가 PHY 협상 시
Asym_Pause 비트를 자동으로 광고합니다. 상대방과의 협상 결과로 TX pause가
활성화됩니다.

### CONFIG_MACB=y 문제

```bash
$ grep CONFIG_MACB /boot/config-6.12.47+rpt-rpi-2712
CONFIG_MACB=y
```

`macb`가 커널에 정적으로 내장(built-in)되어 있어 `.ko` 모듈 파일만
교체하는 방법을 쓸 수 없습니다. 커널 전체를 재빌드해야 합니다.

---

## 해결 방법

### 수정 내용 (단 1줄)

파일: `drivers/net/ethernet/cadence/macb_main.c`

```diff
- bp->phylink_config.mac_capabilities = MAC_ASYM_PAUSE |
-     MAC_10 | MAC_100;
+ bp->phylink_config.mac_capabilities =
+     MAC_10 | MAC_100;
```

`MAC_ASYM_PAUSE`를 제거하면 PHY가 pause 기능을 광고하지 않으므로
autoneg 협상에서 흐름제어가 비활성화됩니다.

---

## 패치 적용 절차

### 빌드 환경

| 항목 | 사양 |
|------|------|
| 빌드 머신 | `minmax@100.102.64.67` |
| CPU | AMD Ryzen 7 7800X3D (16코어) |
| OS | Ubuntu 24.04 LTS (WSL2) |
| 크로스컴파일 타겟 | `aarch64-linux-gnu` |

### 1단계: 빌드 도구 설치

```bash
# 빌드 머신에서 실행
sudo apt-get install -y \
    git bc bison flex \
    gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu \
    debhelper cpio libssl-dev libelf-dev
```

### 2단계: 커널 소스 클론

```bash
git clone --depth=1 --branch rpi-6.12.y \
    https://github.com/raspberrypi/linux.git ~/linux-rpi
cd ~/linux-rpi
```

> **주의:** `rpi-6.12.y` 브랜치는 최신 커밋을 가리키므로,
> 클론 시점에 따라 커널 버전이 달라집니다 (현재 기록 기준: 6.12.87).

### 3단계: Pi의 커널 설정 복사

```bash
ssh hsqmmanager@192.168.12.6 'cat /boot/config-$(uname -r)' > .config
```

### 4단계: 드라이버 수정

```bash
sed -i \
  's/bp->phylink_config.mac_capabilities = MAC_ASYM_PAUSE |$/bp->phylink_config.mac_capabilities =/' \
  drivers/net/ethernet/cadence/macb_main.c

# 수정 확인
grep -A2 "mac_capabilities =" drivers/net/ethernet/cadence/macb_main.c | head -4
```

예상 출력:
```
bp->phylink_config.mac_capabilities =
    MAC_10 | MAC_100;
```

### 5단계: 빌드

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export DEB_BUILD_PROFILES="pkg.linux-upstream.nokernelheaders"

make olddefconfig
make -j$(nproc) bindeb-pkg LOCALVERSION=-rpi-2712-nopause
```

빌드 완료 후 상위 디렉토리에 생성되는 파일:
```
~/linux-image-<VERSION>-rpi-2712-nopause_<VER>_arm64.deb   # 약 30-35MB
~/linux-libc-dev_<VER>_arm64.deb                           # 불필요
```

### 6단계: Pi에 전송 및 설치

커널에는 두 종류의 드라이버가 있습니다:
- **built-in (`=y`)**: 커널 이미지 자체에 포함 (macb 이더넷, USB, PCIe 등)
- **모듈 (`=m`)**: `/lib/modules/$(uname -r)/` 폴더의 `.ko` 파일 (Wi-Fi, Bluetooth 등)

`dpkg -i`가 모듈 폴더를 먼저 만들고, 그 다음 커널 이미지를 교체해야 합니다.
**순서를 바꾸면 재부팅 후 Wi-Fi, Bluetooth 등 모듈이 로드되지 않습니다.**

```bash
# 1. 빌드 머신 → Pi로 전송
scp ~/linux-image-*-nopause_*_arm64.deb hsqmmanager@192.168.12.6:/tmp/

# 2. Pi에서 설치 (순서 중요)
ssh hsqmmanager@192.168.12.6 '
  # 기존 커널 백업
  sudo cp /boot/firmware/kernel_2712.img /boot/firmware/kernel_2712.img.bak

  # [1단계] 모듈 설치: /lib/modules/6.12.87-v8-16k-rpi-2712-nopause/ 생성
  sudo dpkg -i /tmp/linux-image-*-nopause_*_arm64.deb

  # [2단계] 커널 이미지 교체: 부트로더가 새 커널을 사용하도록
  sudo cp /boot/vmlinuz-*-nopause /boot/firmware/kernel_2712.img
'
```

### 7단계: 재부팅 및 검증

```bash
ssh hsqmmanager@192.168.12.6 'sudo reboot'

# 부팅 완료까지 대기 (보통 25-35초, 최대 60초)
sleep 10
until ssh -o ConnectTimeout=5 hsqmmanager@192.168.12.6 'uname -r' 2>/dev/null; do
    echo "대기 중..."; sleep 5
done

# 흐름제어 확인
ssh hsqmmanager@192.168.12.6 '
  uname -r
  dmesg | grep "flow control"
  /usr/sbin/ethtool eth0 | grep -i pause
'
```

**성공 기준:**
```
# dmesg
macb ... eth0: Link is Up - 1Gbps/Full - flow control off

# ethtool
Supported pause frame use: No
Advertised pause frame use: No
```

---

## 보관 파일 목록

작업 경로(`rasberrypi/`)에 보관된 파일들입니다.

| 파일 | 크기 | 용도 |
|------|------|------|
| `linux-image-6.12.87-v8-16k-rpi-2712-nopause_..._arm64.deb` | 32MB | 커널 + 모듈 전체 패키지. Pi에 `dpkg -i`로 설치 |
| `kernel_2712-nopause.img` | 9.6MB | deb에서 추출한 커널 이미지. dpkg 설치 완료 후 `/boot/firmware/kernel_2712.img`에 복사 |
| `0001-macb-disable-flow-control.patch` | 1.1KB | 소스 패치. 새 버전으로 재빌드 시 `git apply`로 적용 |
| `ethernet-flowcontrol-patch.md` | 이 문서 | 분석, 설치 절차, 위험성 기록 |

> **주의:** `kernel_2712-nopause.img`만 단독으로 복사하는 것은 충분하지 않습니다.
> 반드시 `deb` 설치로 모듈을 먼저 설치한 후 커널 이미지를 교체해야 합니다.

---

## 소요 시간

| 단계 | 소요 시간 |
|------|-----------|
| 빌드 도구 설치 | ~2분 |
| 커널 소스 클론 | ~3-5분 (네트워크 속도에 따라) |
| 빌드 (Ryzen 7 7800X3D 16코어) | **약 12-15분** |
| Pi로 전송 및 dpkg 설치 | ~3분 |
| 재부팅 및 확인 | ~2분 |
| **총 소요 시간** | **약 25-30분** |

> Pi에서 직접 빌드 시 (4코어): 약 1.5~3시간 소요

---

## 위험성 및 주의사항

### 1. 부팅 실패 위험 (낮음)

커널 이미지가 손상되거나 Pi 5와 호환되지 않을 경우 부팅에 실패할 수 있습니다.

**완화 방법:**
- 설치 전 기존 커널을 반드시 백업합니다.
  ```bash
  sudo cp /boot/firmware/kernel_2712.img /boot/firmware/kernel_2712.img.bak
  ```
- 부팅 실패 시 SD 카드를 PC에 연결하여 백업 파일을 복원합니다.
  ```bash
  # PC에서 SD 카드의 /boot/firmware 파티션에 접근 후
  cp kernel_2712.img.bak kernel_2712.img
  ```

### 2. 커널 버전 업그레이드 시 패치 소실 (중간)

`apt upgrade`로 RPi 공식 커널이 업데이트되면 `/boot/firmware/kernel_2712.img`가
공식 커널로 덮어씌워져 흐름제어가 다시 활성화됩니다.

**완화 방법:**
```bash
# 커널 패키지 업데이트를 차단 (단기 조치)
sudo apt-mark hold linux-image-rpi-2712

# 해제 방법
sudo apt-mark unhold linux-image-rpi-2712
```

장기적으로는 새 공식 커널 출시 시 동일한 빌드 과정을 반복하거나,
빌드 스크립트를 보관하여 재사용합니다.

### 3. 공식 커널과의 버전 차이 (낮음)

현재 설치된 커널(`6.12.87`)이 RPi 공식 커널(`6.12.47`)보다 높은 버전입니다.
`rpi-6.12.y` 브랜치는 RPi 재단이 유지보수하는 공식 브랜치이므로 안정성
면에서 문제가 없으나, 일부 RPi 전용 기능이 다르게 동작할 가능성이 있습니다.

### 4. 모듈 호환성 (낮음)

DKMS(Dynamic Kernel Module Support)로 설치된 서드파티 커널 모듈
(예: 외부 Wi-Fi 드라이버, VPN 모듈 등)은 새 커널 버전에 맞게 재컴파일이
필요할 수 있습니다.

```bash
# 설치된 DKMS 모듈 확인
dkms status
```

### 5. DTB(Device Tree Blob) 호환성 (낮음)

현재 빌드에서는 기존 DTB 파일을 그대로 사용합니다. 커널 버전 변경에 따라
DTB와 커널 간 불일치가 발생할 수 있으나, macb 드라이버 수정은 DTB와 무관하여
실질적인 문제 가능성은 낮습니다.

---

## 롤백 방법

```bash
# Pi에서 실행
sudo cp /boot/firmware/kernel_2712.img.bak /boot/firmware/kernel_2712.img
sudo reboot
```

재부팅 후 `uname -r`이 `6.12.47+rpt-rpi-2712`로 돌아오면 롤백 성공입니다.

롤백 시에는 기존 모듈(`/lib/modules/6.12.47+rpt-rpi-2712/`)이 그대로 유지되어 있으므로
Wi-Fi, Bluetooth 등 모든 기능이 즉시 복원됩니다.

---

## 대안 방법 검토

흐름제어를 드라이버 수정 없이 완화하는 방법들과 한계를 정리합니다.

### 방법 1: RX 링 버퍼 증가

NIC 하드웨어가 수신 패킷을 임시 보관하는 링 버퍼를 최대값으로 늘립니다.
버퍼가 차는 빈도를 줄여 pause frame 발생을 억제합니다.

```bash
# 현재 설정 확인
sudo ethtool -g eth0
# Pre-set maximums RX: 4096 / Current RX: 512 (RPi5 기본값)

# 즉시 적용 (재부팅 후 초기화)
sudo ethtool -G eth0 rx 4096 tx 4096
```

**영구 적용 (systemd 서비스):**
```bash
sudo tee /etc/systemd/system/eth-ring-buffer.service <<EOF
[Unit]
Description=Set eth0 ring buffer size
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -G eth0 rx 4096 tx 4096
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now eth-ring-buffer.service
```

### 방법 2: 커널 소켓 버퍼 튜닝

소켓 수신 버퍼가 작으면 커널이 패킷을 빨리 소화하지 못해
링 버퍼 포화를 가속시킵니다. 소켓 버퍼를 128MB로 확장합니다.

```bash
sudo tee /etc/sysctl.d/99-network-performance.conf <<EOF
net.core.rmem_max     = 134217728
net.core.rmem_default = 134217728
net.core.wmem_max     = 134217728
net.core.wmem_default = 134217728
net.ipv4.tcp_rmem = 4096 131072 134217728
net.ipv4.tcp_wmem = 4096 131072 134217728
net.ipv4.tcp_moderate_rcvbuf = 1
EOF
sudo sysctl -p /etc/sysctl.d/99-network-performance.conf
```

### 버퍼 구조와 흐름

```
[NIC 하드웨어]              [커널]             [애플리케이션]
  RX 링 버퍼      →    소켓 수신 버퍼    →     iperf3 등
  (최대 4096개)        (최대 128MB)
       ↑                     ↑
  여기가 차면           여기가 차면
  pause frame 전송      링 버퍼가 차기 시작
```

두 버퍼를 모두 늘려야 효과가 있습니다.
링 버퍼만 늘려도 소켓 버퍼가 작으면 역류가 발생합니다.

### 900Mbps 지속 측정에서의 한계

1Gbps 링크에서 900Mbps 지속 부하 시 패킷 처리 요구량:

```
900Mbps ÷ (1500byte × 8bit) = 약 75,000 패킷/초
패킷 1개당 처리 허용 시간 = 약 13 마이크로초

링 버퍼 최대(4096개) 기준 여유 시간:
4096개 ÷ 75,000 PPS = 약 55ms
```

55ms 안에 CPU가 버퍼를 지속적으로 소화해야 합니다.
스케줄러 지연, 인터럽트 처리, 메모리 복사 등으로
**근거리 최고속 부하에서는 결국 버퍼가 포화됩니다.**

### 방법별 효과 비교

| 방법 | 순간 burst | 500Mbps 지속 | 900Mbps 지속 |
|------|-----------|-------------|-------------|
| RX 링 버퍼 증가 | ✅ 효과 있음 | 🔶 어느 정도 | ❌ 결국 포화 |
| 소켓 버퍼 튜닝 | ✅ 효과 있음 | 🔶 어느 정도 | ❌ 링 버퍼 포화 못 막음 |
| 상대 장비 flow control off | ✅ | ✅ | ✅ (상대 장비 설정 필요) |
| **macb 드라이버 수정 (적용됨)** | ✅ | ✅ | ✅ 완전 차단 |

**결론:** 1Gbps 링크에서 900Mbps 수준의 지속 성능 측정이 목적이라면
버퍼 튜닝만으로는 신뢰할 수 없습니다. 드라이버 수정이 유일한 근본 해결책입니다.

---

## 검증 결과

```
# dmesg (수정 후)
macb 1f00100000.ethernet eth0: Link is Up - 1Gbps/Full - flow control off

# ethtool eth0 (수정 후)
Supported pause frame use: No
Advertised pause frame use: No
Link partner advertised pause frame use: Symmetric Receive-only
```

상대방(`Link partner`)이 pause를 광고하더라도 Pi가 pause를 광고하지 않으므로
802.3 협상 결과 양방향 흐름제어가 모두 비활성화됩니다. 상대방이 pause frame을
강제 전송하더라도 Pi MAC의 PAE(Pause Accept Enable) 비트가 꺼져 있어 무시됩니다.
