# 🧪 컨테이너 부하 시뮬레이션 및 모니터링 시스템

## 📌 소개

이 프로젝트는 Ubuntu 환경에서 Docker 컨테이너들을 순차적으로 실행하며 시스템/컨테이너 리소스 사용량을 주기적으로 기록하고, 부하가 점진적으로 증가하다가 스파이크 후 자동 안정화되는 패턴을 metrics.log에 저장합니다.

매 1분마다 로그 수집

2분 간격으로 컨테이너 2개 실행

각 컨테이너는 10분 후 자동 종료

이후 log를 통해 확인

## 🛠 설치 방법

### 1. Docker 설치

```
sudo apt update
sudo apt install docker.io
```

### 2. sudo 없이 Docker 사용

```
sudo usermod -aG docker $USER
newgrp docker
```

### 3. 네트워크 설정

VM 네트워크를 브릿지 모드로 설정

IP 확인 및 방화벽/포트 설정 (필요 시 MobaXterm 접속용)

### 4. docker 컨테이너

```
docker run -d --name web --network $NETWORK_NAME -p 8080:80 nginx:latest
```

```
docker run -d --name cache --network $NETWORK_NAME redis:latest
```

## 🚀 사용 방법

### 1. 로그 수집 스크립트 설정 (crontab 등록)

```
* * * * * /home/ubuntu/collect.sh >> /home/ubuntu/log/metrics.log 2>&1
```

### 2. 부하 테스트 실행

```
bash /home/ubuntu/docker_stress.sh
```

2분 간격으로 컨테이너 2개 실행 (stress-1 ~ stress-2)

각 컨테이너는 10분 후 자동 종료

## ✨ 주요 기능

1분 주기의 시스템 + 컨테이너 리소스 로깅

계단형 부하 증가 → 스파이크 → 자동 안정화 시나리오

컨테이너별 부하 조절

앞 3개: CPU 20% 제한 + 600MB 메모리

마지막: CPU 최대 + 약 2.7GB 메모리

10분 후 자동 종료 → 자연스러운 하강 그래프


## ⚙️ 사용 shell script

### docker stat을 통한 로그 추출
```
MEM_TOTAL="$(free -m | awk 'NR==2{print $2}')"
MEM_USED="$( free -m | awk 'NR==2{print $3}')"
MEM_PERCENT="$(awk "BEGIN {printf \"%.2f\", ($MEM_USED/$MEM_TOTAL)*100}")"

level="INFO"

host_cpu_total=$(awk -v u="$CPU_USER" -v s="$CPU_SYS" 'BEGIN{printf "%.1f", (u+s)}')
if (( $(awk -v v="$host_cpu_total" -v w="$HOST_CPU_WARN"  'BEGIN{print (v>w)}') )); then level="WARN"; fi
if (( $(awk -v v="$host_cpu_total" -v e="$ESCALATE_ERROR" 'BEGIN{print (v>e)}') )); then level="ERROR"; fi
if (( $(awk -v v="$MEM_PERCENT"     -v w="$HOST_MEM_WARN"  'BEGIN{print (v>w)}') )); then [[ "$level" = "INFO" ]] && level="WARN"; fi
if (( $(awk -v v="$MEM_PERCENT"     -v e="$ESCALATE_ERROR" 'BEGIN{print (v>e)}') )); then level="ERROR"; fi

IFS=',' read -r -a ITEMS <<< "$DOCKER_STATS"
max_c_cpu=0; max_c_mem=0
for item in "${ITEMS[@]}"; do
  c_cpu=$(echo "$item" | awk -F'|' '{gsub("%","",$2); print $2+0}')
  c_mem=$(echo "$item" | awk -F'|' '{gsub("%","",$4); print $4+0}')
  awk -v a="$c_cpu" -v b="$max_c_cpu" 'BEGIN{print (a>b)?a:b}' | read max_c_cpu
  awk -v a="$c_mem" -v b="$max_c_mem" 'BEGIN{print (a>b)?a:b}' | read max_c_mem
done

if (( $(awk -v v="$max_c_cpu" -v w="$CONT_CPU_WARN"  'BEGIN{print (v>w)}') )); then [[ "$level" = "INFO" ]] && level="WARN"; fi
if (( $(awk -v v="$max_c_mem" -v w="$CONT_MEM_WARN"  'BEGIN{print (v>w)}') )); then [[ "$level" = "INFO" ]] && level="WARN"; fi
if (( $(awk -v v="$max_c_cpu" -v e="$ESCALATE_ERROR" 'BEGIN{print (v>e)}') )); then level="ERROR"; fi
if (( $(awk -v v="$max_c_mem" -v e="$ESCALATE_ERROR" 'BEGIN{print (v>e)}') )); then level="ERROR"; fi

echo "$TS CPU_user=$CPU_USER CPU_sys=$CPU_SYS CPU_idle=$CPU_IDLE \
MEM_used=${MEM_USED}MB MEM_percent=$MEM_PERCENT \
DOCKER_STATS=[$DOCKER_STATS] LEVEL=$level" >> "$MAIN_LOG"
```

### 스트레스 부하 컨테이너 자동 실행
```
#!/bin/bash

set -euo pipefail

NET="test-net"
TIMEOUT_SEC=600
STAGGER_SEC=120

docker network ls | grep -q " $NET " || docker network create "$NET"

cleanup_if_exist () {
  local name="$1"
  if docker ps -a --format '{{.Names}}' | grep -qx "$name"; then
    echo "[i] removing old container: $name"
    docker rm -f "$name" >/dev/null 2>&1 || true
  fi
}

run_cpu20_memlight () {
  local name="$1"
  cleanup_if_exist "$name"
  docker run -d --name "$name" --network "$NET" \
    -m 1g --memory-swap 1g \
    alpine sh -c 'apk add --no-cache stress-ng >/dev/null && \
      stress-ng --cpu 1 --cpu-load 20 --vm 1 --vm-bytes 600M --vm-keep --timeout '"${TIMEOUT_SEC}"'s'
}

run_cpu20_memlight "stress-1"
sleep "${STAGGER_SEC}"

run_cpu20_memlight "stress-2"

```

## 📊  결과

### 1. 로그 실시간 확인

```
tail -f /home/ubuntu/log/metrics.log
```

<img width="719" height="300" alt="Image" src="https://github.com/user-attachments/assets/ec77b51b-d601-45d3-94e3-9f944f94748b" />


### 2. LEVEL 별 로그 확인

#### INFO (평상시)

```
awk -F'LEVEL=' 'NF>1{lvl=$2; sub(/[[:space:]].*/,"",lvl); if(lvl=="INFO") print}' /home/ubuntu/log/metrics.log
```

<img width="1592" height="266" alt="Image" src="https://github.com/user-attachments/assets/afe8b634-719e-49f0-ae85-b332528e9ddc" />

#### WARN (CPU 사용량 50% 이상)

```
awk -F'LEVEL=' 'NF>1{lvl=$2; sub(/[[:space:]].*/,"",lvl); if(lvl=="WARN") print}' /home/ubuntu/log/metrics.log
```

<img width="1585" height="581" alt="Image" src="https://github.com/user-attachments/assets/cc2a12ec-eafd-4866-b6b3-40a8256fc789" />

#### ERROR (CPU 사용량 95% 이상)

```
awk -F'LEVEL=' 'NF>1{lvl=$2; sub(/[[:space:]].*/,"",lvl); if(lvl=="ERROR") print}' /home/ubuntu/log/metrics.log
```

<img width="1591" height="463" alt="Image" src="https://github.com/user-attachments/assets/bc9caf97-6605-4b81-ab3b-1a637a3897fe" />

### 3. 최근 CPU 상위 컨테이너 정렬

```
tail -n 10 /home/ubuntu/log/metrics.log \
  | sed -n 's/.*DOCKER_STATS=\[\(.*\)\] LEVEL.*/\1/p' \
  | tr ',' '\n' \
  | awk -F'|' '{print $1, $2}' \
  | sort -k2 -nr
```

<img width="628" height="722" alt="Image" src="https://github.com/user-attachments/assets/cbe2e814-8462-4f21-bba4-c9132820e557" />

### 4. LEVEL 별 개수 확인

```
awk -F'LEVEL=' 'NF>1{lvl=$2; sub(/[[:space:]].*/,"",lvl); cnt[lvl]++} END{for(k in cnt) printf "%s\t%d\n",k,cnt[k]}' /home/ubuntu/log/metrics.log
```

<img width="1586" height="104" alt="Image" src="https://github.com/user-attachments/assets/bbab34be-769e-4df4-846f-5a396001f32d" />

### 5. 시간 구간으로 로그 확인

```
awk -F'[ =]+' '$0 ~ /LEVEL=ERROR/ {for(i=1;i<=NF;i++) if($i=="CPU_user"){sum+=$(i+1); n++}} END{if(n) printf "CPU_user_avg=%.2f%%\n",sum/n}' /home/ubuntu/log/metrics.log
```

<img width="1594" height="301" alt="Image" src="https://github.com/user-attachments/assets/bf4b7c8c-df3c-4fb7-92da-daed9b319408" />

## 🧩 문제 해결

### 1. log 파일 permission denied

<img width="566" height="188" alt="Image" src="https://github.com/user-attachments/assets/5538f259-6512-495e-b7a9-f18f407db13a" />

#### 해결방안

터미널에서 다음 명령어 실행:

```
chmod +x /home/ubuntu/collect.sh
```

그리고 파일 상단에 아래 내용을 추가:

```
#!/bin/bash
```

progrium/stress 이미지 오류	alpine + stress-ng로 대체
메모리 부하가 잘 안 보임	--vm-keep, 부하량 증가, 계산 방식 변경
CPU가 항상 100%처럼 보임	--cpus 옵션으로 제한 비율 적용
