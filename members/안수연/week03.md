# [3회차] 인스턴스 생성 전 과정 & SSH — 예습 요약

## 📌 이번 주 할 일
- **이론**: 인스턴스 생성 버튼 뒤에서 일어나는 6단계
- **실습**: 배포받은 VM에 SSH 접속 + 서버 상태 검증

---

## 1. 인스턴스 생성 6단계

| 단계        | 담당                    | 내용                                            | 소요시간           |
| --------- | --------------------- | --------------------------------------------- | -------------- |
| 1. 인증     | Keystone              | 토큰 발급 (`X-Auth-Token`)                        | 수 초 미만         |
| 2. 접수     | nova-api              | 요청 검증 후 DB에 레코드 생성, 상태 `BUILD`                | 수 초 미만         |
| 3. 배치 결정  | scheduler             | Placement 조회 → Filter(탈락) → Weigh(점수) → 노드 확정 | 수 초            |
| 4. 작업 수신  | nova-compute          | RabbitMQ로 비동기(`cast`) 지시 수신                   | 즉시             |
| 5. 준비물 수집 | Glance/Neutron/Cinder | 이미지, 포트+IP, 볼륨 준비                             | **수십 초 (느림①)** |
| 6. 기동     | libvirt→QEMU          | VM 프로세스 실행, `BUILD`→`ACTIVE`                  | **수십 초 (느림②)** |

### 핵심 관점
- **1~4단계 = 빠른 "결정" 구간**, **5~6단계 = 느린 "실물 준비" 구간** → 생성이 느리면 5~6단계를 의심
- 모든 요청에 `req-`로 시작하는 request-id 부여 → 여러 서비스 로그에서 추적 가능
- 대시보드에 "보이는" 시점(2단계, DB 레코드 생성)과 "실제 존재"하는 시점(6단계, ACTIVE) 사이에 간극 존재 → 클라우드 트러블슈팅의 출발점

---

## 2. SSH 원리

- SSH = 원격 터미널을 암호화 채널로 사용하는 프로토콜 (기본 포트 22)
- 클라우드는 거의 항상 **키페어 방식** 인증
  - **공개키**: 자물쇠 → 서버의 `~/.ssh/authorized_keys`
  - **개인키**: 열쇠 → 내 로컬 PC만 보관 (`.pem`)
- 개인키는 절대 유출 금지 (단톡방 업로드 금지)

---

## 3. 실습 절차 (SSH 접속 & 검증)

**① 키 파일 권한 설정 (macOS/Linux만)**
```bash
chmod 600 ~/Downloads/mykey.pem
```

**② 접속**
```bash
ssh -i ~/Downloads/mykey.pem ubuntu@<공인IP>
```
- 첫 접속 시 `authenticity of host` 질문 → `yes` 입력 → 서버 지문이 `known_hosts`에 저장

**③ 서버 검증**
```bash
lsb_release -a        # OS 확인
nproc                 # vCPU 개수
free -h               # 메모리
df -h /               # 디스크 여유공간
egrep -c '(vmx|svm)' /proc/cpuinfo   # 중첩 가상화 지원 여부 (1이상=KVM 가능, 0=에뮬레이션)
ip a                  # 네트워크 인터페이스
```

**④ 인증**: 접속 성공 화면 + 검증 출력 스크린샷 제출

### 자주 만나는 에러

| 메시지 | 의미 | 조치 |
|---|---|---|
| `Permission denied (publickey)` | 키/계정명 오류 | `-i` 경로, 계정명(ubuntu) 확인 |
| `UNPROTECTED PRIVATE KEY FILE` | 키 파일 권한 과다 개방 | `chmod 600` |
| `Connection timed out` | 네트워크가 서버까지 도달 못함 (길이 막힘) | IP, 방화벽(22번 포트), 내 네트워크 확인 |
| `Connection refused` | 서버는 도달했으나 22번에서 거부 (문전박대) | 부팅 직후면 잠시 대기 |

> **timed out = 길이 막힘 / refused = 도착했는데 문전박대** → 이 구분으로 트러블슈팅 방향이 절반으로 줄어듦

---

## 4. [심화] 인스턴스 상태 전이
- scheduling = 3단계 (배치 결정 중)
- networking = 5단계 중 Neutron 포트 작업
- spawning = 6단계 QEMU 기동 중
- `No valid host was found` → 3단계에서 모든 노드가 Filter 탈락 (대부분 자원 부족)

---

## 5. [심화] cloud-init & 메타데이터 서비스

- **cloud-init**: VM 최초 부팅 시 실행되는 초기화 도구
  - SSH 공개키를 `authorized_keys`에 배치
  - 호스트네임 설정, 파티션 확장, user-data 스크립트 실행
- 공개키/설정 정보의 출처 = **메타데이터 서비스** (`http://169.254.169.254`, 링크-로컬 주소)
- OpenStack에서는 Neutron의 `neutron_metadata_agent` 컨테이너가 이 역할 담당

- **트러블슈팅 포인트**: 메타데이터 서비스 장애 시 → "ACTIVE인데 SSH 안 되는" 상황 발생 가능

---

## 6. [심화] known_hosts와 서버 지문

- 서버도 자신만의 호스트 키(지문)를 가짐
- 첫 접속 시 `yes` → 지문이 `~/.ssh/known_hosts`에 저장
- 이후 지문이 다르면 `REMOTE HOST IDENTIFICATION HAS CHANGED!` 경고 (중간자 공격 방지 목적)
- **VM을 지우고 같은 IP로 재발급받으면** 이 경고가 정상적으로 뜸 → `ssh-keygen -R <IP>`로 해결

---

## ✅ 체크리스트
- [ ] VM SSH 접속 성공
- [ ] `lsb_release -a`, `nproc`, `free -h`, `df -h /`, `egrep -c '(vmx|svm)'`, `ip a` 실행 및 결과 확인
- [ ] 중첩 가상화 지원 여부(0 or 1+) 확인
- [ ] 스크린샷 제출