# DFU Nordic Firmware

## Version

|버전|내용|
|:--:|----|
|dfu_v0.0|Nordic에서 개발|
|dfu_v1.0|v0.0보다 안정성을 10x 개선|
|dfu_v1.1|dfu 실패 fwid 전송 및 릴레이|
|dfu_v1.2|서로 다른 firmware 간의 dfu 실패 릴레이|
|dfu_v1.3|DFU 모듈화|

## Usage

Application에 DFU 기능을 추가하려면 [이 문서를 보세요](nrf5_SDK_for_Mesh_v5.0.0_src/examples/dfu_module/README.md)

### Example

```shell
cd ~
git clone https://github.com/tot0rokr/MeshSerial-of-nRF5-SDK-for-Mesh.git dfu
cd ~/dfu
cmake -G Ninja \
    -DTOOLCHAIN=gccarmemb \
    -DPLATFORM=nrf52840_xxAA \
    -DBOARD=pca10056 \
    -DSOFTDEVICE=s140_7.2.0 \
    -DBUILD_BOOTLOADER=ON \
    ../nrf5_SDK_for_Mesh_v5.0.0_src/                                        # 펌웨어 빌드환경 빌드
#-------------------------------------------------------------------------- 빌드 환경 세팅 (처음 한 번만 수행)

ninja mesh/bootloader/mesh_bootloader_serial_gccarmemb_nrf52840_xxAA.hex  # bootloader 빌드
ninja dfu_nrf52840_xxAA_s140_7.2.0                                        # application 빌드
#-------------------------------------------------------------------------- 펌웨어 빌드 (펌웨어 변경될 때)

docker run -itd --privileged \
    -v ~/dfu:/dfu \
    -v /dev:/dev \
    --name nrfutil \
    tot0ro/nrfutil                                                          # nrfutil(python2) 실행을 위한 docker
#-------------------------------------------------------------------------- dfu 작업을 위한 도커 실행

cd ~/dfu/dfu
docker exec nrfutil /bin/bash /dfu/dfu/commands.sh config 10000           # bootloader config 생성. 인자는 company id
docker exec nrfutil /bin/bash /dfu/dfu/commands.sh genkey                 # 암호키 생성
./commands.sh version 2                                                   # application version 설정
docker exec nrfutil /bin/bash /dfu/dfu/commands.sh device_page            # device page 생성
#-------------------------------------------------------------------------- dfu 설정

./commands.sh gather_hex                                                  # application, bootloader, softdevice를 가져옴
sudo ./commands.sh download                                               # 펌웨어 다운로드 (옵션은 아래를 참고)
sudo ./commands.sh verify 683241207 "/dev/ttyACM2"                        # 다운로드 펌웨어 검증. (옵션은 아래를 참고)
#-------------------------------------------------------------------------- dfu 펌웨어 다운로드

docker exec nrfutil /bin/bash /dfu/dfu/commands.sh dfu_archive \          # dfu 바이너리 생성.  (옵션은 아래를 참고)
    -a /dfu/dfu/application.hex 3 \                                         # 새로운 펌웨어 application version 및 hex파일 설정
    /dfu/dfu/dfu3.zip                                                       # output
docker exec -it nrfutil /bin/bash /dfu/dfu/commands.sh dfu \              # dfu 전송 (옵션은 아래를 참고)
    /dfu/dfu/dfu3.zip \                                                     # 바이너리
    /dev/ttyACM2 \                                                          # serial COM port
    60                                                                      # 전송 속도
#-------------------------------------------------------------------------- dfu 전송
```


## Explanation

위 예제는 아래 프로세스들을 적용한 예시입니다.
Python2와 Python3 모두 필요합니다.

### Python2 기반 세팅

docker 이미지 tot0ro/nrfutil 이미지를 이용해서 컨테이너로 실행

```bash
docker run -itd -v ~/neogw:/neogw --privileged -v /dev:/dev --name nrfutil tot0ro/nrfutil
docker exec -it nrfutil /bin/bash
```


#### device_page

device page 생성

- config: bootloader\_config.json
- board: nrf52840\_xxAA
- soft device: s140\_7.2.0

#### genkey

dfu를 위한 RSA 암호키를 생성(private_key.txt 생성)하여 bootloader_config.json 에 해당 key값
추가/업데이트.

#### dfu_archive

hex파일을 이용해서 dfu 바이너리를 생성

- `nrfutil dfu genpkg --company-id 0x10000 --key-file private_key.txt --sd-req 0x0100 --mesh`: 이
옵션들은 고정

```bash
./commands.hex dfu_archive {[options, ...]} {output binary name}
```

- `-s {hex file}`
- `-a {hex file} {version}`: id는 현재 1로 고정
- `-b {hex file} {id}`
- `-v` : verbose 옵션


#### dfu

시리얼로 연결된 장치를 통해 dfu 수행

```bash
./commands.hex dfu {dfu archive binary} {COM port} [interval ms] [-v]
```

- interval ms: dfu 전송 패킷 간의 간격. milli seconds 단위로 설정
- `-v`: verbose 모드


### Python3 기반 세팅

로컬에 파이썬3 설치 후 수행

#### verify

device에 넣은 바이너리가 유효한지 검증. root 권한 있어야함.

```bash
./commands.hex verify {serial number} "{COM port}"
```

serial number를 아는 방법

```bash
./nrfjprog --ids
```

#### download

device에 softdevice, bootloader, application, devicepage를 다운로드. nrfjprog설치 되어있어야 함.
root 권한이 있어야 함.

`--chiperase --verify --reset` 옵션은 기본. Chip erase all + Verify binary + Reboot을 포함함.

```bash
./commands.hex download [옵션]
```

- `-s {hex file}`: softdevice
- `-b {hex file}`: bootloader
- `-a {hex file}`: application
- `-d {hex file}`: device page
- `--snr {serial number}`: 특정 시리얼 넘버 장치에 다운로드(기본은 첫 번째 장치에 다운)
- `--log`: logging

#### gather_hex

- board: nrf52840\_xxAA
- soft device: s140\_7.2.0
- compiler: gccarmemb

example/dfu의 application hex 파일과 mesh/bootloader의 bootloader hex 파일을 가져옴.  softdevice와
위에서 생성한 device_\page를 가져옴.

#### config

bootloader\_config.json를 생성. company id를 인자로 받음

#### version

bootloader\_config.json의 application과 bootloader 버전을 설정

### Interface Full Text

```bash
#!/bin/bash
FILE_PATH=`dirname "$0"`
echo $FILE_PATH
echo $0
if [ $# -lt 1 ]; then
    echo "$0 [device_page|genkey|version|dfu_archive|verify|download|dfu]"
    exit 1
fi
python_version=$(python --version 2>&1 | awk '{print $2}' | sed 's/\.[0-9]*$//g')
echo "python: $python_version, operator: $1"
if [ "$1" == "config" ]; then
    if [ $# -lt 2 ]; then
        echo "$0 $1 <company id>"
        exit 1
    fi
    if [ -e "$FILE_PATH/tools_dfu/bootloader_config.json" ]; then
        rm "$FILE_PATH/tools_dfu/bootloader_config.json"
    fi
    cp "$FILE_PATH/tools_dfu/bootloader_config_default.json" "$FILE_PATH/tools_dfu/bootloader_config.json"
    $(python -c "import json; file = open('$FILE_PATH/tools_dfu/bootloader_config.json'); jsonString = json.load(file); jsonString['bootloader_config']['company_id'] = $2; file2 = open('$FILE_PATH/tools_dfu/bootloader_config.json', 'w'); json.dump(jsonString, file2)")
fi
if [ "$1" == "device_page" ]; then
    if [ "$python_version" != "2.7" ]; then
        echo "Need python >= 2.7"
        exit 1
    fi
	python "$FILE_PATH/tools_dfu/device_page_generator.py" -c "$FILE_PATH/tools_dfu/bootloader_config.json" -d nrf52840_xxAA -sd "s140_7.2.0" -o "$FILE_PATH/devicepage.hex"
fi
if [ "$1" == "gather_hex" ]; then
    cp -f "$FILE_PATH/../build/mesh/bootloader/mesh_bootloader_serial_gccarmemb_nrf52840_xxAA.hex" "$FILE_PATH/bootloader.hex"
    cp -f "$FILE_PATH/../build/examples/dfu/dfu_nrf52840_xxAA_s140_7.2.0.hex" "$FILE_PATH/application.hex"
    cp -f "$FILE_PATH/../nrf5_SDK_for_Mesh_v5.0.0_src/bin/softdevice/s140_nrf52_7.2.0_softdevice.hex" "$FILE_PATH/softdevice.hex"
fi
if [ "$1" == "genkey" ]; then
    if [ "$python_version" != "2.7" ]; then
        echo "Need python >= 2.7"
        exit 1
    fi
    if [ -e "$FILE_PATH/private_key.txt" ]; then
        rm "$FILE_PATH/private_key.txt"
    fi
    nrfutil keys --gen-key "$FILE_PATH/private_key.txt"
    key=$(nrfutil keys --show-vk hex "$FILE_PATH/private_key.txt" | awk -F ":" '{print $2}' | xargs echo | sed 's/ //g')
    $(python -c "import json; file = open('$FILE_PATH/tools_dfu/bootloader_config.json'); jsonString = json.load(file); jsonString['bootloader_config']['public_key'] = \"$key\"; file2 = open('$FILE_PATH/tools_dfu/bootloader_config.json', 'w'); json.dump(jsonString, file2)")
fi
if [ "$1" == "version" ]; then
    re='^[0-9]+$'
    if ! [[ "$2" =~ $re ]]; then
        exit 1
    fi
    if [ $# -lt 2 ]; then
        echo "$0 version <version>"
        exit 1
    fi
    $(python -c "import json; file = open('$FILE_PATH/tools_dfu/bootloader_config.json'); jsonString = json.load(file); jsonString['bootloader_config']['application_version'] = $2; file2 = open('$FILE_PATH/tools_dfu/bootloader_config.json', 'w'); json.dump(jsonString, file2)")
fi
if [ "$1" == "dfu_archive" ]; then
    if [ "$python_version" != "2.7" ]; then
        echo "Need python >= 2.7"
        exit 1
    fi
    if [ $# -lt 4 ]; then
        echo "Too few arguments"
        exit 1
    fi
    shift 1
    company_id=$(python -c "import json; file = open('$FILE_PATH/tools_dfu/bootloader_config.json'); jsonString = json.load(file); print(jsonString['bootloader_config']['company_id'])")
    cmd="nrfutil dfu genpkg --company-id $company_id --key-file $FILE_PATH/private_key.txt --sd-req 0x0100 --mesh"
    while [ $# -gt 0 ]; do
        case ${1} in
            "-s"|"--softdevice")
                cmd="$cmd --softdevice $2"
                shift 2
                ;;
            "-b"|"--bootloader")
                re='^(0x)?[0-9A-Fa-f]+$'
                if ! [[ "$3" =~ $re ]]; then
                    echo "bootloader id should be a number."
                    exit 1
                fi
                cmd="$cmd --bootloader $2 --bootloader-id $(($3 * 256 | 0x01))"
                shift 3
                ;;
            "-a"|"--application")
                re='^(0x)?[0-9A-Fa-f]+$'
                if ! [[ "$3" =~ $re ]]; then
                    echo "application version should be a number."
                    exit 1
                fi
                cmd="$cmd --application $2 --application-id 1 --application-version $3"
                shift 3
                ;;
            *)
                cmd="$cmd $1"
                shift 1
        esac
    done
    echo "$cmd"
    $cmd
fi
if [ "$1" == "verify" ]; then
    if [ "$python_version" == "2.7" ]; then
        echo "Need python >= 3"
        exit 1
    fi
    if [ $# -lt 3 ]; then
        echo "$0 verify <serial number> <COM port>"
        exit 1
    fi
    python3 "$FILE_PATH/tools_dfu/bootloader_verify.py" "$2" "$3"
fi
if [ "$1" == "download" ]; then
    if [ "$python_version" == "2.7" ]; then
        echo "Need python >= 3"
        exit 1
    fi
    softdevice="$FILE_PATH/softdevice.hex"
    bootloader="$FILE_PATH/bootloader.hex"
    application="$FILE_PATH/application.hex"
    devicepage="$FILE_PATH/devicepage.hex"
    options=""
    shift 1
    while [ $# -gt 0 ]; do
        case ${1} in
            "-s"|"--softdevice")
                softdevice="$2"
                shift 2
                ;;
            "-b"|"--bootloader")
                bootloader="$2"
                shift 2
                ;;
            "-a"|"--application")
                application="$2"
                shift 2
                ;;
            "-d"|"--devicepage")
                devicepage="$2"
                shift 2
                ;;
            "--snr")
                options="$options --snr $2"
                shift 2
                ;;
            "--log")
                options="$options --log"
                shift 1
                ;;
            *)
                echo "Unknown arguments"
                exit 1
        esac
    done
    mergehex -m $softdevice $bootloader $application $devicepage -o "$FILE_PATH/out.hex"
    nrfjprog --program "$FILE_PATH/out.hex" --chiperase --verify --reset $options
    # nrfjprog --chiperase $options
    # nrfjprog --program $softdevice --verify $options
    # nrfjprog --program $bootloader --verify $options
    # nrfjprog --program $application --verify $options
    # nrfjprog --program $devicepage --verify $options
    # nrfjprog --reset $options
fi
if [ "$1" == "dfu" ]; then
    if [ "$python_version" != "2.7" ]; then
        echo "Need python >= 2.7"
        exit 1
    fi
    interval=200
    if [ $# -ge 4 ]; then
        interval=$4
    elif [ $# -lt 3 ]; then
        echo "$0 dfu <output> <COM port> [<interval ms>] [-v]"
        exit 1
    fi
    if [ $# -eq 5 ] && [ $5 == "-v" ]; then
        nrfutil --verbose dfu serial -pkg $2 -p $3 -b 115200 -fc --mesh -i "$interval"
    else
        nrfutil dfu serial -pkg $2 -p $3 -b 115200 -fc --mesh -i "$interval"
    fi
fi
```
