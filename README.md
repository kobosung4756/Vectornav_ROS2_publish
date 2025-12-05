# Vectornav_ROS2_publish
Gemini 3로 작성되었습니다.

**참고:** https://github.com/dawonn/vectornav/tree/ros2

## ⚠️ 개발환경
**HW:** Vectornav-100T

**환경:** Ubuntu 22.04 + ROS2 Humble

## 💻 하드웨어 설정 (USB 권한 및 Latency 설정)
IMU 데이터는 고속으로 들어오기 때문에 리눅스의 시리얼 포트 설정이 매우 중요합니다.

**▶ 사용자 권한 설정 (USB 접근 허용):**

터미널을 열고 다음 명령어를 입력하여 현재 사용자를 dialout 그룹에 추가합니다. (재부팅 후 적용됨)
```bash
sudo usermod -aG dialout $USER
```

**▶ Latency Timer 최적화 (UDEV Rule 작성):**

FTDI 칩셋을 사용하는 USB-Serial 변환기는 데이터를 버퍼링하므로, IMU 같은 실시간 센서에서는 딜레이가 발생할 수 있습니다. 이를 최소화하기 위해 `udev` 규칙을 설정합니다.

```bash
# 1. udev rule 파일 생성
sudo nano /etc/udev/rules.d/99-vectornav.rules
```

파일 안에 다음 내용을 붙여넣으세요. (VN-100 장치를 자동으로 인식하고 지연 시간을 1ms로 줄여줍니다.)
```bash
# VectorNav VN-100 USB setting
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", MODE="0666", SYMLINK+="vectornav", ATTR{latency_timer}="1"
```
- 저장 후 종료 (`Ctrl+O`, `Enter`, `Ctrl+X`)

설정 적용: 
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

## 📁 워크스페이스 생성 및 드라이버 다운로드
**▶ 워크스페이스 생성 및 소스 다운로드:**
```bash
# 1. 워크스페이스 생성
mkdir -p ~/vn_ws/src
cd ~/vn_ws/src

# 2. 리포지토리 클론
git clone https://github.com/dawonn/vectornav.git

# 3. ROS 2 브랜치로 변경 (필수 과정)
cd vectornav
git checkout ros2
```
`Branch 'ros2' set up to track remote branch 'ros2' from 'origin'.` 메시지가 뜨면 성공입니다.

### 🛠️ VN-100T용 파라미터 수정
`ros2` 브랜치에는 기본적으로 `vectornav.yaml` 설정 파일이 준비되어 있습니다. 이를 본인 환경에 맞게 수정합니다.

**▶ 설정 파일 폴더:**
```bash
cd ~/vn_ws/src/vectornav/vectornav/config
code .
```

**▶ 내용 수정:**

아래 내용(또는 `vn_100_800hz.yaml` 참고)을 참고하여 수정하세요.

```yaml
vectornav:
  ros__parameters:
    port: "/dev/vectornav" # UDEV Rule 수정했다면 /dev/ttyUSB0 -> /dev/vectornav
    baud: 921600
    reconnect_ms: 500
    
    AsyncDataOutputType: 0      # Async Output Type (ASCII) - 0: OFF (Binary만 사용)
    AsyncDataOutputFrequency: 1 # Not sure why this has to be 1Hz to make it work. Very strange.

    # ... (Sync/Comm 설정은 그대로 두세요) ...
    # ... (중략) ...
    errorMode: 1                     

    # [중요] Binary output register 1 수정 부분
    BO1:
      asyncMode: 1                   # ASYNCMODE_BOTH (Serial 1 & 2 출력)
      
      # 1. 출력 속도 설정 (VN-100 내부 Loop는 800Hz)
      # 계산식: 800 / rateDivisor = 출력Hz
      # 40 -> 8 로 변경 (800 / 8 = 100Hz)
      rateDivisor: 8                 

      commonField: 0x201  # ask for only bare bones IMU data
      timeField: 0x0000              
      imuField: 0x0000               
      attitudeField: 0x0000          
      insField: 0x0000               
      gps2Field: 0x0000              

    # BO2, BO3는 모두 0 (사용 안 함) 그대로 두시면 됩니다.
    BO2:
      asyncMode: 0
      # ... (생략) ...
    BO3:
      asyncMode: 0
      # ... (생략) ...

    # Frame ID는 URDF와 맞춰주세요. (보통 vectornav_link 또는 imu_link 사용)
    frame_id: "vectornav_link"

vn_sensor_msgs:
  ros__parameters:
    use_enu: true
    orientation_covariance: [0.01, 0.0, 0.0, 0.0, 0.01, 0.0, 0.0, 0.0, 0.01]
    angular_velocity_covariance: [0.01, 0.0, 0.0, 0.0, 0.01, 0.0, 0.0, 0.0, 0.01]
    linear_acceleration_covariance: [0.01, 0.0, 0.0, 0.0, 0.01, 0.0, 0.0, 0.0, 0.01]
    magnetic_covariance: [0.01, 0.0, 0.0, 0.0, 0.01, 0.0, 0.0, 0.0, 0.01]
```

**▶ 빌드:**
``` bash
cd ~/vn_ws

# 1. 의존성 설치
sudo apt update
rosdep install --from-paths src --ignore-src -r -y

# 2. 빌드 수행
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
```
**▶ 실행 및 확인:**
```bash
source ~/vn_ws/install/setup.bash

# 런치 파일 실행 (`vectornav.yaml` 파라미터 사용)
ros2 launch vectornav vectornav.launch.py
```
새로운 터미널을 열고 다음을 실행해 데이터를 확인하세요.
```bash
# IMU 데이터 확인
ros2 topic echo /vectornav/imu_uncompensated
```
