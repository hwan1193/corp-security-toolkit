# Linux Security Quick Check (Ubuntu / Rocky / RHEL 공통)

운영/테스트 서버에서 계정/권한, 네트워크, 서비스, 로그, 무결성, 흔적(IR)을 빠르게 확인하는 **읽기 전용 점검 명령어 모음**.

> ⚠️ 원칙  
> - 운영 서버는 **범위 좁게(/etc, 웹루트, 특정 서비스)** 점검  
> - `find /` 같은 전체 스캔은 부하/시간 큼 → **-xdev / 특정 경로 제한**  
> - 조치(차단/변경)는 별도 승인 후 진행

---

## 0. 기본 정보 / 자원 상태

| 항목 | 명령어 | 정상 기준(예시) | 비정상 시 의심 |
|---|---|---|---|
| OS/커널 | `cat /etc/os-release` `uname -a` | 기대 OS/커널 | 낯선 커널/버전 급변 |
| 호스트/업타임 | `hostnamectl` `uptime` | 예상 호스트명, 정상 uptime | 재부팅 흔적(원인 확인) |
| 디스크/메모리 | `df -hT` `free -h` | 임계치 미만(여유) | 디스크 90%↑, mem 스왑 폭증 |

```bash
hostnamectl
cat /etc/os-release
uname -a
uptime
df -hT
free -h
timedatectl
```

---

## 1. 계정 / 권한 / 인증(SSH 포함)

### 1-1) 로컬 계정 / 로그인 가능한 계정

```bash
cat /etc/passwd
cut -d: -f1 /etc/passwd

# 로그인 가능한 계정만(쉘이 nologin/false 아닌 것)
awk -F: '$7 !~ /(nologin|false)/ {print $1":"$7}' /etc/passwd
```

**정상 기준**
- 서비스 계정은 보통 `/usr/sbin/nologin` 또는 `/bin/false`
- “사람 계정”만 로그인 쉘 존재

**비정상 포인트**
- 낯선 사용자 + 로그인 쉘(`/bin/bash`) + sudo 권한까지 있으면 의심

---

### 1-2) UID 0(루트 권한) 계정 확인

```bash
awk -F: '$3==0 {print $1}' /etc/passwd
```

**정상:** `root`만  
**비정상:** root 외 UID 0 계정 존재 → 거의 사고급

---

### 1-3) sudo 권한자 확인

```bash
# Ubuntu/Debian
getent group sudo

# RHEL/Rocky/CentOS
getent group wheel

# sudoers 파일(읽기)
sudo cat /etc/sudoers | egrep -v '^\s*#|^\s*$'
sudo ls -la /etc/sudoers.d 2>/dev/null
sudo cat /etc/sudoers.d/* 2>/dev/null
```

**정상 기준**
- sudo 그룹 인원 최소
- sudoers.d에 낯선 파일 없음

---

### 1-4) 최근 로그인/실패 로그인(브루트포스 흔적)

```bash
last -a | head
lastb -a | head 2>/dev/null

# SSH 로그(서비스명은 배포판별)
journalctl -u ssh --since "24 hours ago" | tail -n 80 2>/dev/null
journalctl -u sshd --since "24 hours ago" | tail -n 80 2>/dev/null
```

**비정상 포인트**
- 실패 로그인 폭증
- 특정 IP가 반복 시도
- 새벽/비업무 시간대 관리자 로그인

---

### 1-5) SSH 설정(중요)

```bash
# 실제 적용 설정 출력(권장)
sshd -T | egrep 'port|permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|allowgroups|x11forwarding' 2>/dev/null

# config 원문(주석 제거)
cat /etc/ssh/sshd_config | egrep -v '^\s*#|^\s*$'
```

**정상 기준(권장)**
- `PermitRootLogin no`
- `PasswordAuthentication no` (가능하면)
- `PubkeyAuthentication yes`
- Allowlist(AllowUsers/AllowGroups) 있으면 더 좋음

---

## 2. 네트워크 / 포트 / 세션 / 방화벽

### 2-1) 리스닝 포트 / 외부 세션 확인

```bash
# 리스닝 포트(프로세스 포함)
ss -lntup

# 연결 세션(ESTABLISHED 위주)
ss -antp | awk '$1=="ESTAB"{print}' | head -n 50
```

**비정상 포인트**
- 예상 못한 포트가 LISTEN(예: 4444, 8081 등)  
- 외부 IP로 지속 ESTABLISHED 세션 + 해당 프로세스가 낯설다

---

### 2-2) 네트워크 설정(인터페이스/라우팅/DNS)

```bash
ip a
ip r
resolvectl status 2>/dev/null || cat /etc/resolv.conf
```

---

### 2-3) 방화벽 상태(ufw/firewalld/nft/iptables)

```bash
# Ubuntu
ufw status verbose 2>/dev/null

# RHEL/Rocky
firewall-cmd --state 2>/dev/null
firewall-cmd --list-all 2>/dev/null

# nftables/iptables
nft list ruleset 2>/dev/null | head -n 120
iptables -S 2>/dev/null | head -n 120
```

**정상 기준**
- 정책 존재(인바운드 제한)
- 운영 서버는 “필요한 포트만” 오픈

---

## 3. 서비스 / 프로세스 / 자동실행(지속성) / 스케줄러

### 3-1) 상위 프로세스(CPU/MEM)

```bash
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```

**비정상 포인트**
- 낯선 프로세스가 과점유
- 실행 경로가 `/tmp`, `/dev/shm` 같은 위치

---

### 3-2) systemd 서비스(Enabled / Failed)

```bash
systemctl --failed
systemctl list-unit-files --type=service | awk '$2=="enabled"{print $1}' | head -n 80
systemctl list-timers --all | head -n 80
```

---

### 3-3) 크론/AT(지속성 1순위)

```bash
crontab -l 2>/dev/null
sudo crontab -l 2>/dev/null

ls -la /etc/cron.* /etc/crontab 2>/dev/null
ls -la /var/spool/cron 2>/dev/null

atq 2>/dev/null
```

**비정상 포인트**
- `curl/wget|bash` 형태 실행
- 알 수 없는 스크립트 주기 실행
- root crontab에 낯선 항목

---

## 4. 패키지 / 업데이트 상태

```bash
# Ubuntu
apt list --upgradable 2>/dev/null | head -n 60

# RHEL/Rocky
dnf check-update 2>/dev/null | head -n 60
rpm -qa | wc -l
```

---

## 5. 로그 기반(인증/권한상승/의심 행동)

### 5-1) auth 로그 파일(있는 경우)

```bash
# Ubuntu
sudo tail -n 200 /var/log/auth.log 2>/dev/null

# RHEL/Rocky
sudo tail -n 200 /var/log/secure 2>/dev/null
```

### 5-2) sudo 로그(journal)

```bash
journalctl _COMM=sudo --since "24 hours ago" | tail -n 200
```

**비정상 포인트**
- 갑자기 sudo 사용 폭증
- 새 사용자 생성/권한 변경 흔적
- 시간대가 이상함

---

## 6. 파일 무결성 / 최근 변경 / 웹쉘 후보

> 운영 서버는 **전체 디스크 스캔 금지**. `/etc`, 웹루트, 특정 앱 경로만.

### 6-1) /etc 최근 변경(설정 변조)

```bash
sudo find /etc -type f -mtime -1 -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | head -n 80
```

### 6-2) 웹루트(nginx/apache) 최근 변경

```bash
# 흔한 웹루트 예시(환경에 맞게 하나만)
sudo find /var/www -type f -mtime -3 -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | head -n 80
```

### 6-3) 웹쉘 키워드(가볍게)

```bash
sudo grep -R --line-number "eval(|base64_decode|gzinflate|shell_exec|passthru|system(" /var/www 2>/dev/null | head -n 30
```

---

## 7. 권한 상승 흔적(SUID/SGID) + 이상 경로

```bash
# 같은 파티션 내만(-xdev)로 제한
sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -printf '%m %u %g %p\n' 2>/dev/null | head -n 120
```

**비정상 포인트**
- SUID 파일이 `/tmp`, `/var/tmp`, `/dev/shm` 등에 존재 → 거의 사고

---

## 8. 보안 모듈 상태(SELinux/AppArmor) + 핵심 sysctl

```bash
# RHEL/Rocky
getenforce 2>/dev/null
sestatus 2>/dev/null

# Ubuntu
aa-status 2>/dev/null

# sysctl 핵심
sysctl net.ipv4.ip_forward net.ipv4.conf.all.accept_redirects net.ipv4.conf.all.send_redirects net.ipv4.conf.all.rp_filter 2>/dev/null
```

---

## 9. IR 빠른 요약(복붙용)

```bash
echo "=== HOST/OS ==="; hostnamectl; cat /etc/os-release; echo
echo "=== UID0 ==="; awk -F: '$3==0{print $1}' /etc/passwd; echo
echo "=== SUDO GROUP ==="; getent group sudo 2>/dev/null; getent group wheel 2>/dev/null; echo
echo "=== SSH EFFECTIVE ==="; sshd -T 2>/dev/null | egrep 'port|permitrootlogin|passwordauthentication|allowusers|allowgroups' ; echo
echo "=== LISTEN ==="; ss -lntup | head; echo
echo "=== ESTAB ==="; ss -antp | awk '$1=="ESTAB"{print}' | head; echo
echo "=== FAILED SERVICES ==="; systemctl --failed; echo
echo "=== TIMERS ==="; systemctl list-timers --all | head; echo
echo "=== AUTH TAIL ==="; tail -n 80 /var/log/auth.log 2>/dev/null || tail -n 80 /var/log/secure 2>/dev/null
```

---

# (별도) 외부 노출/이상 징후 시 "조치 방향" 체크리스트 (명령 최소)

> 아래는 방향만. 변경 전 승인/백업/롤백 플랜 필수.

- SSH 외부 노출이면: **Allowlist + Key 기반 + Root 로그인 금지 + PasswordAuth 차단**
- 방화벽: 인바운드 최소화(필요 포트만)
- 의심 계정/크론/서비스 발견 시: 증적 확보 → 격리/차단 → 재발 방지

---

## License
Internal / Team Use
