---
title: "[AWS] SAP SLES 15 SP3 → SP6 OS Upgrade 시 SMT 인증 오류 해결기"
excerpt: "오랫동안 업데이트하지 않은 SAP 서버에서 OS Upgrade를 진행하며 겪은 SMT 인증서 문제와 해결 과정을 정리해보자!"

categories:
  - AWS
tags:
  - [AWS, SUSE, SLES, SAP, SMT, PAYG, OS Upgrade, Troubleshooting]

permalink: /aws/sles-sap-os-upgrade-smt-troubleshooting/

toc: true
toc_sticky: true

date: 2026-04-10
last_modified_at: 2026-04-10
---

## 🦥 배경

SAP on AWS 환경에서 운영 중인 SLES(SUSE Linux Enterprise Server) for SAP 서버의 OS Upgrade를 진행하게 되었어요. **SLES 15 SP3 → SLES 15 SP6**으로의 서비스팩 업그레이드였는데, 이 서버는 꽤 오래전에 생성된 이후 한 번도 패키지 업데이트나 등록 갱신을 하지 않은 상태였어요.

업그레이드를 시작하려고 SMT(Subscription Management Tool) 서버에 등록을 시도했더니 인증서 검증 오류가 발생하면서 등록이 되지 않는 문제를 만났고, 이를 해결한 과정을 공유해요.

---

## 🦥 SLES on AWS 인증 구조 (BYOS vs PAYG)

AWS에서 SUSE를 사용하는 방식은 크게 두 가지가 있어요. 이 구조를 이해해야 이번 이슈의 원인을 파악할 수 있어요.

### BYOS (Bring Your Own Subscription)

SUSE에서 직접 라이선스를 구매하고, SUSE Customer Center(SCC)에 직접 등록하는 방식이에요.

```
[EC2 Instance] → (인터넷) → scc.suse.com (SUSE Customer Center)
```

```bash
# SUSE에서 발급받은 Registration Code로 직접 등록
SUSEConnect -r <registration-code>

# 등록 확인
SUSEConnect --status
```

BYOS는 서버가 `scc.suse.com`과 직접 통신해요. 라이선스를 별도로 구매해야 하지만, SUSE와의 통신 경로가 단순하고 인증 관리가 직관적이에요.

### PAYG (Pay-As-You-Go)

AWS Marketplace에서 제공하는 SUSE AMI를 사용하는 방식이에요. SUSE 라이선스 비용이 EC2 인스턴스 비용에 포함되어 있어서 별도 라이선스 구매가 필요 없어요.

```
[EC2 Instance] → (AWS 내부) → smt-ec2.susecloud.net → SUSE 메인 서버
```

PAYG 인스턴스는 AWS 내부에 있는 **SMT/RMT(Repository Mirroring Tool)** 엔드포인트를 통해 SUSE 리포지토리에 접근해요. 이 엔드포인트가 바로 `smt-ec2.susecloud.net`이에요.

PAYG 등록은 `registercloudguest`라는 전용 도구를 사용해요. 이 도구는 `cloud-regionsrv-client` 패키지에 포함되어 있고, 리전 서비스에 질의해서 가장 가까운 SMT/RMT 서버를 자동으로 찾아 등록해줘요.

### 한눈에 비교

| 항목 | BYOS | PAYG |
|---|---|---|
| **라이선스** | SUSE에서 직접 구매 | AWS 비용에 포함 |
| **등록 대상** | scc.suse.com | smt-ec2.susecloud.net |
| **등록 도구** | `SUSEConnect` | `registercloudguest` |
| **인증 방식** | Registration Code | SMT 인증서 기반 자동 인증 |
| **네트워크** | 인터넷 → SUSE 직접 통신 | AWS 내부 → SMT 엔드포인트 |

---

## 🦥 문제 상황

이번에 작업한 SAP 서버는 **PAYG** 방식으로 오래전에 프로비저닝된 인스턴스였어요.

OS Upgrade를 위해 SMT 서버에 등록을 시도했더니 이런 에러가 발생했어요.

```
CERTIFICATE_VERIFY_FAILED - certificate verify failed
```

`smt-ec2.susecloud.net`과의 HTTPS 통신에서 **SSL 인증서 검증에 실패**하는 상황이었어요.

### 원인 분석

장기간 업데이트 없이 방치된 서버였기 때문에 다음과 같은 상황이 복합적으로 작용한 것으로 판단했어요.

1. **SMT 서버의 인증서가 갱신/변경됨**: AWS 내부의 SUSE 인프라가 업데이트되면서 SMT 서버의 SSL 인증서가 변경되었을 가능성
2. **로컬 CA 인증서 번들이 오래됨**: 서버의 CA 번들(`ca-bundle.crt`)이 오래되어 새로운 SMT 인증서의 CA를 신뢰하지 못하는 상태
3. **cloud-regionsrv-client 패키지가 오래됨**: 등록 도구 자체가 최신 인증 방식에 대응하지 못하는 상태

---

## 🦥 해결 과정

### Step 1: 기존 등록 정보 초기화

먼저 기존의 등록 정보와 리포지토리 캐시를 완전히 초기화했어요.

```bash
# 기존 SUSEConnect 등록 정보 삭제
SUSEConnect --cleanup

# zypper 캐시 및 리포지토리 정보 전체 초기화
zypper clean --all

# 기존 SMT 서비스 정보 삭제 (있다면)
rm -f /etc/zypp/services.d/SMT-*
rm -f /etc/zypp/repos.d/SMT-*
```

> `SUSEConnect --cleanup`은 `/etc/SUSEConnect`와 관련된 등록 정보, credentials, 서비스 파일을 정리해줘요.

### Step 2: SMT 서버 인증서 수동 다운로드

`smt-ec2.susecloud.net`의 인증서를 검증할 수 없는 상태이므로, **curl의 `-k`(insecure) 옵션**을 사용해서 인증서를 먼저 다운로드했어요.

```bash
# SMT 서버의 CA 인증서 다운로드
curl -k https://smt-ec2.susecloud.net/smt.crt -o /tmp/smt.crt
```

> `-k` 옵션은 SSL 검증을 건너뛰어요. 인증서를 가져오기 위한 **임시 조치**이므로, 이후 fingerprint 검증을 반드시 수행해야 해요.

### Step 3: 인증서 Fingerprint 확인

다운로드한 인증서가 정상적인 SMT 서버의 인증서가 맞는지 fingerprint를 확인했어요.

```bash
# 인증서 내용 확인
openssl x509 -in /tmp/smt.crt -text -noout

# SHA256 fingerprint 추출
openssl x509 -in /tmp/smt.crt -noout -fingerprint -sha256
# 출력 예시: SHA256 Fingerprint=AA:BB:CC:DD:EE:FF:...
```

별도 채널(AWS Support 또는 SUSE 문서)을 통해 확인한 fingerprint와 대조해서 인증서의 진위를 검증했어요.

### Step 4: CA Trust Store에 인증서 등록

검증된 인증서를 시스템의 CA Trust Store에 등록해요.

```bash
# Trust Anchor 디렉토리에 인증서 복사
cp /tmp/smt.crt /usr/share/pki/trust/anchors/smt-ec2.crt

# CA 인증서 번들 재생성
update-ca-certificates
```

`update-ca-certificates` 명령어가 `/usr/share/pki/trust/anchors/` 디렉토리의 인증서들을 읽어서 `/etc/ssl/ca-bundle.pem`(또는 `/var/lib/ca-certificates/ca-bundle.pem`)을 갱신해줘요. 이후 시스템의 모든 SSL 통신에서 이 인증서를 신뢰하게 돼요.

### Step 5: registercloudguest로 등록

이제 `registercloudguest` 명령어로 SMT 서버에 등록을 진행했어요.

```bash
# SMT 서버의 IP 확인 (DNS 조회 또는 /etc/hosts 확인)
dig smt-ec2.susecloud.net +short
# 또는
getent hosts smt-ec2.susecloud.net

# registercloudguest로 등록
registercloudguest \
  --smt-ip <smt-server-ip> \
  --smt-fqdn smt-ec2.susecloud.net \
  --smt-fp <sha256-fingerprint>
```

| 옵션 | 설명 |
|---|---|
| `--smt-ip` | SMT 서버의 IP 주소 |
| `--smt-fqdn` | SMT 서버의 FQDN (인증서의 CN/SAN과 일치해야 함) |
| `--smt-fp` | SMT 인증서의 fingerprint (openssl로 확인한 값) |

> `--smt-fp`를 지정하면 `registercloudguest`가 SMT 서버에 연결할 때 인증서의 fingerprint를 대조해서 MITM(Man-in-the-Middle) 공격을 방지해줘요.

### Step 6: 등록 확인

```bash
# 등록 상태 확인
SUSEConnect --status
```

정상적으로 등록이 완료되면 활성화된 모듈과 리포지토리 목록이 표시돼요.

### Step 7: Public Cloud Module 활성화

클라우드 환경에서 `zypper migration`을 진행하려면 **Public Cloud Module**이 활성화되어 있어야 해요. `SUSEConnect -l`로 사용 가능한 모듈/확장 목록을 확인하고, Public Cloud Module이 비활성 상태라면 활성화해줘야 해요.

```bash
# 사용 가능한 모듈/확장 목록 확인
SUSEConnect -l

# Public Cloud Module 활성화 (비활성 상태인 경우)
SUSEConnect -p sle-module-public-cloud/15.3/x86_64
```

> Public Cloud Module이 없으면 `zypper migration` 실행 시 마이그레이션 경로를 찾지 못하거나 클라우드 관련 패키지 의존성 오류가 발생할 수 있어요.

### Step 8: 현재 SP의 최신 패키지로 업데이트

`zypper migration`을 바로 실행하면 안 돼요! **먼저 현재 SP 버전에서 가장 최신 패키지로 업데이트**를 진행해야 해요.

```bash
# 현재 SP3의 최신 패키지로 업데이트
zypper up
```

이 단계를 건너뛰면 `zypper migration`이 의존성 문제로 실패할 수 있어요. 특히 `cloud-regionsrv-client`, `zypper`, `libzypp` 같은 핵심 패키지가 최신이어야 마이그레이션이 정상적으로 동작해요.

### Step 9: OS Upgrade (SP 마이그레이션) 진행

```bash
# 사용 가능한 마이그레이션 경로 확인
zypper migration --query

# SP 마이그레이션 실행
zypper migration
```

SLES는 **최대 3단계의 서비스팩을 한 번에 건너뛸 수 있어요.** SP3 → SP6은 3단계 차이이므로 **한 번의 `zypper migration`으로 직접 업그레이드가 가능**해요.

```
SP3 → SP6 (3단계 점프, 가능!)
SP3 → SP4 → SP5 → SP6 (순차 업그레이드도 가능)
```

`zypper migration` 실행 시 사용 가능한 마이그레이션 경로가 번호로 표시되니까, SP6을 선택하면 돼요.

```bash
# 마이그레이션 완료 후 버전 확인
cat /etc/os-release
```

---

## 🦥 전체 작업 흐름 요약

```
 1. SUSEConnect --cleanup            ← 기존 등록 정보 초기화
 2. zypper clean --all               ← 캐시/리포 정보 초기화
 3. curl -k ... /smt.crt             ← SMT 인증서 다운로드
 4. openssl x509 ... -fingerprint    ← fingerprint 확인 및 검증
 5. cp → update-ca-certificates      ← CA Trust Store에 등록
 6. registercloudguest               ← SMT 서버에 재등록
 7. SUSEConnect --status             ← 등록 확인
 8. SUSEConnect -l → 모듈 활성화      ← Public Cloud Module 활성화
 9. zypper up                        ← 현재 SP 최신 패키지로 업데이트
10. zypper migration                 ← SP3 → SP6 업그레이드 진행
```

---

## 🦥 주의 사항

### PAYG 인스턴스는 주기적 갱신이 필요해요

PAYG 인스턴스의 SMT 등록은 영구적이지 않아요. `cloud-regionsrv-client` 패키지가 주기적으로 등록을 갱신하는데, 장기간 업데이트를 하지 않으면 이번처럼 인프라 변경에 대응하지 못하는 상황이 발생할 수 있어요.

### curl -k 사용 시 반드시 fingerprint 검증

`curl -k`는 SSL 검증을 우회하기 때문에 보안 위험이 있어요. 다운로드한 인증서가 정상적인지 반드시 fingerprint를 별도 채널로 확인해야 해요.

### 업그레이드 전 스냅샷/백업 필수

SAP 시스템의 OS Upgrade는 리스크가 큰 작업이에요. 반드시 아래 항목을 사전에 수행해야 해요.

- **AMI 또는 EBS 스냅샷** 생성
- **SAP HANA 백업** (해당 시)
- **변경 전 상태 기록** (`cat /etc/os-release`, `SUSEConnect --status`, `zypper repos` 등)

### Security Group / VPC 엔드포인트 확인

`smt-ec2.susecloud.net`으로의 HTTPS(443) 통신이 가능해야 해요. Private Subnet에 있는 인스턴스라면 NAT Gateway 또는 적절한 네트워크 경로가 설정되어 있는지 확인이 필요해요.

---

## 🦥 마무리

이번 이슈의 핵심은 **장기간 방치된 PAYG 인스턴스의 CA Trust Store가 오래되어 SMT 서버의 갱신된 인증서를 신뢰하지 못한 것**이었어요.

정확히 AWS 측의 SUSE Cloud 인프라에서 무엇이 변경되었는지는 확인하기 어렵지만, SMT 서버의 SSL 인증서가 갱신되었거나 인증 체계가 변경된 것으로 추정돼요. PAYG 인스턴스를 운영한다면 `cloud-regionsrv-client` 패키지와 CA 인증서를 주기적으로 업데이트하는 것이 이런 문제를 예방하는 가장 좋은 방법이에요.

혹시 비슷한 상황을 겪고 있다면, 위 해결 과정을 순서대로 따라해보세요!

---

## 🦥 참고 자료

- [AWS re:Post — Resolve SUSE OS upgrade issues on EC2 instances](https://repost.aws/knowledge-center/ec2-linux-fix-suse-upgrade-errors)
- [AWS Systems Manager — AWSSupport-TroubleshootSUSERegistration](https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-awssupport-troubleshoot-suse-registration.html)
- [SUSE Documentation — Managing cloud instances](https://documentation.suse.com/sle-public-cloud/all/html/public-cloud/cha-admin.html)
- [registercloudguest man page](https://manpages.opensuse.org/Leap-15.6/cloud-regionsrv-client/registercloudguest.1.en.html)
- [SUSE KB — SMT certificate authentication failure](https://www.suse.com/support/kb/doc/?id=000017005)
