# RHEL 9 이중화·자동화 홈랩 (Ansible)

Proxmox 홈랩에서 RHEL 9 서버 4대(LB 2대, WAS 2대)를 Ansible로 구축한 실습 코드입니다.
HAProxy + Keepalived로 VIP 이중화를 구성하고, JBoss EAP 8 2대에 부하를 분산합니다.
구축 후 일부러 장애를 내서 전환·복구 시간을 측정했습니다.

## 구성

```
사용자 → VIP 10.10.10.80 ─┬─ lb1 (HAProxy + Keepalived MASTER)
                          └─ lb2 (HAProxy + Keepalived BACKUP)
                                   ↓  L7 헬스체크 (GET /app/)
                          was1 · was2 (JBoss EAP 8, :8080)

관리 경로: 관리 PC → ansible (점프 서버, 10.10.10.70) → lb·was 4대
```

## 디렉터리 구조

```
.
├── site.yml          # 전체 실행 파일
├── inventory.ini     # 서버 목록 (lb, was 그룹)
├── ansible.cfg
├── app/index.jsp     # 테스트 앱 소스 (응답한 서버 이름 출력)
└── roles/
    ├── common/       # 모든 서버: 패키지, chronyd, firewalld, 운영 계정, 시간대
    ├── lb/           # HAProxy + Keepalived (Jinja2 템플릿으로 MASTER/BACKUP 구분)
    └── was/          # Java 17, JBoss EAP, systemd 서비스, 방화벽, 앱 배포
```

## 실행 방법

1. 준비: `ansible-core`, `ansible.posix` 컬렉션
2. JBoss EAP zip을 `roles/was/files/`에 두고, `roles/was/defaults/main.yml`의 파일 이름을 맞춥니다. (라이선스 때문에 저장소에는 포함하지 않았습니다.)
3. 테스트 앱 빌드: `cd app && zip -r ../roles/was/files/app.war . && cd ..`
4. 전체 실행: `ansible-playbook site.yml -K`
5. 서버 한 대만 다시 구축: `ansible-playbook site.yml -K --limit was2`

## 장애 테스트 결과

| 테스트 | 결과 | 이유 |
| --- | --- | --- |
| lb1 정상 종료 | 1초 미만 | Keepalived가 우선순위 0 신호를 보내 즉시 인계 |
| lb1 전원 차단 | 약 3초 | VRRP 신호(1초 간격) 3회 누락 후 lb2가 VIP 인수 |
| lb1 HAProxy만 중단 | 약 6초 | 상태 확인 스크립트(2초 간격) 2회 실패 후 판단 |
| was1 JBoss 중단 | 1~2초 실패 후 was2만 응답 | HAProxy 헬스체크가 was1 제외 |
| was1 JBoss 강제 종료 | 약 15초 후 자동 복귀 | systemd `Restart=on-failure` |
| was1 앱만 삭제 | was1 DOWN(L7STS/404), 재실행으로 복구 | L7 헬스체크로 앱 장애 서버 자동 제외 |
| was2 VM 교체 | 약 15분 (수동 10분 + 플레이북 5분), 무중단 | 그동안 was1이 서비스 |

## 배운 점과 개선 과제

- 플레이북을 두 번 실행해 `changed=0`을 확인하며 멱등성을 검증했습니다.
- lb1 복구 시 VIP를 되찾으며 한 번 더 끊겼습니다 → 운영 환경이라면 `nopreempt`를 검토합니다.
- 서버 교체 시간의 3분의 2가 수동 준비였습니다 → 템플릿에 cloud-init을 넣어 단축할 계획입니다.
- Keepalived 인증 비밀번호는 실습용 값입니다 → 운영에서는 Ansible Vault로 암호화합니다.
