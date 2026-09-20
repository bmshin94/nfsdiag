# nfsdiag 전수조사 분석 정리 (한국어)

> 이 문서는 `nfsdiag` 저장소를 전수조사하며 나눈 대화 내용을 정리한 기록입니다.
> 작성일: 2026-09-20

## 관련 링크

| 항목 | 주소 |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/nfsdiag |
| 원본 저장소 (upstream) | https://github.com/lsferreira42/nfsdiag |
| 공식 웹사이트 | https://www.nfsdiag.org |
| 릴리스 | https://github.com/lsferreira42/nfsdiag/releases/latest |
| 컨테이너 이미지 | `ghcr.io/lsferreira42/nfsdiag` |
| JSON 스키마 | [docs/nfsdiag.schema.json](./nfsdiag.schema.json) |
| man 페이지 | [docs/nfsdiag.8](./nfsdiag.8) |

---

## 1. 프로젝트 개요

`nfsdiag`는 **NFS(Network File System)가 왜 동작하지 않는지 자동으로 진단하는 C 언어 CLI 도구**입니다.

| 항목 | 내용 |
|---|---|
| 언어 | 순수 C (C11), 약 9,282줄 |
| 버전 | 0.22.0 |
| 라이선스 | MIT (상업적 이용·수정·재배포 자유) |
| 플랫폼 | Linux 전용 (`/proc`, `mount.nfs` 의존 — macOS/BSD 미지원) |
| 아키텍처 | amd64, arm64 사전 빌드 제공 |
| 런타임 의존성 | `libtirpc`, `nfs-utils`/`nfs-common`, (서버측 NFSv3 탐색 시) `rpcbind` |
| 개발 기간 | 2026-04-30 첫 커밋 → 2026-09-18 (약 5개월에 0.22.0) |
| 기여자 | Leandro Ferreira(37 커밋), bmshin94(2), dependabot(2) |

### 서브커맨드

```bash
nfsdiag client <host>            # 외부에서 NFS 서버를 진단 (클라이언트 관점)
nfsdiag server --all             # NFS 서버 본인의 설정/상태 감사
nfsdiag diff before.json after.json   # 두 JSON 리포트 비교
nfsdiag version | help
```

> 참고: 서브커맨드 없는 `nfsdiag [OPTIONS] <host>` 형태는 deprecated이며 1.0에서 제거 예정입니다.

---

## 2. 폴더 구조 전수조사

```
nfsdiag/
├── src/                      핵심 C 소스 (본체)
│   ├── main.c        (  79줄) 서브커맨드 라우터
│   ├── client.c      (1575줄) 클라이언트 진단 메인 로직
│   ├── server.c      (1718줄) 서버 진단 메인 로직
│   ├── server_checks.c(568줄) 서버 개별 점검 항목
│   ├── server_exports.c(196줄) /etc/exports 파싱·감사
│   ├── rpc.c         ( 631줄) RPC/rpcbind 통신 (libtirpc, XDR)
│   ├── network.c     ( 222줄) TCP 도달성, 지연, MTU
│   ├── mount.c       ( 539줄) 마운트 시도 (v4.2→4.1→4→3 캐스케이드)
│   ├── tests.c       ( 814줄) 파일시스템 테스트 (권한/ACL/락/ESTALE)
│   ├── stats.c       ( 412줄) /proc/self/mountstats 파싱
│   ├── report.c      ( 916줄) JSON/HTML/table/NDJSON/Prometheus/JUnit 출력
│   ├── serve.c       ( 121줄) HTTP 메트릭 리스너 (--listen)
│   ├── peer.c        (  59줄) paired 모드 (클라이언트↔서버 상관분석)
│   ├── validation.c  ( 358줄) 입력 검증 (보안 핵심)
│   ├── util.c / bench.c / ebpf.c / ebpf_latency.c
│   ├── bpf/nfsdiag.bpf.c(124줄) 커널에 로드되는 BPF 프로그램
│   └── nfsdiag.h     ( 629줄) 공통 헤더 (RPC 프로그램 번호, 한계값, 구조체)
├── dockerfiles/              고장난 NFS 상황을 재현하는 테스트 픽스처 14종
├── tests/                    유닛/골든/스키마/시그널/서브커맨드 테스트 + 퍼징 4종
├── packaging/                deb, rpm, apk, Homebrew, AUR 패키징 템플릿
├── website/                  공식 문서 사이트 정적 HTML (Cloudflare 배포)
├── docs/                     man 페이지, JSON 스키마, (이 문서)
├── completions/              bash / zsh / fish 자동완성
├── .github/workflows/        CI (Ubuntu/Fedora/Debian/Alpine 매트릭스), 릴리스, 웹사이트 배포
├── configure / configure.ac  autoconf 빌드 (--enable-ebpf 토글)
├── Makefile (20KB)           빌드, 테스트, 패키징, 버전 범프
├── flake.nix                 Nix 플레이크
└── CHANGELOG.md (25KB) / README.md (34KB) / SECURITY.md / CONTRIBUTING.md
```

---

## 3. 기능 상세

### 3-1. client 모드 — 외부에서 서버 진단

계층별로 약 40단계를 순차 검사합니다.

1. **네트워크층** — rpcbind(111)/NFS(2049) 포트 도달성, TCP 연결 지연, 경로 MTU, IPv4/IPv6 강제
2. **RPC층** — rpcbind v3/v4 DUMP로 서비스 맵 조회(네이티브 IPv6 + 레거시 portmapper), 동적 등록된 mountd/lockd/statd 포트의 방화벽 통과 여부, RPC 서비스 레이아웃 기반 서버 구현체 핑거프린팅
3. **프로토콜층** — NFS v2/v3/v4(+4.1/4.2 힌트) NULLPROC, mountd v1/v2/v3, 선택적 UDP 프로브
4. **익스포트 열거** — mountd로 export 목록 조회 (XDR 직접 디코딩)
5. **클라이언트 전제조건** — `nfs-client.target`, `rpc.gssd`, `nfs-idmapd` 상태
6. **Kerberos** — `--krb5`로 티켓/설정 확인, `sec=krb5`/`krb5i`/`krb5p` 실제 마운트 테스트
7. **마운트** — NFSv4.2 → 4.1 → 4 → 3 캐스케이드, private mount namespace 안에서 수행
8. **마운트 옵션 검증** — `/proc/self/mountinfo`에서 실제 적용된 옵션 파싱
9. **통계** — RPC 재전송/인증 갱신(before/after), `/proc/self/mountstats` 심층 지연 지표, `/proc/fs/nfsfs/servers`
10. **파일시스템 테스트** — close-to-open 일관성, 특수 파일, 쿼터, 읽기/순회 권한, 디렉터리 목록
11. **ACL/속성** — POSIX ACL, NFSv4 ACL, 일반 xattr, SELinux 컨텍스트
12. **I/O** — create/write/read/fsync, `copy_file_range`, `fallocate`, `O_DIRECT`
13. **락** — `fcntl` 권고 락
14. **보안** — 실질적 `root_squash` 동작 확인, UID/GID + 보조 그룹 시뮬레이션
15. **성능** — 메타데이터 지연 벤치(create/rename/unlink), 내부 스모크 벤치 또는 외부 `fio`, `--sweep`으로 rsize/wsize/nconnect 조합 벤치 후 최적 마운트 옵션 제안
16. **안정성** — ESTALE 루프(기본 100회), 255바이트 긴 파일명, 공백/콜론/UTF-8 멀티바이트 특수문자
17. **NFSv4 고급** — delegation 활동(DELEGRETURN), pNFS 레이아웃 감지

### 3-2. server 모드 — 서버 자체 감사 (25종 체크)

| 체크 | 내용 |
|---|---|
| `--daemons` | nfsd, rpcbind, rpc.mountd, rpc.statd, rpc.idmapd, gssproxy 상태 + rpcbind 등록 |
| `--exports-audit` | `/etc/exports` 문법 오류, 클라이언트 목록 없는 export, `no_root_squash`·`insecure`·와일드카드 위험 옵션 |
| `--ports-firewall` | 2049/111 리스너, 동적 mountd/statd/lockd 포트, firewalld/nftables 규칙 |
| `--storage-health` | export별 디렉터리 존재 여부, 공간/inode 사용률, tmpfs·overlayfs·NFS 재export 경고 |
| `--version-matrix` | 활성화된 NFS 버전, lease/grace 시간, 블록 크기 |
| `--sysctl-advisor` | nfsd 스레드 고갈(`/proc/net/rpc/nfsd`), 네트워크 버퍼 튜너블 권고 |
| `--security-audit` | `subtree_check`, `insecure_locks`, `anonuid` 없는 `all_squash`, `sec=sys`, 중복/중첩 export |
| `--idmap-check` | `idmapd.conf` 도메인 vs DNS 도메인 (`nobody:nobody`의 전형적 원인), nobody 매핑 |
| `--krb5-server` | 키탭 존재/권한, `nfs/` 프린시펄, 기본 realm, gss 데몬, NTP 동기화 |
| `--acl-check` | export 하위 파일시스템의 POSIX ACL 지원 |
| `--squash-check` | localhost에서 마운트 후 root로 파일 생성해 실제 매핑 확인 (침습적, `--all` 미포함, root 필요) |
| `--audit-trail` | exports/nfs.conf/idmapd.conf/krb5.conf 사본 + `CONFIG.SHA256SUMS` (사고 증거 보존) |
| `--rpc-stats` | reply cache 적중률, bad RPC calls, 패킷/바이트 트래픽 |
| `--locks` | `/proc/locks` 유형별 락, NFSv4 lease/grace, NLM/NSM 등록 |
| `--clients` | `/proc/fs/nfsd/clients/` 접속 NFSv4 클라이언트, minor 버전, 콜백 상태 |
| `--client-states` | 클라이언트별 open/lock/delegation/layout 보유 수, 핸들 누수·delegation 독점 탐지 |
| `--log-intel` | 저널(또는 `--root` 하위 `/var/log/messages`)에서 알려진 nfsd/mountd/statd 문제 시그니처 탐지 + 해결책 |
| `--rmtab-audit` | 오래된 `rmtab` 항목, 고아 NSM 모니터(부팅 시 sm-notify 폭주 원인) |
| `--memory-pressure` | MemAvailable vs reply cache, dentry/inode 캐시, `vm.*` 튜너블 |
| `--latency-profile` | **eBPF**: nfsd read/write/commit 지연 히스토그램 (root + eBPF 빌드 필요) |
| `--per-client-trace` | **eBPF**: 클라이언트별 nfsd ops/평균 지연 |
| `--backend-bench` | export 하위 스토리지 원시 성능(디스크 상한) 측정 |
| `--capture` | `tcpdump`로 2049 포트 캡처, `tshark` 있으면 요약 |
| `--ha-check` | 명시적 `fsid=` 누락, `/var/lib/nfs` 공유 스토리지 여부, pacemaker NFS 리소스 |
| `--ganesha-check` | nfs-ganesha(유저랜드) vs 커널 nfsd 감지, `ganesha.conf` EXPORT/FSAL 파싱, 컨테이너/K8s 환경 감지 |

> `--root DIR`로 추출한 sosreport 트리를 지정하면 **서버 접속 없이 오프라인 진단**이 가능합니다.

### 3-3. Paired 모드 (0.22.0 신규)

서버와 클라이언트를 동시에 관측해 **"서버/네트워크/클라이언트 중 누가 병목인지"** 판정합니다.

```bash
# 서버에서
nfsdiag server --listen 0.0.0.0:9100 --watch 2
# 클라이언트에서
nfsdiag client --peer <server>:9100 <server>
```

출력 예: `paired: both sides healthy — 211 ops served during the test, no bad calls`

### 3-4. 출력 포맷 6종

| 포맷 | 소비 주체 | 사용법 |
|---|---|---|
| text | 사람(터미널) | 기본값, `--verbose`로 전체 단계 |
| table | 사람(요약표) | `--output-format=table` |
| json | 프로그램/AI | `--json[=PATH]` (schema_version 2.0 고정) |
| html | 보고서 | `--html[=PATH]` (CSP 헤더 포함) |
| ndjson | 로그 파이프라인 | `--output-format=ndjson` |
| prometheus | 모니터링 | `--output-format=prometheus`, `--listen` |
| junit | CI/CD | `--output-format=junit` |

JSON에는 `schema_version`, `timestamp_iso8601`, `duration_sec`, 이벤트 `category`, 안정적인 `check_id`, `severity`, `remediation`(해결책 텍스트)이 포함됩니다.

### 3-5. Docker 픽스처 14종

고장난 NFS 상황을 의도적으로 재현해 진단기가 제대로 탐지하는지 회귀 테스트합니다.

`rpcbind-unreachable`, `nfs-port-unreachable`, `rpc-map-missing-nfs`, `mountd-unavailable`,
`empty-exports`, `mount-denied`, `permission-denied`, `acl-unsupported`, `identity-denied`,
`read-only-export`, `root-squash`, `locking-missing`, `stale-handle`, `slow-performance`

```bash
make docker-build-all   # 전체 픽스처 이미지 빌드
make test-fixtures      # 전체 픽스처 테스트
make test-fixtures-list # 목록
```

> 주의: 픽스처는 `no_root_squash`, `insecure`, 와일드카드를 **의도적으로** 사용합니다. 테스트 전용이며 운영에 쓰면 안 됩니다.

### 3-6. 보안 설계

- 비파괴 기본값: 고유 랜덤 이름(`getrandom()`) + `O_CREAT|O_EXCL`로 자기 테스트 파일만 생성/삭제
- 기본 마운트 옵션 `vers=<n>,nosuid,nodev,noexec` (해제하려면 `--allow-risky-mount-options`)
- 마운트는 private mount namespace에서 수행, `hard` 마운트 유지(hang은 워커 데드라인과 `--command-timeout`으로 제한)
- 호스트/경로/마운트 옵션을 네트워크 활동 전에 검증 (`validation.c`)
- `--on-fail-exec` 스크립트와 `--config` 파일은 소유자/권한 검사 통과해야 실행, 셸 경유 없이 신뢰 경로로 실행
- 외부 명령 출력의 터미널 이스케이프 시퀀스 살균, 서버 제공 XDR 문자열의 제어문자 살균
- 리포트 파일은 `O_NOFOLLOW` + 모드 `0600`, HTML은 CSP 헤더 + 전체 이스케이프
- UID/GID 시뮬레이션 자식 프로세스는 `setuid()` 전에 ambient capability 제거
- 심링크/하드링크/FIFO/디바이스 노드 프로브는 `--dangerous-fs-tests` 필요
- ⚠️ `--listen` 익스포터는 **인증·TLS 없음**. 기본 127.0.0.1 바인딩, 외부 노출 시 리버스 프록시 필수

---

## 4. 설치 및 사용법

### 설치

```bash
# ① Docker (컴파일 불필요)
docker run --rm --privileged ghcr.io/lsferreira42/nfsdiag client 192.168.1.10

# ② 소스 빌드
sudo apt-get install -y build-essential pkg-config libtirpc-dev nfs-common   # Debian/Ubuntu
sudo dnf install -y gcc make pkgconf-pkg-config libtirpc-devel nfs-utils     # Fedora/RHEL
git clone https://github.com/bmshin94/nfsdiag && cd nfsdiag
make && make check && sudo make install

# ③ eBPF 기능 포함 (clang, bpftool, libbpf 필요)
./configure --enable-ebpf && make && sudo make install

# ④ 패키지 생성
make deb | make rpm | make apk | make packages

# ⑤ GitHub Releases의 사전 빌드 바이너리(amd64/arm64) + SBOM + 체크섬
```

### 자주 쓰는 명령

```bash
sudo nfsdiag client 192.168.1.10                     # 기본 전체 진단
nfsdiag client --no-mount 192.168.1.10               # 네트워크/RPC만
sudo nfsdiag client --dry-run 192.168.1.10           # 실행 계획만 출력
sudo nfsdiag client --profile quick|safe|full|performance|security|readonly <host>
sudo nfsdiag client --export /data 192.168.1.10      # 특정 export만
sudo nfsdiag client --sweep 192.168.1.10             # 최적 마운트 옵션 탐색
sudo nfsdiag client --uid 1000 --gid 1000 <host>     # 특정 사용자 권한 시뮬레이션
sudo nfsdiag client --parallel 4 <host>              # export 동시 테스트
sudo nfsdiag client --output-dir ./evidence <host>   # JSON+HTML+증거+SHA256SUMS
sudo nfsdiag client --hosts-file /etc/nfs-servers.txt --json=audit.json
sudo nfsdiag client --watch 60 <host>                # 주기 재실행
sudo nfsdiag client --diff-baseline <host>           # 직전 기준선과 비교
sudo nfsdiag server --all
sudo nfsdiag server --exports-audit --security-audit
nfsdiag server --listen 127.0.0.1:9100 --watch 30    # Prometheus 익스포터
nfsdiag diff before.json after.json
```

### 종료 코드

| 코드 | 의미 |
|---|---|
| 0 | 경고·실패 없음 |
| 1 | 경고 또는 실패 발견 (자동화에서 주의 필요) |
| 2 | 사용법 오류 또는 로컬 런타임 오류 |

> 전체 마운트 진단은 root(또는 `CAP_SYS_ADMIN`) 필요. `--no-mount`, `--self-test`, `--help`, `--version`은 일반 사용자로 실행 가능.

---

## 5. Q&A 정리

### Q. 플러그인? 스킬? MCP?

**셋 다 아닙니다. 독립 실행형 네이티브 CLI 바이너리입니다.**

- `src/main.c`의 `int main(int argc, char **argv)` — 전형적 CLI 엔트리포인트
- MCP의 필수 요소인 JSON-RPC stdio 핸들러가 코드에 없음
- `package.json`, `mcp.json`, `SKILL.md` 모두 없음
- 런타임 의존성은 `libtirpc` 뿐 (Node/Python 런타임 불필요)

> 저장소 루트의 `CLAUDE.md`는 이 포크에서 추가한 개발 페르소나 설정 파일이며, 프로젝트 본체 기능과 무관합니다.
>
> 다만 **MCP 서버로 감싸기에는 매우 좋은 후보**입니다: 고정 JSON 스키마, 명확한 종료 코드, 깔끔한 서브커맨드, NDJSON 스트리밍.

### Q. API 토큰이 필요한가?

**전혀 필요 없습니다.** 소스 전체에 API 키/토큰/인증/라이선스 서버/텔레메트리 코드가 없고, 외부 서비스로 나가는 통신도 없습니다. MIT 라이선스로 완전 무료입니다.

관련되는 "인증"은 NFS 자체의 것입니다:
- `--krb5` — 시스템의 기존 Kerberos 티켓/설정 사용
- `--uid/--gid` — 로컬 UID/GID 시뮬레이션(root 필요)
- `--listen` — 익스포터에 인증/TLS 없음, 기본 loopback 바인딩

### Q. 왜 GitHub에서 주목받는가?

> 정확한 스타 수는 이 환경에서 GitHub API 접근이 차단되어 확인하지 못했습니다. 아래는 코드/저장소 구조에서 확인되는 근거입니다.

1. **보편적 고통 지점** — NFS 장애 디버깅은 리눅스 인프라 담당자 누구나 겪지만 절차가 복잡함
2. **문서 밀도** — README 34KB/760줄, 전용 웹사이트, man 페이지, JSON 스키마, CHANGELOG 25KB, 한계(Limitations)를 솔직히 명시
3. **완성도** — 4종 OS CI 매트릭스(glibc+musl), `-Werror`, clang-tidy, shellcheck, 퍼징 4종, Docker 픽스처 14종, 5종 패키징, SBOM/체크섬/provenance, SECURITY.md/CoC/CONTRIBUTING
4. **의존성 최소** — C + libtirpc. 에어갭/레거시/컨테이너 환경에 바이너리 하나로 배포 가능
5. **모던 키워드 조합** — eBPF + Prometheus/OpenMetrics + K8s/ganesha 감지 + JSON 스키마로 레거시 프로토콜을 감쌈
6. **개발 속도** — 5개월에 0.22.0, 릴리스마다 기능 뭉텅이 추가

### Q. 로컬 에이전트 구축에 도움이 되는가?

**세 가지 레벨에서 도움이 됩니다.**

**레벨 1 — 에이전트의 도구(Tool)로 직접 사용**

| 에이전트 도구 요구사항 | nfsdiag |
|---|---|
| 구조화된 출력 | JSON schema 2.0 고정 |
| 명확한 성공/실패 신호 | exit code 0/1/2 |
| 스트리밍 | NDJSON |
| 안전 모드 | `--dry-run`, `--read-only`, `--no-mount` |
| 타임아웃 제어 | `--timeout`, `--command-timeout`, `--fs-timeout` |
| 멱등성 | 진단 전용, 서버 설정 변경 없음 |
| 해결책 제공 | `recommendations` + `remediation` 필드 |

마지막 항목이 핵심입니다. `check_id`, `category`, `severity`, `remediation`이 JSON에 이미 들어 있어 **LLM이 원인과 해결책을 추론할 필요가 없고, 환각 위험이 크게 줄어듭니다.**

**레벨 2 — 좋은 에이전트 도구 설계의 참고 사례**

1. 안전 등급을 플래그로 분리 (`--dry-run` / `--read-only` / `--dangerous-fs-tests`)
2. 프로파일 프리셋으로 선택지 축소 (`quick/safe/full/performance/security`) — 에이전트 정확도 향상
3. 안정적인 스키마 버전 + COMPATIBILITY 문서 — 업데이트에도 파이프라인이 깨지지 않음
4. 메시지 문자열이 아닌 **안정적인 `check_id`** 로 식별 — 문자열 매칭 의존 제거
5. 모든 타임아웃 제어 가능 — 에이전트 hang 방지
6. 사이드이펙트 격리 — private mount namespace, 랜덤 파일명, 자동 cleanup

**레벨 3 — 시스템 레벨 구현 학습**

- `fork()` + `pipe()` 워커 패턴(`client.c: parallel_child`) — 에이전트 샌드박싱 기본형
- `worker_deadline_sec()` + 시그널 핸들러 — 작업 데드라인 관리
- `serve.c` 121줄 미니 HTTP 서버 — 경량 메트릭 엔드포인트
- `validation.c` — 신뢰할 수 없는 입력 처리(= LLM 출력 검증과 동일한 문제)

### Q. React나 PHP로 만들 수 있는가?

**진단 엔진 자체는 불가능하고, 그 위의 제품 계층은 React/PHP가 적합합니다.**

불가능한 부분 (커널 영역):

| 기능 | 이유 |
|---|---|
| `mount()` 시스템콜 | 브라우저/PHP는 커널 마운트 불가 |
| RPC NULLPROC (`rpc.c`) | 원시 RPC/XDR + libtirpc. JS/PHP에 실용적 NFS RPC 라이브러리 없음 |
| `/proc/self/mountstats` 파싱 | 로컬 리눅스 프로세스가 읽어야 함 |
| 원시 TCP 소켓 / MTU 측정 | 브라우저 샌드박스에서 금지 |
| eBPF 커널 프로브 | C + libbpf + root 필수 |
| `setuid()` UID 시뮬레이션 | root 권한 + 프로세스 제어 |
| POSIX ACL / xattr / `fcntl` 락 | 저수준 시스템콜 |

가능한 계층 구조:

```
React 프론트엔드 (대시보드, 차트, 리포트 뷰어)
        ↓
PHP/Node 백엔드 (스케줄러, 멀티호스트, 알림, DB)
        ↓
nfsdiag 바이너리 (exec 후 --json 결과 수신)  ← MIT라 그대로 사용
```

```php
<?php
$host = escapeshellarg($validatedHost);   // 입력 이스케이프 필수
exec("nfsdiag client --profile safe --json - $host 2>/dev/null", $out, $code);
$report = json_decode(implode("\n", $out), true);
foreach ($report['events'] as $e) {
    if ($e['severity'] === 'fail') notify($e['message'], $e['remediation']);
}
```

가장 현실적인 첫 프로젝트: **JSON 리포트 뷰어(React 단독)** — 백엔드/root 권한 없이 정적 호스팅 가능하며, 원본 저장소에 기여할 여지도 있습니다.

---

## 6. 수익화 아이디어

> 전제: MIT 라이선스이므로 상업적 이용·수정·재배포가 자유롭습니다. 저작권 고지는 유지해야 하며, 파생 제품에 원작자 크레딧을 남기는 것이 권장됩니다.

### 아이디어 1 — NFS 헬스 모니터링 SaaS

- **컨셉**: 고객 서버의 경량 에이전트가 주기적으로 nfsdiag 실행 → 결과 JSON 수집 → 웹 대시보드 + 장애 예측 알림
- **근거**: `--hosts-file`, `--diff-baseline`, `--watch`, Prometheus 출력이 이미 있어 데이터 수집 계층이 완성돼 있음
- **가격 예시**: Free(3 hosts) / Pro $29월(25) / Business $149월(200) / Enterprise 연간+온프렘
- **타겟**: 미디어·VFX 렌더팜, HPC/연구소, 금융 배치, 레거시 ERP 운영사
- **리스크**: 온프렘 고객의 데이터 반출 거부 → 셀프호스트 버전 필수. 범용 APM과의 경쟁 → "NFS 특화 + 해결책 제시"로 차별화
- 난이도 ★★★ / 수익성 ★★★★★

### 아이디어 2 — AI NFS 트러블슈터 (MCP 서버 + LLM) ★ 최고 추천

- **컨셉**: nfsdiag를 MCP 서버로 래핑해 Claude/Cursor 등에 연결. "NFS가 왜 느려?" → 진단 실행 → JSON 해석 → 원인·해결책 제시
- **근거**: JSON에 `check_id`/`severity`/`remediation`이 이미 있어 **사실 기반 답변 → 환각 최소화**. 대부분의 AI 운영 도구가 로그 추측에 의존하는 것과 대비되는 차별점
- **모델**: MCP 서버는 오픈소스 공개(신뢰·유입), 유료는 Fleet 버전(다중 서버, 히스토리, 팀 공유, 승인 워크플로) $49~199/월 + 기업 컨설팅
- **확장성**: 첫 도구로 nfsdiag를 붙인 뒤 다른 진단 도구를 계속 추가하면 "인프라 진단 에이전트 플랫폼"으로 성장 가능
- 난이도 ★★ / 수익성 ★★★★

### 아이디어 3 — 스토리지 벤더 / MSP 지원 자동화

- **컨셉**: 벤더 기술지원팀이 고객에게 진단 번들을 요청 → `--output-dir` 증거 번들 수령 → `--root`로 오프라인 분석
- **근거**: `--output-dir`(JSON+HTML+증거+SHA256), `--audit-trail`, `--root DIR` 오프라인 분석이 지원 티켓 워크플로에 맞춰 설계돼 있음
- **가치 제안**: 티켓당 MTTR 단축 = 직접적 비용 절감 (ROI 산출이 쉬움)
- **가격**: 화이트라벨 라이선스 연 $5,000~$50,000, 장비 특화 플러그인은 프로젝트 과금
- 난이도 ★★★★(영업 난이도) / 수익성 ★★★★★

### 아이디어 4 — Kubernetes NFS 오퍼레이터 / CSI 헬스체크

- **컨셉**: DaemonSet으로 배포해 노드별 NFS PV 헬스체크 → Prometheus/Grafana → 마운트 실패 전 경보
- **근거**: K8s에서 NFS PV 장애는 파드 전체를 hang 시키는 치명적 사고. nfsdiag는 이미 `--ganesha-check`로 컨테이너/K8s 환경을 감지하고 Prometheus 출력을 내장
- **모델**: 오퍼레이터 오픈소스 + Managed/서포트 구독(Red Hat 모델), 마켓플레이스 등록
- 난이도 ★★★★ / 수익성 ★★★★

### 아이디어 5 — 교육 콘텐츠 / 정보 상품 (가장 빠른 현금화)

| 상품 | 가격대 | 제작 기간 |
|---|---|---|
| "NFS 트러블슈팅 완전정복" 온라인 강의 | 5.5만~9.9만원 | 1~2개월 |
| "C로 배우는 시스템 프로그래밍" (nfsdiag 코드 해부) | 9.9만원 | 2~3개월 |
| 기술 블로그 + 유튜브 | 간접 수익 | 지속 |
| 전자책 / 유료 뉴스레터 | 1.5만~3만원 | 2주~1개월 |

- **근거**: Docker 픽스처 14종이 그대로 **핸즈온 실습 환경**이 됨. 한국어 NFS 심화 콘텐츠는 희소
- 난이도 ★ / 수익성 ★★

### 아이디어 6 — 웹 SaaS "NFS 리포트 스튜디오"

- **무료**: JSON 리포트 드래그앤드롭 → 대시보드 렌더링
- **유료**: 리포트 히스토리, 팀 공유, PDF 내보내기, 시계열 추이, 화이트라벨(MSP 보고서용)
- **가격**: Free / Pro $9월 / Team $49월
- **장점**: React 프론트만으로 MVP 가능(백엔드·root 불필요), 정적 호스팅으로 무료 배포, 사용자 확보 후 아이디어 1로 확장
- 난이도 ★★ / 수익성 ★★★

### 추천 로드맵

```
1~2개월차 : 아이디어 6 (React 리포트 뷰어) — 무료 공개, 포트폴리오·스타 확보
2~4개월차 : 아이디어 2 (MCP 서버) — 오픈소스 공개, MCP 생태계 초기 선점
3~6개월차 : 아이디어 5 (교육 콘텐츠) — 위 경험을 그대로 강의화, 첫 현금 흐름
6개월차~  : 아이디어 6 유료화 → 아이디어 1 (본격 SaaS)
```

### 현실 체크

1. 엔진을 새로 만들지 말 것. MIT이므로 그대로 사용하고, 부가가치는 **UI/자동화/AI/통합**에서 창출
2. NFS는 니치 시장 — 폭발적 성장은 어렵지만 **고객당 단가가 높고 이탈률이 낮음**
3. 원작자 크레딧 명시 및 업스트림 기여 — 커뮤니티 평판이 곧 B2B 신뢰
4. GPL이 아닌 MIT라 클로즈드 소스 상용화도 법적으로 가능

---

## 7. 알려진 한계 (README 기준)

- `ESTALE`은 테스트 구간 내에 실제로 발생해야만 탐지됨 ("없음" = "재현 못 함")
- SELinux/AppArmor 문제가 일반 permission denied처럼 보일 수 있음
- ACL 정보는 NFS 클라이언트가 노출하는 범위에 의존
- 성능 수치는 스모크 테스트 값이며 용량 벤치마크가 아님
- Docker 픽스처는 호스트 커널과 Docker 권한에 의존
- `--listen` 익스포터는 인증·TLS 없음

---

## 8. 요약

`nfsdiag`는 NFS 장애 진단이라는 **명확한 고통 지점**을, **의존성이 거의 없는 단일 C 바이너리**로, **클라이언트·서버 양방향 + AI/모니터링 친화적 구조화 출력**과 함께 해결하는 도구입니다.
MIT 라이선스이므로 그대로 가져다 쓰고, 그 위에 **UI·자동화·AI 통합 계층을 얹는 것**이 가장 현실적인 활용 및 수익화 경로입니다.
