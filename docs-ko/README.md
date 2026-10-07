# Ansible 저장소 분석 및 활용 정리

> 이 문서는 `bmshin94/ansible` 저장소를 전수조사하여 분석한 결과와,
> 이를 바탕으로 한 활용 방안·수익화 아이디어를 정리한 문서입니다.

| 항목 | 내용 |
|---|---|
| **내 저장소 (GitHub)** | **https://github.com/bmshin94/ansible** |
| **원본 (Upstream)** | https://github.com/ansible/ansible |
| 공식 문서 | https://docs.ansible.com/ansible-core/devel/ |
| Galaxy 허브 | https://galaxy.ansible.com |
| 커뮤니티 포럼 | https://forum.ansible.com |
| 작성일 | 2026-10-07 |
| 분석 대상 브랜치 | `devel` (`21a07ef`) |

---

## 목차

1. [저장소 정체 — 전수조사 결과](#1-저장소-정체--전수조사-결과)
2. [Ansible이란 무엇인가](#2-ansible이란-무엇인가)
3. [폴더 구조 상세](#3-폴더-구조-상세)
4. [핵심 용어와 첫 실습](#4-핵심-용어와-첫-실습)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인? 스킬? MCP?](#6-플러그인-스킬-mcp)
7. [API 토큰이 필요한가](#7-api-토큰이-필요한가)
8. [AI 에이전트 구축에 도움이 되는가](#8-ai-에이전트-구축에-도움이-되는가)
9. [React / PHP로 만들 수 있는가](#9-react--php로-만들-수-있는가)
10. [유튜브 강의 제작 가능성](#10-유튜브-강의-제작-가능성)
11. [수익화 아이디어 10선](#11-수익화-아이디어-10선)
12. [라이선스 주의사항](#12-라이선스-주의사항)
13. [12개월 실행 로드맵](#13-12개월-실행-로드맵)
14. [참고 자료](#14-참고-자료)

---

## 1. 저장소 정체 — 전수조사 결과

**결론: 이것은 `ansible/ansible`(Ansible Core) 오픈소스 원본을 `bmshin94` 계정으로 복사(fork)한 것입니다.**
플러그인이나 작은 도구가 아니라, 전 세계 IT 인프라 자동화 표준 도구의 **소스코드 전체**입니다.

| 항목 | 값 |
|---|---|
| 패키지명 | `ansible-core` |
| 버전 | `2.23.0.dev0` (개발 중 최신) |
| 언어 | Python (3.13 이상 필수) |
| 전체 파일 | 5,836개 |
| Python 파일 | 1,854개 |
| 코드 라인 | 약 267,000줄 |
| 저장소 크기 | 48MB |
| 라이선스 | **GPL-3.0-or-later** |
| 브랜치 | `devel`(개발), `claude/sweet-davinci-daz7c4`(작업용) |
| 의존성 | jinja2, PyYAML, cryptography, packaging, resolvelib |
| 원본과의 차이 | 거의 없음. PR #1로 `CLAUDE.md` 가이드 문서만 추가 |

### 구성 비율

```
전체 5,836개 파일
├── 테스트       4,869개 (83%)  ← 코드보다 테스트가 5배 많음
└── 실제 코드      967개 (17%)
```

> **시사점**: 수만 대 서버를 동시에 다루는 도구이므로, 과도하다 싶을 만큼 테스트합니다.
> 테스트 작성법을 배울 최고의 교재이기도 합니다.

---

## 2. Ansible이란 무엇인가

### 한 줄 정의

> 수백 대의 서버에 필요한 설정과 프로그램을 **레시피 한 장으로 한 번에 자동 설치**해 주는 도구.

### 비유: 라면집 100개 체인점

| 상황 | 수동 방식 | Ansible 방식 |
|---|---|---|
| 레시피 전달 | 100개 지점 직접 방문 | 레시피 한 장 작성 (`playbook.yml`) |
| 실행 | 1년 소요 | 명령 한 줄, 수 분 |
| 오류 추적 | 어디가 틀렸는지 못 찾음 | 서버별 결과 자동 보고 |
| 신메뉴 추가 | 또 100번 방문 | 레시피에 한 줄 추가 |

### 설계 철학 3가지

| 철학 | 설명 | 경쟁 도구와의 차이 |
|---|---|---|
| **에이전트리스**<br>(Agentless) | 관리 대상 서버에 아무것도 설치하지 않음. 이미 있는 **SSH만** 사용 | Puppet/Chef는 서버마다 전용 에이전트 설치 필요 → Ansible이 이긴 결정적 이유 |
| **멱등성**<br>(Idempotency) | 같은 레시피를 100번 실행해도 결과가 동일. 이미 되어 있으면 건너뜀 | 일반 셸 스크립트는 중복 실행 시 망가짐 |
| **선언형**<br>(Declarative) | "어떻게"가 아니라 **"어떤 상태여야 하는지"**를 기술 | `state: present`(있어야 함), `state: started`(켜져 있어야 함) |

### 멱등성 체감 예시

```
"냉장고에 계란 10개 있게 해라"   ← Ansible 방식 (상태 선언)
"냉장고에 계란 10개 넣어라"      ← 일반 스크립트 (동작 명령)

→ Ansible: 100번 실행해도 계란은 항상 10개
→ 스크립트: 100번 실행하면 계란 1000개
```

---

## 3. 폴더 구조 상세

```
/home/user/ansible
├── bin/                 사용자가 입력하는 명령어 10개
├── lib/ansible/         엔진 본체 (두뇌)
├── test/                테스트 (전체의 83%)
├── context/             AI·개발자용 코딩 규칙 문서 14개
├── .claude/             Claude Code 전용 스킬 4개
├── hacking/             개발자용 보조 도구
├── changelogs/          변경 이력
├── .azure-pipelines/    CI 자동 테스트 설정
├── .github/             이슈·PR 템플릿, 보안정책
├── licenses/            외부 라이브러리 라이선스 5종
├── packaging/           배포 패키징
├── pyproject.toml       패키지 정의
├── requirements.txt     의존성
├── CLAUDE.md            프로젝트 개요 (PR #1로 추가)
└── AGENTS.md            AI 에이전트용 작업 지침
```

### 3.1 `bin/` — 명령어 10개

| 명령어 | 용도 | 사용 빈도 |
|---|---|---|
| `ansible-playbook` | YAML 레시피 실행 | ★★★★★ |
| `ansible` | 단발성 즉석 명령 | ★★★★ |
| `ansible-doc` | 모듈 사용법 조회 (내장 매뉴얼) | ★★★★ |
| `ansible-vault` | 비밀번호·API키 암호화 | ★★★★ |
| `ansible-galaxy` | 외부 롤·컬렉션 설치 (npm 같은 역할) | ★★★ |
| `ansible-inventory` | 서버 목록 확인·정리 | ★★★ |
| `ansible-config` | 설정 조회·덤프 | ★★ |
| `ansible-console` | 대화형 셸 모드 | ★ |
| `ansible-pull` | 역방향 모드 (서버가 스스로 git에서 당겨와 실행) | ★ |
| `ansible-test` | 개발자용 테스트 러너 | ★ (기여자용) |

### 3.2 `lib/ansible/modules/` — 실제 일을 하는 모듈 69개

| 분류 | 모듈 |
|---|---|
| 패키지 | `apt` `dnf` `dnf5` `yum_repository` `pip` `package` `apt_key` `apt_repository` `deb822_repository` `dpkg_selections` `rpm_key` `debconf` `package_facts` |
| 파일 | `copy` `file` `template` `lineinfile` `blockinfile` `replace` `fetch` `stat` `find` `unarchive` `assemble` `slurp` `tempfile` `mount_facts` |
| 서비스 | `service` `systemd` `systemd_service` `sysvinit` `reboot` |
| 사용자 | `user` `group` `known_hosts` `hostname` `getent` |
| 명령 실행 | `command` `shell` `script` `raw` `expect` `async_status` `async_wrapper` |
| 네트워크 | `uri` `get_url` `iptables` `wait_for` `wait_for_connection` |
| 소스코드 | `git` `subversion` |
| 제어 흐름 | `include_tasks` `import_tasks` `include_role` `import_role` `import_playbook` `include_vars` `set_fact` `set_stats` `meta` `pause` `debug` `assert` `fail` `validate_argument_spec` |
| 정보 수집 | `setup` `gather_facts` `service_facts` `ping` |
| 기타 | `cron` `add_host` `group_by` |

### 3.3 `lib/ansible/plugins/` — 확장 포인트 17종

| 플러그인 | 개수 | 역할 | 주요 항목 |
|---|---|---|---|
| **filter** | 77 | Jinja2 데이터 변환 | `to_json` `b64encode` `regex_replace` |
| **test** | 54 | 조건 검사 | `is file` `is match` `is truthy` |
| **action** | 29 | 모듈 실행 전처리 | — |
| **lookup** | 26 | 외부에서 값 가져오기 | `env` `file` `url` `password` `csvfile` `ini` `pipe` `template` `vars` `first_found` |
| **doc_fragments** | 20 | 문서 재사용 조각 | — |
| **inventory** | 10 | 서버 목록 소스 | `yaml` `ini` `toml` `script` `auto` `constructed` `generator` `host_list` |
| **callback** | 6 | 실행 결과 출력 형식 | `default` `minimal` `oneline` `tree` `junit` |
| **connection** | 5 | 접속 방법 | `ssh`(기본) `local` `winrm`(윈도우) `psrp` |
| **strategy** | 5 | 실행 순서 전략 | `linear`(기본) `free` `host_pinned` `debug` |
| **become** | 4 | 권한 상승 | `sudo` `su` `runas` |
| **shell** | 4 | 원격 셸 종류 | — |
| **cache** | 4 | 수집 정보 캐싱 | `jsonfile` `memory` |
| **vars** | 2 | 변수 소스 | — |
| **cliconf / netconf / terminal / httpapi** | 각 1 | 네트워크 장비(스위치·라우터) 제어 | — |

> **핵심**: Ansible은 "모든 것을 플러그인으로 교체 가능"하게 설계되어 있습니다.
> 직접 모듈·필터·커넥션을 만들어 끼울 수 있습니다.

### 3.4 `lib/ansible/` — 나머지 핵심 디렉터리

| 경로 | 역할 |
|---|---|
| `cli/` | 명령줄 파싱 (`adhoc.py` `playbook.py` `vault.py`) |
| `executor/` | **실행 엔진.** `task_queue_manager.py`(병렬), `play_iterator.py`(순서), `task_executor.py`(개별 실행), `module_common.py`(모듈 전송), `interpreter_discovery.py`(원격 파이썬 탐지) |
| `parsing/` | YAML 파싱 + Vault 암복호화 |
| `template/` | Jinja2 템플릿 엔진 연동 |
| `module_utils/` | 모듈 공유 라이브러리. **`basic.py`**(모든 모듈의 기반 클래스), `urls.py`, `facts/` |
| `galaxy/` | Galaxy API 클라이언트, 의존성 해결기(`dependency_resolution/`), `token.py` |
| `inventory/` `vars/` `playbook/` | 서버 목록, 변수 우선순위, 플레이북 객체 모델 |

### 3.5 실행 흐름 (`ansible-playbook deploy.yml`을 치면)

```
 1. cli/playbook.py            명령줄 옵션 해석
 2. parsing/                   YAML 읽기 + Vault 복호화
 3. inventory/                 서버 목록 파악
 4. executor/playbook_executor 플레이북 실행 시작
 5. modules/setup.py           각 서버 정보(팩트) 수집
 6. executor/play_iterator     태스크 순서 결정
 7. plugins/strategy/linear    실행 전략 적용
 8. executor/task_queue_manager 병렬 전송 (기본 5대씩)
 9. template/ (Jinja2)         변수 치환
10. plugins/connection/ssh.py  SSH 접속
11. executor/module_common.py  ★ 모듈 코드를 압축해 원격 전송
12. [원격 서버]                 파이썬으로 실행 → JSON 반환 → 임시파일 삭제
13. plugins/callback/default.py 결과 출력 (ok / changed / failed)
```

> **11번이 Ansible의 핵심 마법입니다.** 원격 서버에 Ansible을 설치하지 않습니다.
> 필요한 파이썬 코드만 그때그때 보내서 실행하고 지웁니다. 그래서 "에이전트리스"입니다.

### 3.6 `test/` — 테스트 구성

| 경로 | 파일 수 | 내용 |
|---|---|---|
| `test/integration/` | 3,833 | 실제 컨테이너(Ubuntu/Fedora/RHEL)를 띄워 진짜 설치해 보는 통합 테스트 |
| `test/units/` | 628 | 함수 단위 테스트 (pytest) |
| `test/lib/ansible_test/` | 299 | `ansible-test` 도구 자체 코드 |
| `test/sanity/` | 57 | 코드 품질 검사 (`black` `mypy` `codespell` `pymarkdown` `pylint`) |
| `test/support/` | 52 | 테스트 보조 |

### 3.7 `.claude/` — Claude Code 전용 스킬 4개

Ansible 프로젝트가 **공식적으로** AI 협업을 전제하고 넣어둔 스킬입니다.

| 스킬 | 호출 | 기능 |
|---|---|---|
| `context` | `/context` | `AGENTS.md`를 읽어 개발 규칙을 AI 컨텍스트에 로드 |
| `review` | `/review <PR번호>` | PR을 프로젝트 표준 7단계로 자동 리뷰 |
| `azp-logs` | `/azp-logs <PR번호>` | Azure CI 실패 로그 자동 다운로드 (5~10분, 실행 전 사용자 확인 필수) |
| `creating-backports` | `/creating-backports` | 머지된 PR을 안정 브랜치로 백포트 (6단계) |

#### 가장 배울 점: 권한 최소화

```yaml
---
name: review
description: Review an Ansible PR following the project's standardized process
argument-hint: <pr_number>
allowed-tools: [TodoWrite, Bash(gh pr view:*), Bash(gh pr diff:*),
                Bash(gh pr checkout:*), Bash(gh pr checks:*),
                Read, Grep, Glob, Search]
user-invocable: true
---
```

> `Bash` 전체가 아니라 **`Bash(gh pr view:*)`** — 명령 단위 화이트리스트입니다.
> AI가 실수로 `rm -rf`를 실행할 수 없습니다. **AI 에이전트 보안 설계의 핵심 패턴**입니다.

### 3.8 `context/` — AI가 읽을 것을 전제로 쓴 규칙서 14개

```
README.md                  ci.md                    code-structure.md
coding-style.md            contributing.md          data-tagging.md
deprecation.md             dev-environment.md       documentation-standards.md
error-handling.md          licensing.md             public-api.md
running-tests.md           writing-tests.md
```

### 3.9 `AGENTS.md`의 주요 규칙

| 규칙 | 원문 취지 |
|---|---|
| 라이선스 | "NEVER suggest, recommend, or approve code that violates the project's licensing requirements. This is non-negotiable." (GPLv3 / BSD-2-Clause만 허용, 협상 불가) |
| 리뷰 범위 | "Don't flag issues that `ansible-test sanity` already catches. Focus review effort on things automated checks can't verify." |
| AI 기여 공개 | "Agents should disclose their involvement. We recommend using an `Assisted-by:` commit trailer." |
| 인간 저작 명시 | README 마지막 줄: "This project is substantially coded by humans." |

---

## 4. 핵심 용어와 첫 실습

### 4.1 용어 6개

| 용어 | 비유 | 의미 | 실제 파일 |
|---|---|---|---|
| **인벤토리** (Inventory) | 지점 주소록 | 관리할 서버 목록 | `hosts.ini` |
| **플레이북** (Playbook) | 레시피 카드 | 작업 순서 YAML | `deploy.yml` |
| **태스크** (Task) | 레시피 한 줄 | 작업 하나 | "nginx 설치" |
| **모듈** (Module) | 주방 도구 | 실제 일을 하는 프로그램 | `apt` `copy` `service` |
| **롤** (Role) | 레시피 묶음집 | 재사용 가능한 작업 세트 | `roles/webserver/` |
| **팩트** (Facts) | 지점 현황 보고서 | 서버에서 자동 수집한 정보 | OS, 메모리, IP |

### 4.2 인벤토리 예시

```ini
# hosts.ini
[local]
localhost ansible_connection=local

[webservers]
web1 ansible_host=203.0.113.10 ansible_user=ubuntu
web2 ansible_host=203.0.113.11 ansible_user=ubuntu

[dbservers]
db1 ansible_host=203.0.113.20 ansible_user=ubuntu

[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_become=yes
ansible_become_method=sudo
```

### 4.3 플레이북 예시

```yaml
# deploy.yml
- name: 웹서버 세팅
  hosts: webservers
  become: yes

  tasks:
    - name: nginx 설치
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: 설정 파일 배포
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: nginx 재시작        # 실제로 바뀌었을 때만 핸들러 호출

  handlers:
    - name: nginx 재시작
      ansible.builtin.service:
        name: nginx
        state: restarted
```

### 4.4 실행과 결과

```bash
ansible-playbook -i hosts.ini deploy.yml
```

```
PLAY [웹서버 세팅] ****************************

TASK [nginx 설치] ***************************
ok: [web1]              ← 이미 설치됨 (아무것도 안 함)
changed: [web2]         ← 새로 설치함

TASK [설정 파일 배포] *************************
changed: [web1]
changed: [web2]

RUNNING HANDLER [nginx 재시작] ****************
changed: [web1]
changed: [web2]

PLAY RECAP **********************************
web1 : ok=3  changed=2  unreachable=0  failed=0
web2 : ok=3  changed=3  unreachable=0  failed=0
```

> `ok` = 이미 올바른 상태 / `changed` = 바꿨음 / `failed` = 실패
> **두 번째로 실행하면 전부 `ok`가 됩니다. 이것이 멱등성입니다.**

---

## 5. 설치 및 사용법

### 5.1 일반 사용자 (99%가 여기에 해당)

> 이 저장소를 clone할 필요 없습니다. pip으로 설치하세요.

```bash
# 가상환경 (권장)
python3 -m venv ~/ansible-env
source ~/ansible-env/bin/activate

# 풀세트 — 대부분 이것을 선택
pip install ansible
#  → ansible-core + 커뮤니티 컬렉션 수천 개 (AWS, Azure, Docker, K8s, MySQL)

# 또는 코어만 (가볍게)
pip install ansible-core
#  → 이 저장소의 모듈 69개만

ansible --version
```

**OS 패키지 매니저**

```bash
sudo apt install ansible          # Ubuntu / Debian
sudo dnf install ansible          # RHEL / Rocky / Fedora
brew install ansible              # macOS
wsl --install -d Ubuntu           # Windows → WSL2 설치 후 위 방법
```

> **중요**: Ansible을 **실행하는** 컴퓨터는 Linux/macOS여야 합니다 (Windows는 WSL2로 우회).
> 관리 **대상**은 Windows도 가능합니다 (`winrm` / `psrp` 커넥션).

### 5.2 개발자 — 이 저장소로 직접 실행 (기여 목적)

```bash
git clone https://github.com/bmshin94/ansible.git
cd ansible

# 원본 연결 (최신 코드 동기화용)
git remote add upstream https://github.com/ansible/ansible.git

pip install -r requirements.txt

# 개발 모드 설치 (권장) — 코드 수정이 즉시 반영됨
pip install -e .

# 또는 환경변수 방식 (설치 없이)
source ./hacking/env-setup          # bash / zsh
source ./hacking/env-setup.fish     # fish

ansible --version                   # → 2.23.0.dev0
```

**테스트 실행**

```bash
ansible-test sanity      --docker default     # 코드 스타일·타입 검사
ansible-test units       --docker default     # 단위 테스트
ansible-test integration ping --docker default # 통합 테스트
```

### 5.3 10분 실습

```bash
mkdir ~/ansible-practice && cd ~/ansible-practice

# 1) 인벤토리
cat > hosts.ini << 'EOF'
[local]
localhost ansible_connection=local
EOF

# 2) 연결 테스트 → "pong"이 나오면 성공
ansible local -i hosts.ini -m ping

# 3) 서버 정보 보기
ansible local -i hosts.ini -m setup -a "filter=ansible_distribution*"
```

```yaml
# 4) first.yml
- name: 첫 번째 플레이북
  hosts: local
  vars:
    my_message: "안녕하세요, Ansible!"

  tasks:
    - name: 메시지 출력
      ansible.builtin.debug:
        msg: "{{ my_message }}"

    - name: 서버 정보 출력
      ansible.builtin.debug:
        msg: "OS {{ ansible_distribution }} {{ ansible_distribution_version }}, 메모리 {{ ansible_memtotal_mb }}MB"

    - name: 파일 생성
      ansible.builtin.copy:
        content: "Ansible이 만든 파일입니다. 생성: {{ ansible_date_time.iso8601 }}\n"
        dest: /tmp/ansible-test.txt
        mode: '0644'
```

```bash
# 5) 모의 실행 → 실제 실행 → 한 번 더 (멱등성 확인)
ansible-playbook -i hosts.ini first.yml --check --diff
ansible-playbook -i hosts.ini first.yml
ansible-playbook -i hosts.ini first.yml      # 전부 ok로 바뀜
```

### 5.4 실무 프로젝트 구조

```
my-infra/
├── ansible.cfg
├── inventories/
│   ├── dev/hosts.ini
│   ├── staging/hosts.ini
│   └── prod/hosts.ini
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   └── vault.yml            # 암호화된 비밀 (ansible-vault)
├── host_vars/
│   └── web1.yml
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   ├── files/
│   │   ├── vars/main.yml
│   │   ├── defaults/main.yml
│   │   └── meta/main.yml
│   ├── nginx/
│   └── nodejs/
├── site.yml
└── requirements.yml
```

```ini
# ansible.cfg
[defaults]
inventory = inventories/dev/hosts.ini
roles_path = roles
host_key_checking = False
forks = 20
stdout_callback = yaml

[ssh_connection]
pipelining = True              # 속도 대폭 향상
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### 5.5 명령어 치트시트

```bash
# ── 조회 ──
ansible-doc -l                              # 모듈 전체 목록
ansible-doc copy                            # 모듈 사용법
ansible-inventory --list -i hosts.ini       # 인벤토리 확인
ansible-config dump --only-changed          # 변경된 설정만

# ── 실행 ──
ansible-playbook site.yml --check --diff    # 모의 실행 (실무 필수 습관)
ansible-playbook site.yml --limit web1      # 특정 서버만
ansible-playbook site.yml --tags deploy     # 특정 태그만
ansible-playbook site.yml --skip-tags slow  # 특정 태그 제외
ansible-playbook site.yml --start-at-task "nginx 설치"
ansible-playbook site.yml -e "version=1.2.3"
ansible-playbook site.yml -f 50             # 동시 50대
ansible-playbook site.yml --syntax-check    # 문법만 검사
ansible-playbook site.yml -vvv              # 상세 디버그

# ── 비밀 관리 ──
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt file.yml
ansible-vault rekey secrets.yml
ansible-vault encrypt_string 'mySecret' --name 'db_password'

# ── 외부 컬렉션·롤 ──
ansible-galaxy collection install community.docker
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy install -r requirements.yml
ansible-galaxy init my_role
```

---

## 6. 플러그인? 스킬? MCP?

### 결론: 셋 다 아닙니다. **독립 실행형 CLI 애플리케이션**입니다.

```
이 저장소 = Ansible Core
정체: 독립 실행형 Python CLI 애플리케이션

├── 내부에 "플러그인 시스템"을 가지고 있음 (17종)
│     → 플러그인을 "받는 호스트"이지, 플러그인 자체가 아님
├── .claude/skills/ 에 Claude Code 스킬 4개 포함
│     → 개발자 편의용 부가물. Ansible의 본질이 아님
└── MCP 서버가 아님 (전수조사 결과 MCP 관련 파일 0개)
```

### 비교표

| 구분 | 정의 | 혼자 실행? | Ansible은 |
|---|---|---|---|
| **독립 CLI 앱** | 터미널에서 직접 실행 | 예 | **이것입니다** |
| 플러그인 | 호스트 프로그램의 확장 | 아니오 | 플러그인을 *받는* 쪽 |
| 스킬 (Claude) | Claude Code 작업 지침서(.md) | 아니오 | `.claude/`에 4개 *포함* |
| MCP 서버 | AI에게 도구를 제공하는 프로토콜 서버 | 아니오 | **아님** (만들 수는 있음) |

### 헷갈리는 이유 3가지

1. **"Ansible 플러그인"이라는 말** — Ansible 안에 끼워 넣는 확장을 뜻합니다. "워드프레스 플러그인"이 있다고 워드프레스가 플러그인은 아닌 것과 같습니다.
2. **`.claude/` 폴더** — Ansible 개발을 AI로 돕기 위한 도구입니다. 자동차 공장에 로봇 팔이 있다고 자동차가 로봇은 아닙니다.
3. **컬렉션 설치** — `ansible-galaxy collection install`은 Ansible이 자기 안에 모듈 묶음을 설치하는 것입니다 (npm과 동일).

### MCP 서버로 감쌀 수 있습니다 (사업 기회)

```
┌──────────────┐   MCP   ┌──────────────┐   SSH   ┌─────────┐
│ Claude Code  │ ←────→  │ ansible-mcp  │ ──────→ │ 서버 100대│
│ Cursor       │         │ (직접 제작)   │         │          │
└──────────────┘         └──────────────┘         └─────────┘
```

> 상세 구현은 [11장 아이디어 1](#아이디어-1-ansible-mcp-서버-ai--인프라) 참고.

---

## 7. API 토큰이 필요한가

### 결론: **Ansible 자체는 토큰 없이 작동합니다. 인증은 SSH 키입니다.**

### 7.1 토큰이 전혀 필요 없는 경우 (기본 사용)

```bash
ansible all -m ping                  # 연결 테스트
ansible-playbook deploy.yml          # 서버 설정·배포
ansible-doc copy                     # 문서 조회
ansible-inventory --list             # 인벤토리 확인
ansible-vault create secrets.yml     # 비밀 암호화
```

**SSH 키 설정**

```bash
ssh-keygen -t ed25519 -C "ansible@mylaptop"
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@203.0.113.10
ansible all -i hosts.ini -m ping
```

### 7.2 토큰이 필요한 경우

| 상황 | 토큰 | 비고 |
|---|---|---|
| Galaxy에서 컬렉션 **설치** | ❌ 불필요 | — |
| Galaxy에 내 컬렉션 **업로드** | ✅ Galaxy API Token | `~/.ansible/galaxy_token`. 관련 코드: `lib/ansible/galaxy/token.py` |
| AWS 모듈 | ✅ Access Key 또는 IAM Role | Ansible 토큰이 아니라 **AWS 자격증명** |
| Azure 모듈 | ✅ Service Principal | `~/.azure/credentials` |
| GCP 모듈 | ✅ 서비스 계정 JSON | `GOOGLE_APPLICATION_CREDENTIALS` |
| GitHub / Slack / Docker 모듈 | ✅ 각 서비스 토큰 | 모듈 파라미터 |
| Kubernetes 모듈 | ✅ kubeconfig 또는 서비스 토큰 | — |
| 이 저장소에 PR 기여 | ✅ GitHub Token | `gh auth login` |
| AAP / AWX (유료 제품) | ✅ OAuth 토큰 | 오픈소스 `ansible-core`와 별개 |
| **AI 토큰 (OpenAI/Claude)** | ❌ | Ansible은 AI와 무관. 연동을 직접 만들면 그때 필요 |

> **정리**: 토큰은 "Ansible이 다른 서비스를 조종할 때 그 서비스가 요구하는 것"일 뿐,
> Ansible 자체가 요구하는 것이 아닙니다.

### 7.3 Ansible Vault — 비밀을 안전하게 다루기

```bash
ansible-vault create group_vars/all/vault.yml
```

```yaml
vault_db_password: "SuperSecret123!"
vault_aws_access_key: "AKIAIOSFODNN7EXAMPLE"
vault_github_token: "ghp_xxxxxxxxxxxxxxxxxxxx"
```

저장하면 파일이 AES256으로 암호화됩니다.

```
$ANSIBLE_VAULT;1.1;AES256
3336383634393066623738666362623365396430...
```

> **이 상태로 GitHub에 커밋해도 안전합니다.**

```yaml
# 플레이북에서는 평범하게 사용 (자동 복호화)
- name: DB 사용자 생성
  community.mysql.mysql_user:
    name: appuser
    password: "{{ vault_db_password }}"
    state: present
```

```bash
ansible-playbook site.yml --ask-vault-pass                      # 암호 입력
ansible-playbook site.yml --vault-password-file ~/.vault_pass   # CI/CD용
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass                # 환경변수
```

---

## 8. AI 에이전트 구축에 도움이 되는가

### 결론: **매우 큽니다. 두 가지 층위에서.**

```
층위 1: Ansible을 "AI의 손발"로 쓴다        → 즉각적, 실용적
층위 2: Ansible의 설계를 "AI 설계도"로 쓴다  → 근본적, 더 가치 있음
```

### 8.1 층위 1 — AI의 실행 엔진으로

AI 에이전트의 최대 난관은 "AI가 생성한 명령을 어떻게 안전하게 실행하나"입니다.

```
위험한 방식:
AI → "rm -rf /var/log/*" 생성 → bash로 바로 실행 → 복구 불가

Ansible 방식:
AI → playbook.yml 생성 → --check 모의실행 → 사람이 diff 확인 → 승인 → 실행
```

| AI 에이전트의 문제 | Ansible의 해결책 |
|---|---|
| 같은 작업 중복 실행 | **멱등성** — 몇 번 해도 결과 동일 |
| 실행 전 결과를 모름 | **`--check --diff`** — 모의 실행 + 변경 미리보기 |
| 실패 시 중간에 멈춤 | **`register` + `failed_when` + `block/rescue`** |
| 100대 동시 적용 | **`forks`** — 병렬 실행 내장 |
| 비밀번호가 AI 컨텍스트에 노출 | **Vault** — 플레이북엔 `{{ vault_x }}`만 |
| 작업 기록이 없음 | **callback 플러그인** — JSON/JUnit 로그 |
| 결과 파싱이 어려움 | **모든 모듈이 JSON 반환** |

**실전 아키텍처**

```
사용자: "웹서버들 nginx를 1.25로 업그레이드해줘"
   ↓
LLM: 현재 상태 조회 → 플레이북 생성 → 모의 실행
   ↓
사람에게 제시: "web1: 1.24→1.25 (changed) / web2: 이미 1.25 (ok)
               설정파일 3줄 변경 예정. 계속할까요?"
   ↓ 승인
실제 실행 → JSON 결과를 LLM에 반환
   ↓
LLM: "완료. web1 업그레이드됨, web2는 이미 최신."
```

### 8.2 층위 2 — 설계 교과서로 (더 중요)

#### 패턴 1: 도구 정의 = `argument_spec`

```python
# lib/ansible/modules/ping.py — 세상에서 가장 단순한 "도구" 정의
def main():
    module = AnsibleModule(
        argument_spec=dict(
            data=dict(type='str', default='pong'),   # 입력 스키마 선언
        ),
        supports_check_mode=True                      # dry-run 지원 선언
    )
    result = dict(ping=module.params['data'])         # 작업 수행
    module.exit_json(**result)                        # JSON 결과 반환
```

→ LLM 도구 스키마로 변환하면:

```json
{
  "name": "ping",
  "description": "호스트 연결 및 Python 사용 가능 여부를 확인하고 pong을 반환합니다",
  "input_schema": {
    "type": "object",
    "properties": {
      "data": { "type": "string", "default": "pong" }
    }
  }
}
```

> **구조가 1:1로 대응합니다.** `입력 스키마 선언 → 검증 → 실행 → JSON 반환`
> 이것이 LLM 도구 호출(tool calling)의 구조와 완전히 동일합니다.

#### `argument_spec`이 제공하는 검증 기능 9가지

| 기능 | 의미 | LLM 도구에서의 가치 |
|---|---|---|
| `type=` | 타입 강제 변환·검증 | 문자열로 준 값을 안전하게 변환 |
| `required=True` | 필수 파라미터 | 빼먹으면 즉시 거부 |
| `choices=[...]` | 허용 값 목록 | **환각(hallucination) 차단** |
| `default=` | 기본값 | 모든 값을 줄 필요 없음 |
| **`no_log=True`** | **로그·출력에서 마스킹** | **비밀값이 AI 로그에 안 남음** |
| `mutually_exclusive` | 상호 배타 | 모순된 요청 거부 |
| `required_together` | 함께 필수 | 반쪽 요청 거부 |
| `required_if` | 조건부 필수 | 맥락 의존 검증 |
| `aliases=` | 별칭 허용 | 다른 이름으로 불러도 동작 |

> AI 도구를 만들 때 이 9가지를 전부 구현해야 합니다.
> **`lib/ansible/module_utils/basic.py` 한 파일에 전부 있습니다.**

#### 패턴 2: 자동 문서 = `DOCUMENTATION` 블록

```yaml
DOCUMENTATION = """
module: ping
short_description: Try to connect to host, verify a usable python and return pong
description:
  - A trivial test module, this module always returns pong on successful contact.
options:
  data:
    description:
      - Data to return for the RV(ping) return value.
      - If this parameter is set to V(crash), the module will cause an exception.
    type: str
    default: pong
attributes:
    check_mode:
        support: full        # dry-run 완전 지원 (안전 메타데이터)
"""
```

```bash
ansible-doc --json -l        # 모든 모듈 문서를 JSON으로 덤프
ansible-doc --json copy      # → LLM 도구 정의 자동 생성 가능
```

> **Ansible 모듈 69개(+컬렉션 수천 개) = 즉시 LLM 도구로 변환 가능한 자산.**

#### 패턴 3~6

| 패턴 | 위치 | AI 에이전트 적용 |
|---|---|---|
| **권한 최소화** `allowed-tools` | `.claude/skills/*/SKILL.md` | 에이전트 권한 화이트리스트 |
| **에이전트 규칙 문서** | `context/*.md` 14개 | 에이전트 온보딩 문서 템플릿 |
| **실행 전략** `strategy` 플러그인 | `plugins/strategy/` | `linear`=단계 동기화, `free`=병렬, `debug`=**실패 시 사람 개입(HITL)** |
| **에러 복구** `block/rescue/always` | 플레이북 문법 | try/catch/finally의 인프라 버전 |

```yaml
# block/rescue/always — AI 에이전트에 필수인 실패 복구 패턴
- name: 위험한 작업
  block:
    - name: DB 마이그레이션
      ansible.builtin.command: php artisan migrate
  rescue:
    - name: 실패 시 롤백
      ansible.builtin.command: php artisan migrate:rollback
    - name: Slack 알림
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body: '{"text":"마이그레이션 실패, 롤백했습니다"}'
  always:
    - name: 항상 실행 (유지보수 모드 해제)
      ansible.builtin.command: php artisan up
```

### 8.3 종합 매핑표

| Ansible 개념 | 코드 위치 | AI 에이전트에서 | 중요도 |
|---|---|---|---|
| `argument_spec` | `module_utils/basic.py` | 도구 입력 스키마 + 검증 | ★★★ |
| `DOCUMENTATION` | 각 모듈 상단 | 도구 설명(description) | ★★★ |
| `--check` / `check_mode` | 전 모듈 | 실행 전 미리보기 | ★★★ |
| 멱등성 | 설계 철학 | 재시도 안전성 | ★★★ |
| `no_log=True` | `argument_spec` | 비밀값 마스킹 | ★★★ |
| `allowed-tools` | `.claude/skills/*` | 권한 화이트리스트 | ★★★ |
| `context/*.md` | `context/` | 에이전트 규칙 문서 | ★★ |
| `block/rescue/always` | 플레이북 문법 | 에러 복구·롤백 | ★★ |
| `register` + `when` | 플레이북 문법 | 조건부 실행 | ★★ |
| `strategy` 플러그인 | `plugins/strategy/` | 실행 순서·병렬 전략 | ★★ |
| `callback` 플러그인 | `plugins/callback/` | 관측·로깅 | ★★ |
| Vault | `parsing/vault/` | 비밀 관리 | ★★ |
| `plugins/loader.py` | `plugins/loader.py` | 동적 도구 등록 | ★ |
| `become` | `plugins/become/` | 권한 상승 경계 | ★ |
| `gather_facts` | `modules/setup.py` | 환경 컨텍스트 자동 수집 | ★ |
| 83% 테스트 비율 | `test/` | 에이전트 신뢰성 평가(eval) | ★ |

### 8.4 한계 (솔직하게)

| 한계 | 설명 |
|---|---|
| AI 기능이 전혀 없음 | LLM 호출 모듈도 없습니다. 연동은 직접 만들어야 합니다 |
| YAML은 LLM이 틀리기 쉬움 | 들여쓰기 오류가 잦습니다. 생성 후 `--syntax-check` 필수 |
| 실시간 대화형엔 부적합 | 플레이북 실행은 수초~수분. 챗봇 응답용으론 느립니다 |
| 상태 저장 없음 | Terraform과 달리 state 파일이 없습니다 |
| 학습 데이터 편향 | LLM이 구버전 문법(`ansible.builtin.` 누락)을 생성하는 경우가 많습니다 |

---

## 9. React / PHP로 만들 수 있는가

### 결론

| 질문 | 답 |
|---|---|
| Ansible을 React/PHP로 **다시 만들기** | ❌ 비현실적 |
| **Ansible을 쓰는 제품**을 React/PHP로 만들기 | ✅ **완벽하게 가능. 이게 정답** |

### 9.1 재구현이 비현실적인 이유

| 이유 | 설명 |
|---|---|
| 규모 | 26.7만 줄 + 테스트 4,869개. 혼자서 몇 년 |
| 생태계 종속 | 모듈 69개 + 커뮤니티 컬렉션 **수천 개**가 모두 Python. 재구현하면 이 자산을 전부 버림 |
| 원격 실행 방식 | Ansible은 **Python 코드를 원격 서버에 전송해 실행**(`executor/module_common.py`). JS/PHP는 서버에 런타임이 없으면 실행 불가. Python은 거의 모든 Linux에 기본 탑재 |
| 의미 없음 | Node 생태계엔 이미 Pulumi 등 대안 존재 |

### 9.2 올바른 아키텍처

```
┌──────────────┐  HTTP  ┌──────────────┐  exec  ┌──────────┐
│ 프론트엔드     │ ────→  │ 백엔드 (API)  │ ────→  │ ansible  │
│ React / Vue  │ ←────  │ PHP / Node /  │ ←────  │ (CLI)    │
│              │  JSON  │ Python        │  JSON  │ → SSH    │
└──────────────┘        └──────────────┘        └──────────┘
                                                 GPL 안전
                                        (코드를 섞지 않고 "호출")
```

> **중요**: 이 방식은 GPL 전염이 없습니다. Ansible을 **별도 프로그램으로 실행**하기 때문입니다.

### 9.3 만들 수 있는 제품 7가지

| # | 제품 | 핵심 기술 |
|---|---|---|
| 1 | **웹 실행 대시보드** (가장 현실적) | React + SSE 실시간 로그 + Laravel Process |
| 2 | **플레이북 비주얼 빌더** (드래그앤드롭) | React Flow + js-yaml |
| 3 | **AI 플레이북 생성기** | LLM + `--syntax-check` 피드백 루프 |
| 4 | **인벤토리 관리 CMS** | Laravel + 동적 인벤토리 JSON API |
| 5 | **모니터링 대시보드** | `ansible all -m setup --tree` + Recharts |
| 6 | **컴플라이언스 감사 리포트** | 점검 플레이북 + PDF 생성 (고단가) |
| 7 | **교육용 플레이그라운드** | Monaco Editor + xterm.js + Docker |

#### 예시: Laravel 백엔드 (모의 실행)

```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Symfony\Component\Process\Process;

class PlaybookController extends Controller
{
    public function dryRun(Request $request)
    {
        $validated = $request->validate([
            'playbook'  => ['required', 'regex:/^[\w\-]+\.ya?ml$/'],   // 화이트리스트
            'inventory' => ['required', 'regex:/^[\w\-]+$/'],
            'limit'     => ['nullable', 'regex:/^[\w\-\.,:]+$/'],
        ]);

        $cmd = [
            'ansible-playbook',
            '-i', base_path("ansible/inventories/{$validated['inventory']}/hosts.ini"),
            base_path("ansible/playbooks/{$validated['playbook']}"),
            '--check', '--diff',
        ];

        $process = new Process($cmd, base_path('ansible'), [
            'ANSIBLE_STDOUT_CALLBACK'     => 'json',
            'ANSIBLE_VAULT_PASSWORD_FILE' => storage_path('vault_pass'),
        ]);
        $process->setTimeout(600);
        $process->run();

        return response()->json([
            'success' => $process->isSuccessful(),
            'output'  => $process->getOutput(),
            'errors'  => $process->getErrorOutput(),
        ]);
    }
}
```

#### 예시: 동적 인벤토리 API (PHP ↔ Ansible 연동)

```php
<?php
// Ansible이 이 JSON을 인벤토리로 직접 읽습니다
Route::get('/api/ansible-inventory', function () {
    $servers = Server::with('groups')->where('active', true)->get();
    $inventory = ['_meta' => ['hostvars' => []]];

    foreach ($servers as $s) {
        foreach ($s->groups as $g) {
            $inventory[$g->name]['hosts'][] = $s->hostname;
        }
        $inventory['_meta']['hostvars'][$s->hostname] = [
            'ansible_host' => $s->ip_address,
            'ansible_user' => $s->ssh_user,
            'server_role'  => $s->role,
        ];
    }
    return response()->json($inventory);
});
```

```bash
#!/bin/bash
# my_inventory.sh — Ansible 동적 인벤토리 스크립트
curl -s -H "Authorization: Bearer $MY_API_TOKEN" \
     https://my-cms.example.com/api/ansible-inventory
```

```bash
ansible-playbook -i ./my_inventory.sh site.yml
```

> `lib/ansible/plugins/inventory/script.py`가 이 JSON을 읽어줍니다. **PHP와 Ansible이 완벽히 연동됩니다.**

### 9.4 보안 수칙 10개 (절대 타협 불가)

웹에서 Ansible을 실행하면 본질적으로 "원격 코드 실행"입니다.

| # | 수칙 | 나쁜 예 | 좋은 예 |
|---|---|---|---|
| 1 | 명령 인젝션 방지 | `shell_exec("ansible-playbook $input")` | `new Process(['ansible-playbook', $validated])` |
| 2 | 파일명 화이트리스트 | 사용자 입력 경로 그대로 | DB에 등록된 ID만 허용 |
| 3 | `--check` 우선 | 바로 실제 실행 | 모의 실행 → 확인 → 실행 |
| 4 | Vault 암호 분리 | 웹 요청에 암호 포함 | 서버 파일 `chmod 600` |
| 5 | 권한 분리 (RBAC) | 로그인만 하면 다 실행 | 역할별 플레이북 권한 |
| 6 | 실행 큐 사용 | 웹 요청에서 직접 실행 | Redis/DB 큐 + 워커 |
| 7 | 감사 로그 | 기록 없음 | 누가/언제/무엇을/결과 전부 |
| 8 | 전용 계정 격리 | root로 실행 | `ansible` 전용 유저 + 제한된 sudo |
| 9 | 타임아웃·리소스 제한 | 무제한 | `setTimeout()` + 동시 실행 수 제한 |
| 10 | SSH 키 보호 | 웹 루트에 저장 | 웹 루트 밖 + `chmod 600` |

### 9.5 추천 스택

```
프론트엔드: React + TypeScript + Tailwind + shadcn/ui
            + Monaco Editor (코드) + xterm.js (터미널)
백엔드:     Laravel (PHP) 또는 FastAPI (Python)
큐:         Redis + Laravel Queue / Celery
DB:         PostgreSQL
배포:       Docker Compose
```

---

## 10. 유튜브 강의 제작 가능성

### 결론: **매우 유망. 한국어 시장은 공백에 가깝습니다.**

### 10.1 시장 분석

| 항목 | 현황 | 평가 |
|---|---|---|
| 영어권 강의 | 포화 (Udemy 수십 개, YouTube 수백 개) | 경쟁 과열 |
| **한국어 강의** | **거의 없음.** 체계적 시리즈는 희귀 | **블루오션** |
| 수요 | DevOps/SRE/인프라 채용 필수 스킬 | 꾸준함 |
| 시청자 구매력 | 현업 엔지니어 → 유료 전환율 높음, 회사 교육비 사용 가능 | 수익성 좋음 |
| 콘텐츠 수명 | 문법이 안정적 | **3~5년 유효 (롱테일)** |
| 차별화 소재 | **"Ansible + AI 에이전트"** | 전 세계적으로도 거의 없음 |

### 10.2 추천 커리큘럼 (3시즌 42편)

**시즌 1: 입문 — "서버 100대를 명령 한 줄로" (12편)**

| # | 제목 | 길이 |
|---|---|---|
| 1 | Ansible이 뭔가요? 5분 만에 이해하기 | 5분 |
| 2 | 설치하기 (Mac / Linux / Windows WSL) | 10분 |
| 3 | 첫 명령: `ansible all -m ping` | 8분 |
| 4 | 인벤토리 완전정복 | 15분 |
| 5 | 첫 플레이북 작성 | 15분 |
| 6 | **멱등성이란? (가장 중요한 개념)** | 12분 |
| 7 | 모듈 Top 10 | 20분 |
| 8 | 변수와 Jinja2 템플릿 | 18분 |
| 9 | 조건문·반복문 (`when`, `loop`) | 15분 |
| 10 | 핸들러와 알림 (`notify`) | 10분 |
| 11 | **`--check`로 안전하게: 실무 필수 습관** | 12분 |
| 12 | 종합 실습: 웹서버 3대 세팅 | 25분 |

**시즌 2: 실무 (15편)**

롤 만들기 / Galaxy 활용 / **Vault** / 에러 처리(`block/rescue`) / 태그 / 동적 인벤토리 / **React 앱 자동 배포** / **Laravel 앱 자동 배포** / Docker 관리 / 무중단 롤링 배포 / 성능 튜닝 / 커스텀 모듈 제작 / 커스텀 필터 제작 / GitHub Actions 연동 / 트러블슈팅 총정리

**시즌 3: AI 융합 (15편) — 차별화 핵심**

AI 인프라 관리 시대 / **Ansible 모듈 = LLM 도구인 이유** / `argument_spec` 완전 분해 / `DOCUMENTATION` → 도구 설명 변환 / **MCP 서버 만들기 1~3** / Claude Code 연결 / AI 플레이북 생성 + 자동 검증 / Human-in-the-loop 설계 / `.claude/skills/` 분석 / **권한 최소화 패턴** / `context/*.md` 작성법 / React 대시보드 통합 / 종합: 완전 자동 인프라 에이전트

### 10.3 제작 실무

**장비**

| 항목 | 최소 | 권장 | 비용 |
|---|---|---|---|
| **마이크** (가장 중요) | 에어팟 | Blue Yeti / Shure MV7 | 10~25만원 |
| 화면 녹화 | OBS Studio | OBS + 듀얼 모니터 | 0원 |
| 편집 | DaVinci Resolve | Premiere / Final Cut | 0~월 3만원 |
| 다이어그램 | Excalidraw | + Figma | 0원 |
| 얼굴 노출 | 불필요 | 썸네일만 | 0원 |

> 화질이 나빠도 보지만, **음질이 나쁘면 바로 나갑니다.** 마이크에 투자하세요.

**영상 구성 템플릿**

```
0:00-0:15   훅 — "서버 100대에 nginx 설치, 몇 시간? ...30초입니다."
0:15-0:45   오늘 배울 것 (3줄 요약)
0:45-1:30   문제 상황 (수동 작업의 지루함 체감시키기)
1:30-10:00  실습 (전체의 70%)
            - 타이핑 그대로 보여주기 (복붙 금지)
            - 일부러 에러 내고 고치기 (반응 가장 좋음)
10:00-11:00 복습 + 멱등성 체감 ("한 번 더 실행해 볼까요?")
11:00-11:30 정리 + 다음 영상 예고 + 코드 링크
```

**제목 공식**

```
나쁨: "Ansible 강의 3강"
좋음: "서버 100대에 nginx 설치, 30초 만에 끝내기 | Ansible 입문 #3"

나쁨: "Ansible Vault 사용법"
좋음: "DB 비밀번호를 깃허브에 올려도 안전한 방법 | Ansible Vault"

나쁨: "MCP 서버 만들기"
좋음: "Claude에게 서버 100대 관리 권한 주기 (안전하게) | Ansible MCP"
```

**썸네일 3요소**: ① 큰 숫자·시간("30초", "100대") ② Before→After 대비 ③ 터미널 녹색 글씨

**업로드 전략**: 시즌1 12편 미리 제작 → 주 2편씩 6주 발행 (초기 구독자 이탈 방지)

### 10.4 수익 모델 단계

| 단계 | 시기 | 수익원 | 예상 |
|---|---|---|---|
| 1 | 0~6개월 | 무료 공개 (구독자 확보) | 0원 |
| 2 | 6~12개월 | YouTube 애드센스 | 월 10~50만원 |
| 3 | 9개월~ | **유료 강의** (인프런 / Udemy / 자체) | 건당 5~15만원 × 수백명 |
| 4 | 12개월~ | 기업 출강·사내 교육 | 회당 100~300만원 |
| 5 | 12개월~ | 멤버십 (Q&A, 코드리뷰) | 월 1~3만원 × 구독자 |
| 6 | 상시 | **외주·컨설팅 유입** (영상이 포트폴리오) | **건당 수백만원** |
| 7 | 18개월~ | 전자책·종이책 | 인당 2~4만원 |
| 8 | 상시 | 스폰서십 | 건당 50~300만원 |

> **가장 큰 수익은 6단계(외주 유입)입니다.** 영상 42편이 있으면 영업 없이 문의가 들어옵니다.

### 10.5 플랫폼 비교

| 플랫폼 | 수수료 | 장점 | 단점 | 추천 |
|---|---|---|---|---|
| **인프런** | 30~40% | 한국 개발자 집중, 마케팅 대행 | 수수료 높음, 심사 | ★★★★★ |
| Udemy | 50~97% | 글로벌 노출 | 할인 폭탄, 수수료 최악 | ★★★ |
| 자체 (Teachable/Gumroad) | 3~10% | **수익 최대**, 고객 데이터 | 마케팅 직접 | ★★★★ |
| 유튜브 멤버십 | 30% | 간편, 구독형 | 단가 낮음 | ★★★ |

> **추천 조합**: 유튜브(무료 유입) → 인프런(첫 유료 강의) → 자체 플랫폼(2번째부터)

### 10.6 리스크와 대응

| 리스크 | 대응 |
|---|---|
| 초기 조회수 저조 | 정상입니다. 시즌1을 미리 만들어 두고 6개월 버티기 |
| 한국 시장이 작다 | 영문 자막 추가 → 글로벌 노출 |
| 버전 변경 | 핵심 문법은 안정적. 보충 영상만 추가 |
| AI가 다 알려줘서 수요 감소 | **그래서 시즌3(AI 융합)이 핵심** |
| 제작 시간 부담 | 1편 ≈ 6시간 (기획 2h + 녹화 1h + 편집 3h). 주 1편이 현실적 |
| 상표권 | 채널명에 "Ansible" 단독 사용 금지. "~로 배우는 Ansible" 식으로 |

---

## 11. 수익화 아이디어 10선

### 전체 비교표

| # | 아이디어 | 난이도 | 초기 투자 | 수익 | 경쟁 | 적합도 | 우선순위 |
|---|---|---|---|---|---|---|---|
| 9 | 블로그·뉴스레터 | ★ | 0원 | ★★ | ★★★ | ★★★★ | **1순위** |
| 2 | 유튜브·강의 | ★★ | 20만원 | ★★★★ | ★★ | ★★★★ | **1순위** |
| 1 | MCP 서버 | ★★★ | 0원 | ★★★★★ | ★ | ★★★★ | **1순위** |
| 7 | 템플릿 패키지 | ★★ | 시간 2개월 | ★★★ | ★★ | ★★★★ | 2순위 |
| 4 | 플레이북 마켓 | ★★★ | 시간 3개월 | ★★★ | ★★★ | ★★★ | 2순위 |
| 3 | 외주·컨설팅 | ★★★ | 0원 | ★★★★★ | ★★★ | ★★★ | 2순위 |
| 6 | React 대시보드 | ★★★★ | 시간 6개월 | ★★★★ | ★★ | ★★★★★ | 3순위 |
| 10 | 자격증 교육 | ★★★ | 시간 3개월 | ★★★ | ★ | ★★★ | 3순위 |
| 5 | AI 플레이북 SaaS | ★★★★★ | 월 50~200만 | ★★★★★ | ★★ | ★★ | 4순위 |
| 8 | 컴플라이언스 감사 | ★★★★★ | 자격증 필요 | ★★★★★ | ★★★★ | ★★ | 4순위 |

---

### 아이디어 1: Ansible MCP 서버 (AI × 인프라)

> **"Claude/Cursor 같은 AI가 서버 100대를 안전하게 관리하게 해주는 다리"**

**왜 지금인가**: MCP 생태계가 폭발 성장 중인데 인프라 관리용 MCP 서버는 거의 없습니다. Ansible은 15년 검증된 멱등성 + `--check` 안전장치를 가지고 있습니다.

**도구 3계층 설계 (핵심 차별점)**

| 계층 | 도구 | 위험 | 승인 |
|---|---|---|---|
| 🟢 조회 | `list_hosts` `gather_facts` `list_playbooks` `doc_lookup` | 없음 | 자유 |
| 🟡 모의실행 | `dry_run_playbook` `syntax_check` `diff_preview` | 없음 | 자유 |
| 🔴 변경 | `apply_playbook` `restart_service` `deploy` | 높음 | **사람 승인 필수** |

**핵심 구현 — 승인 토큰 패턴**

```python
@mcp.tool()
def dry_run_playbook(playbook: str, inventory: str = "hosts.ini", limit: str = "") -> str:
    """플레이북을 모의 실행(--check --diff)하여 '무엇이 바뀔지'만 보고합니다.
    실제 서버는 전혀 변경하지 않으므로 안전합니다."""
    result = _run(["ansible-playbook", "-i", inv, pb, "--check", "--diff"])

    # 모의실행을 통과한 조합만 실제 실행 가능 (15분 유효)
    token = hashlib.sha256(f"{playbook}{inventory}{limit}{time.time()}".encode()).hexdigest()[:16]
    PENDING[token] = {"playbook": playbook, "inventory": inventory,
                      "limit": limit, "expires": time.time() + 900}
    result["approval_token"] = token
    return json.dumps(result, ensure_ascii=False)

@mcp.tool()
def apply_playbook(approval_token: str, user_confirmed: bool = False) -> str:
    """플레이북을 실제로 적용합니다. 서버를 변경합니다."""
    if not user_confirmed:
        return "거부됨: 사용자 승인이 필요합니다."
    job = PENDING.pop(approval_token, None)
    if not job:
        return "거부됨: 유효하지 않은 토큰. dry_run_playbook을 먼저 실행하세요."
    if time.time() > job["expires"]:
        return "거부됨: 토큰 만료(15분). 다시 모의 실행하세요."
    return json.dumps(_run([...], timeout=3600), ensure_ascii=False)
```

**사용 시나리오**

```
사용자: "웹서버들 디스크 상태 확인해줘"
Claude: [gather_facts 호출]
        "web1: 78% / web2: 91% ⚠️ / web3: 45%"

사용자: "web2 로그 정리해줘"
Claude: [dry_run_playbook("cleanup-logs.yml", limit="web2") 호출]
        "모의 실행 결과:
         • /var/log/nginx/*.log.* 삭제 (2.1GB)
         • logrotate: rotate 52 → 14
         • 예상 확보: 3.5GB
         실행하시겠습니까?"

사용자: "응"
Claude: [apply_playbook(token, user_confirmed=True) 호출]
        "완료. 91% → 77%로 개선되었습니다."
```

**수익 모델**

| 플랜 | 가격 | 포함 |
|---|---|---|
| Community | 무료 (오픈소스) | 조회 + 모의실행, 단일 사용자 |
| **Pro** | **$29/월** | 변경 도구 + 감사 로그 + 다중 인벤토리 |
| **Team** | **$99/월** (5명) | RBAC + Slack 승인 플로우 + 팀 리포트 |
| **Enterprise** | **$499/월+** | SSO/SAML, 온프레미스, SLA, 커스텀 도구 |
| Setup 서비스 | 건당 200~800만원 | 환경 구축 + 플레이북 작성 + 교육 |

**고객이 돈을 내는 이유**

```
고통: "AI한테 서버 맡기고 싶은데 rm -rf 치면 어떡하나"
      "누가 뭘 바꿨는지 기록이 없다"
      "주니어가 운영 서버 만지는 게 무섭다"

해결: --check 기본 → 실행 전 항상 미리보기
      승인 토큰 → 사람 승인 없이는 변경 불가
      감사 로그 → 전부 기록
      3계층 권한 → 주니어는 조회만
```

**로드맵**

| 기간 | 할 일 |
|---|---|
| 1~2주 | 조회 도구 4개 + GitHub 공개 (MVP) |
| 3~4주 | 모의실행 + 승인 토큰 (차별화 완성) |
| 5~6주 | README·데모 영상·블로그 |
| 7~8주 | Reddit(r/ansible, r/devops), HN, X 공유 |
| 3개월 | Pro 플랜 출시 (Stripe) |
| 6개월 | Team 플랜 + 웹 대시보드, MRR $1,000 목표 |

---

### 아이디어 2: 한국어 Ansible 교육

> 상세는 [10장](#10-유튜브-강의-제작-가능성) 참고.

**3단 로켓**

```
1단: 유튜브 무료 (시즌1 12편) → 구독자 1,000명 + 신뢰 / 3~6개월
2단: 유료 강의 (시즌2+3, 30편) → 8~15만원 × 300~1,000명
3단: 기업 교육 + 외주 유입 → 출강 100~300만원/회, 외주 500~3,000만원/건
```

**가격 책정**

| 등급 | 가격 | 내용 |
|---|---|---|
| Basic | 8만원 | 시즌1 12편 + 예제 코드 |
| **Standard** (주력) | **15만원** | 시즌1+2 (27편) + 실습 환경 + Q&A |
| Pro | 29만원 | 전체 42편 + 1:1 코드리뷰 3회 + 수료증 + 평생 업데이트 |
| 기업용 | 인당 20만원 (5인+) | 전체 + 사내 Q&A 세션 + 맞춤 사례 1건 |

> Basic을 일부러 아쉽게 만들어 Standard로 유도 (decoy pricing).

**예상 수익 (연간)**

| 항목 | 보수적 | 낙관적 |
|---|---|---|
| 애드센스 | 120만원 | 600만원 |
| 강의 매출 (인프런 35% 차감 후) | 1,560만원 | 9,360만원 |
| 기업 출강 | 300만원 | 2,500만원 |
| 외주 유입 | 500만원 | 6,000만원 |
| **합계** | **약 2,480만원** | **약 1억 8,460만원** |

**초기 투자**: 약 20만원 (마이크 15만 + 실습 서버 월 3만)

---

### 아이디어 3: 서버 자동화 구축 외주·컨설팅

> **"수동으로 서버 관리하는 중소기업에 자동화를 구축해 주고 받는 돈"**

**타깃 고객 (돈이 되는 순서)**

| 타깃 | 고통 | 예산 | 접근 경로 |
|---|---|---|---|
| IT 담당자 1~2명 중소기업 | 서버 20~50대 수동 관리. 퇴사 시 지식 소실 | 500~2,000만원 | 중소기업진흥공단, 상공회의소 |
| 스타트업 (시리즈 A~B) | 개발자가 인프라 겸업. 배포 수동 | 300~1,500만원 | 로켓펀치, 데모데이 |
| SI·솔루션 업체 | 고객사마다 반복 세팅 | 1,000~5,000만원 | 영업 네트워크 |
| 쇼핑몰·호스팅 | 서버 증설 시 매번 수동 | 500~3,000만원 | 업계 커뮤니티 |
| 공공기관·학교 | 보안 점검·패치 수동 | 2,000만~1억 | 나라장터 입찰 |

**서비스 패키지**

| 패키지 | 가격 / 기간 | 내용 |
|---|---|---|
| **Starter** | 300만원 / 2주 | 현황 진단 + 로드맵 문서, 인벤토리 구축(~20대), 기본 플레이북 3종, 2시간 교육 |
| **Standard** (주력) | 1,000만원 / 6주 | Starter 전체 + 롤 구조 설계 + Vault 체계 + CI/CD 연동 + 무중단 배포 + ~50대 + 8시간 교육 + 3개월 지원 |
| **Enterprise** | 3,000만원+ / 12주+ | Standard + 멀티환경(dev/stg/prod) + 커스텀 모듈 + 보안 컴플라이언스 + 웹 대시보드 + 100대+ + 6개월 지원 |
| **운영 대행 (리테이너)** | **월 150~500만원** | 월간 패치 자동화 + 리포트, 플레이북 유지보수, 긴급 대응(4시간 내), 분기 개선 제안 |

> **핵심**: 일회성 구축보다 **리테이너**가 훨씬 가치 있습니다.
> 월 300만원 × 5개 고객 = **연 1.8억 안정 수익**.

**ROI 설득 자료 (제안서 필수)**

```
현재 (수동):
  서버 50대 × 월 1회 패치 × 30분 = 25시간/월
  인건비 시간당 5만원 → 월 125만원
  + 장애 복구: 분기 1회 × 8시간 = 월 13만원
  → 연간 약 1,656만원

도입 후:
  검증 20분/월 = 1.7만원/월
  장애: 멱등성으로 거의 제거
  → 연간 약 20만원

연간 절감 약 1,636만원 → 구축비 1,000만원을 7.3개월에 회수
```

**첫 고객 확보 전략**

```
1. 포트폴리오 먼저 (2주)
   가상 시나리오로 완전한 자동화 프로젝트 1개 → GitHub + 블로그 + 유튜브 데모
2. 무료 진단으로 미끼
   "서버 자동화 가능성 무료 진단(2시간)" → 리포트에 절감 시간 수치 제시
3. 콘텐츠 마케팅 (아이디어 2와 결합)
   유튜브·블로그가 24시간 영업사원
4. 커뮤니티 활동
   OKKY, 페이스북 개발자 그룹 → 질문 답변 → 전문가 인식 → DM
5. 기존 인맥
   첫 고객은 보통 인맥에서. 할인해 주고 사례로 활용
```

---

### 아이디어 4: 프리미엄 플레이북·롤 마켓

> **"실무에서 바로 쓸 수 있는 검증된 자동화 레시피 판매"**

```
직접 작성: 플레이북 1세트 + 테스트 = 40시간 (200만원 상당)
구매:      10만원 + 커스터마이징 4시간
→ 명확한 가치 제안. GPL 영향 없음 (내 YAML은 내 저작물)
```

**상품 라인업**

| 패키지 | 내용 | 가격 |
|---|---|---|
| **한국형 보안 기준선** | ISMS-P / 주요정보통신기반시설 점검 자동화 + 조치 + 리포트 | **29만원** |
| LAMP/LEMP 완전 세팅 | nginx/apache + PHP 8.3 + MySQL/MariaDB + Redis + Certbot | 9만원 |
| React/Next.js 배포 세트 | Node + nginx + PM2 + SSL + 무중단 + 롤백 | 9만원 |
| Laravel 운영 세트 | PHP-FPM 튜닝 + Supervisor + Horizon + Scheduler | 12만원 |
| Docker/K8s 호스트 준비 | Docker + Compose + 레지스트리 + 로그 수집 | 12만원 |
| 모니터링 스택 | Prometheus + Grafana + Exporter + 한국어 대시보드 | 19만원 |
| 백업·재해복구 | DB/파일 백업 + S3·NAS 전송 + 복구 테스트 | 19만원 |
| **전체 번들** | 위 전부 + 1년 업데이트 + 이메일 지원 | **79만원** |

**품질 차별화 (무료 Galaxy 롤과의 차이)**

```
✓ 한국어 문서 + 주석 (가장 큰 차별점)
✓ molecule 테스트 포함 (실제 작동 증명)
✓ 멀티 OS (Ubuntu 22/24, Rocky 8/9, Amazon Linux 2023)
✓ --check 모드 완전 지원
✓ 변수 전부 defaults/main.yml 분리 (커스터마이징 쉬움)
✓ 한국 환경 고려 (KST, 한국 미러, 한글 로케일)
✓ 실제 운영 사례 문서
✓ 영상 설치 가이드
```

**판매 채널**

| 채널 | 수수료 | 비고 |
|---|---|---|
| 자체 (Gumroad / Lemon Squeezy) | 5~10% | 추천 |
| GitHub Sponsors + 비공개 저장소 | 10% | 구독형 |
| 인프런 (자료 판매) | 30% | 한국 접근성 |
| Ansible Galaxy (무료 티저) | — | 간단 버전 무료 → 유료 유도 |

**수익 구조**: 제작 320시간(2~3개월) → 월 20건 × 12만원 = **연 2,880만원** (보수적) / 월 60건 × 15만원 = **연 1억 800만원** (낙관적)

---

### 아이디어 5: AI 플레이북 생성 SaaS

> **"자연어로 말하면 검증된 Ansible 플레이북을 만들어 주는 웹 서비스"**

**제품 흐름**

```
"우분투 24.04에 nginx + PHP 8.3 + MySQL 8 설치, Let's Encrypt SSL"
         ↓
1. LLM이 초안 생성 (Ansible 모듈 문서를 RAG로 제공)
         ↓
2. 자동 검증 파이프라인 (핵심 차별점)
   ✓ ansible-playbook --syntax-check
   ✓ ansible-lint (베스트 프랙티스)
   ✓ Docker 컨테이너에서 실제 실행
   ✓ 멱등성 검증 — 2회 실행해 changed=0 확인
   ✗ 실패 → 에러를 LLM에 재투입 → 자동 수정 (최대 3회)
         ↓
3. 검증 완료 플레이북 + 실행 로그 제공
```

**ChatGPT보다 나은 점 (돈 받는 이유)**

| | ChatGPT 직접 | 이 서비스 |
|---|---|---|
| 문법 오류 | 자주, 사용자가 직접 발견 | **자동 검사 + 수정** |
| 구버전 문법 | `ansible.builtin.` 누락 흔함 | **최신 문서 RAG로 보정** |
| 실제 작동 보장 | 없음 | **Docker에서 실제 실행 검증** |
| 베스트 프랙티스 | 모름 | **ansible-lint 통과** |
| 멀티 OS | 한 가지 | **Ubuntu/Rocky/Amazon Linux 동시** |
| **멱등성** | 보장 안 됨 | **2회 실행 changed=0 확인** |

> **"멱등성 자동 검증"이 킬러 기능입니다.** ChatGPT는 절대 못 하는 일입니다.

**핵심 검증 로직**

```python
async def generate_and_verify(prompt: str, target_os: str, max_retries: int = 3) -> dict:
    context = await rag_search(prompt)

    for attempt in range(max_retries):
        playbook = await llm_generate(prompt, context, target_os)

        if err := await syntax_check(playbook):
            context += f"\n\n이전 문법 오류: {err}";  continue
        if lint_err := await ansible_lint(playbook):
            context += f"\n\n린트 경고: {lint_err}";   continue

        run1 = await docker_run(playbook, target_os)
        if not run1["success"]:
            context += f"\n\n실행 실패: {run1['error']}";  continue

        # 멱등성 검증 — 2회차에 changed=0 이어야 함
        run2 = await docker_run(playbook, target_os, reuse_container=True)
        if run2["changed_count"] > 0:
            context += (f"\n\n멱등성 위반: 2회차에 {run2['changed_count']}개 태스크가 "
                        f"changed. 수정 대상: {run2['changed_tasks']}")
            continue

        return {"playbook": playbook, "verified": True, "attempts": attempt + 1,
                "logs": {"first_run": run1, "idempotency_check": run2}}

    return {"playbook": playbook, "verified": False, "note": "수동 확인 필요"}
```

**기술 스택**

```
프론트: React + TypeScript + Tailwind + Monaco Editor
백엔드: FastAPI (Ansible 연동 때문에 Python)
LLM:    Claude API (claude-opus-5 / claude-sonnet-5-5)
RAG:    ansible-doc --json 덤프 → 벡터 DB (Chroma / Qdrant)
샌드박스: Docker-in-Docker 또는 Firecracker (격리 필수)
큐:     Celery + Redis
DB:     PostgreSQL / 결제: Stripe·토스페이먼츠
배포:   Fly.io / Railway / AWS ECS
```

**수익 모델**

| 플랜 | 가격 | 포함 |
|---|---|---|
| Free | 0원 | 월 3회, 문법 검사만, 워터마크 |
| Starter | $19/월 | 월 50회, 전체 검증, Ubuntu/Debian |
| Pro | $49/월 | 무제한, 멀티 OS, 멱등성 검증, GitHub 연동 |
| Team | $149/월 | 5인, 팀 라이브러리 공유 |
| Enterprise | $999/월+ | 온프레미스, 사내 플레이북 파인튜닝, SLA |

**리스크**

| 리스크 | 심각도 | 대응 |
|---|---|---|
| LLM API 비용 | 높음 | 프롬프트 캐싱, 재시도 제한, 결과 캐싱, Free 엄격 제한 |
| Docker 샌드박스 비용 | 중 | 컨테이너 재사용, 배치 처리, 타임아웃 |
| 보안 (임의 코드 실행) | 높음 | 완전 격리(Firecracker/gVisor), 네트워크 차단 |
| ChatGPT가 비슷해지면 | 중 | **"실제 검증"이 방어선.** OpenAI는 Docker를 돌려주지 않습니다 |
| 개발 난이도 | 높음 | MVP는 문법 검사만 → 점진 확장 |

---

### 아이디어 6: React 기반 Ansible 웹 대시보드

> **"AWX/Ansible Tower는 너무 무겁다. 가볍고 예쁜 대안을 판다"**

**시장 기회**

```
Red Hat AAP   → 엔터프라이즈 전용, 연 수천만원, 과도하게 복잡
AWX (오픈소스) → 무료지만 K8s 필요, 설치 복잡, UI 낡음, 리소스 과다
그 외          → 거의 없음  ← 기회

포지션: "docker compose up 한 줄로 설치되는, 가볍고 현대적인 Ansible UI"
```

**기능**

```
인벤토리 관리   서버 CRUD, 그룹·태그, 동적 인벤토리 연동
플레이북 관리   Monaco 에디터, Git 연동, 버전 관리
실행           원클릭, --check 우선, 실시간 로그(SSE)
이력·통계      성공률, 소요시간 추이, 서버별 상태
RBAC          역할별 권한 (조회 / 모의실행 / 실행 / 관리)
Vault 통합     비밀 관리 UI (값은 안 보여주고 참조만)
스케줄         cron 기반 정기 실행
알림           Slack / Discord / 이메일 / Webhook
감사 로그      누가/언제/무엇을/결과 전부
다크 모드      (개발자 필수)
```

**AWX 대비 차별화**

| | AWX | 이 제품 |
|---|---|---|
| 설치 | K8s/Operator, 수십 분 | **`docker compose up` 1분** |
| 리소스 | 4GB RAM+ | **512MB** |
| UI | 낡은 PatternFly | **현대적 React + 다크모드** |
| `--check` 우선 | 옵션 | **기본 워크플로** |
| AI 연동 | 없음 | **MCP 서버와 통합** |
| 한국어 | 없음 | 지원 |

**수익 모델 (Open Core)**

| 플랜 | 가격 | 포함 |
|---|---|---|
| Community (오픈소스) | 무료 | 인벤토리, 실행, 기본 로그 / 1명, 10대 |
| Pro | $49/월 | 무제한 사용자·서버, RBAC, 스케줄, 알림, 감사 로그 |
| Business | $199/월 | + SSO/LDAP, Vault 통합, Git 동기화, API, 다중 환경 |
| Enterprise | $999/월+ | + AI 어시스턴트(MCP), SLA, 전담 지원 |
| 설치·구축 서비스 | 건당 300~1,000만원 | 고객사 직접 구축 + 플레이북 + 교육 |

> Open Core의 힘: GitHub 별 1,000개 = 영업사원 10명 역할.

**보안 설계 (타협 불가)**

```python
# 절대 금지
os.system(f"ansible-playbook {request.playbook}")   # 명령 인젝션

# 반드시 이렇게
ALLOWED = {p.id: p.path for p in db.query(Playbook).all()}
if playbook_id not in ALLOWED:
    raise HTTPException(403, "허용되지 않은 플레이북")
subprocess.run(
    ["ansible-playbook", "-i", str(inv_path), str(ALLOWED[playbook_id])],
    cwd=PROJECT_DIR, env=SAFE_ENV, timeout=3600, user="ansible-runner",
)
```

> 체크리스트는 [9.4절](#94-보안-수칙-10개-절대-타협-불가) 참고.

**적합도**: 당신의 React/PHP 경험을 직접 활용 → ★★★★★

---

### 아이디어 7: 중소기업 전용 자동화 템플릿 패키지

> **"IT 담당자가 1~2명인 회사가 바로 쓸 수 있는 자동화 키트"**

```
대기업:   전담 인프라팀 → 직접 구축
중소기업: IT 담당자 1~2명 → 하고 싶지만 시간·지식 없음  ← 여기
개인:     무료 자료로 충분
```

**제품 구성 — "중소기업 서버 자동화 키트" 149만원**

```
1. 바로 쓰는 프로젝트 구조
   inventories/{dev,prod}/ + roles/ + playbooks/ + 설정 완료

2. 플레이북 12종 (한국어 주석)
   ✓ 신규 서버 초기 세팅 (보안 기본값 + 모니터링 에이전트)
   ✓ 월간 보안 패치 (Ubuntu/Rocky 동시 지원)
   ✓ 전체 서버 백업 (DB + 파일 → NAS/S3)
   ✓ 백업 복구 테스트 (백업만 하고 복구 안 되는 사고 방지)
   ✓ SSL 인증서 자동 갱신 + 만료 알림
   ✓ 로그 정리 + logrotate 설정
   ✓ 사용자 계정 일괄 관리 (입사·퇴사 처리)
   ✓ 디스크·메모리 임계치 점검 + Slack 알림
   ✓ 방화벽 규칙 일괄 적용
   ✓ 서버 자산 현황 리포트 (엑셀 출력)
   ✓ 긴급 전체 재부팅 (롤링, 무중단)
   ✓ 신규 직원 PC 세팅 (Windows WinRM)

3. 한국어 운영 매뉴얼 (PDF 60p)
   설치 → 첫 실행 → 플레이북별 사용법 → 트러블슈팅 20선

4. 영상 가이드 10편 (총 90분)

5. 이메일 지원 3개월 + 1년 업데이트

6. 보너스: 경영진 보고용 ROI 자료 템플릿 (PPT)
```

**가격 3단**

| 등급 | 가격 | 차이 |
|---|---|---|
| Lite | 49만원 | 플레이북 5종 + 매뉴얼 |
| **Standard** | **149만원** | 전체 12종 + 영상 + 3개월 지원 |
| Premium | 349만원 | + 1:1 온보딩 2시간 × 2회 + 맞춤 수정 1건 |

> **왜 팔리나**: "외주 구축 1,000만원, 직접 하면 2개월. 149만원이면 이번 주에 끝납니다."
> 500만원 미만은 보통 팀장 전결 → 결재가 쉬운 금액대.

**결합 효과**: 아이디어 3(외주)의 **입구 상품**. 산 고객이 나중에 외주로 전환.

---

### 아이디어 8: 보안 컴플라이언스 감사 자동화

> **"Ansible로 전 서버 보안 설정을 점검하고, 인증 심사용 리포트를 자동 생성"**

```
기업이 돈을 쓰는 이유: ① 돈을 벌어서 ② 벌금·인증 때문에 어쩔 수 없이
컴플라이언스는 ② → 가격 저항이 가장 낮습니다
ISMS-P 인증 못 받으면 사업 자체가 막히는 기업이 있습니다
```

**타깃 규제**

| 기준 | 대상 | 수요 |
|---|---|---|
| **ISMS-P** | 매출·이용자 기준 충족 기업 (의무) | 매우 높음 |
| **주요정보통신기반시설 취약점 분석·평가** | 공공·금융·통신 | 매우 높음 |
| 전자금융감독규정 | 금융권 | 높음 |
| CIS Benchmark | 글로벌 표준 | 중 |
| PCI-DSS | 카드 결제 취급 | 중 |

**제품 구조**

```
1. 점검 플레이북 — 서버 100대 SSH 접속 → 설정 수집 → 기준 비교
   (읽기 전용 --check 기본 → 심사 중에도 안전)
2. 판정 엔진 — 항목별 양호/취약/해당없음 + 근거 증적 수집
3. 리포트 생성 (React + PDF)
   • 심사 제출용 양식 (ISMS-P 점검표 포맷)
   • 경영진 요약 (대시보드, 등급, 추이)
   • 조치 가이드 (취약 항목별 해결 방법 + 플레이북 링크)
4. 자동 조치 플레이북 (고가 옵션)
   --check로 미리보기 → 승인 → 일괄 조치
```

**점검 항목 예시**

```yaml
# roles/security_audit/tasks/ssh.yml
- name: "U-01 | SSH root 직접 로그인 제한 점검"
  ansible.builtin.command: sshd -T
  register: sshd_config
  changed_when: false
  check_mode: false

- name: "U-01 | 판정"
  ansible.builtin.set_fact:
    audit_results: "{{ audit_results + [{
      'id': 'U-01',
      'title': 'SSH root 직접 로그인 제한',
      'category': '계정관리',
      'severity': '상',
      'result': ('양호' if 'permitrootlogin no' in sshd_config.stdout|lower else '취약'),
      'evidence': (sshd_config.stdout_lines | select('search','permitrootlogin') | list),
      'guide': '/etc/ssh/sshd_config 에서 PermitRootLogin no 설정 후 sshd 재시작',
      'checked_at': ansible_date_time.iso8601
    }] }}"
```

**수익 모델**

| 상품 | 가격 |
|---|---|
| 1회 감사 리포트 | **500~1,500만원** (서버 50~200대) |
| **연간 감사 계약** | **월 200~600만원** (분기별 자동 점검 + 리포트) |
| 점검 도구 라이선스 | 연 1,000~3,000만원 (고객 직접 운영) |
| 자동 조치 옵션 | +300~800만원 |
| 심사 대응 컨설팅 | 시간당 20~40만원 |

**진입 전략 (기술만으론 부족)**

```
이 분야는 "기술"보다 "신뢰"로 팝니다.

1. ISMS-P 또는 정보보안 자격증 (CISSP, CISA, 정보보안기사)
2. 또는 보안 컨설팅 업체와 파트너십 (현실적)
   → 기술(자동화) 제공, 파트너는 영업·심사 대응. 수익 배분 4:6 또는 5:5
3. 첫 사례 1건 — 지인 회사에 저가로 해 주고 reference 확보
```

**차별화**: 자동화로 인건비 1/10 → 가격 경쟁력 또는 마진 극대화

---

### 아이디어 9: 기술 블로그·뉴스레터

> **"수익 자체는 작지만, 다른 모든 아이디어의 유입 엔진"**

```
직접 수익 (연 100~600만원)
├── 애드센스 / 제휴
├── 스폰서십 (클라우드·DevOps 툴 업체) 건당 50~300만원
├── 유료 뉴스레터 (월 5,000~15,000원 × 구독자)
└── 기술 기고 건당 30~100만원

간접 가치 (이게 본질)
├── 아이디어 2(강의) 수강생 유입
├── 아이디어 3(외주) 문의 유입  ← 가장 큼
├── 아이디어 4(플레이북) 판매 유입
└── "전문가" 포지셔닝 → 모든 단가 상승
```

**콘텐츠 전략**

```
잘 되는 글 (문제 해결형 — 검색 유입)
  "Ansible 'UNREACHABLE' 에러 해결 방법 7가지"
  "Ansible이 느릴 때 10배 빠르게 하는 설정 5개"
  "ansible-vault 암호를 CI/CD에서 안전하게 쓰는 방법"
  "Ansible로 Laravel 무중단 배포하기 (실전 코드)"

안 되는 글
  "Ansible 소개" (경쟁 과열)
  "오늘 공부한 것" (검색 유입 0)
```

---

### 아이디어 10: Ansible 자격증 대비 교육

| 자격증 | 발급 | 응시료 | 한국어 교재 |
|---|---|---|---|
| **RHCE (EX294)** — Ansible | Red Hat | 약 60만원 | **거의 없음** |
| RHCSA | Red Hat | 약 60만원 | 일부 있음 |

**제품 — "EX294 합격 패스" 39만원**

```
• 시험 범위 100% 커버 강의 30편
• 모의고사 5회 (실기 환경 그대로)
• 브라우저 실습 환경 (Docker + xterm.js + Monaco)
  → React로 만든 가상 터미널에서 실제 채점
• 합격 수기 + 시험 팁
• 질문 게시판 + 월 1회 라이브 Q&A
• 불합격 시 1회 재수강 무료
```

> **React 활용 포인트**: 브라우저 실습 환경이 핵심 차별점. 아이디어 6의 기술 재활용 가능.
> **리스크**: 응시자 수가 제한적 (연 수백~수천 명 규모).

---

## 12. 라이선스 주의사항

### GPL-3.0이 의미하는 것

| 구분 | 내용 |
|---|---|
| 🟢 **안전** | Ansible을 "도구로 사용"해 서비스 제공 (컨설팅·구축·운영 대행·교육) |
| 🟢 **안전** | 내가 작성한 플레이북·롤·컬렉션 판매 (내 저작물) |
| 🟢 **안전** | Ansible을 **서브프로세스로 호출**하는 제품 (별도 프로그램) |
| 🟢 **안전** | SaaS 호스팅 (GPL-3.0은 AGPL과 달리 네트워크 조항 없음) |
| 🟢 **안전** | 교육 콘텐츠 (강의·책·유튜브·블로그) |
| 🔴 **위험** | Ansible 코드를 **수정해서 비공개 제품으로 배포** → 소스 공개 의무 |
| 🔴 **위험** | Ansible을 **라이브러리로 import**해서 비공개 앱 배포 → GPL 전염 |
| 🔴 **위험** | "Ansible" 상표를 제품명·회사명에 사용 (Red Hat 상표) |
| 🔴 **위험** | Red Hat 공식 제품인 척 하기 |

### 황금 규칙 3개

```
규칙 1: 코드를 "섞지(link)" 말고 "호출(call)"하라
        import ansible.xxx                     ❌
        subprocess.run(["ansible-playbook"])   ✅

규칙 2: 코드를 팔지 말고 "코드가 아닌 것"을 팔라
        지식(교육) · 시간(서비스) · 자산(플레이북) · 편의(UI)

규칙 3: 상표는 서술적으로만 사용하라
        "AnsibleMaster"                         ❌
        "DevFlow — Ansible 기반 자동화 플랫폼"   ✅
```

> 상업적으로 본격 진행 전 **IT 전문 변호사 1회 검토**를 권합니다.

### 기타 주의사항

| # | 주의점 | 대처 |
|---|---|---|
| 1 | 이건 `devel` 브랜치 (`2.23.0.dev0`, 불안정) | 실무용은 `pip install ansible` (안정판) |
| 2 | Python 3.13+ 필수 | 제어 머신만 해당. 관리 대상은 구버전도 OK |
| 3 | 제어 머신은 Linux/macOS만 | Windows는 WSL2. 관리 대상은 Windows OK (`winrm`) |
| 4 | 이건 fork(복사본) | 이력서엔 "컨트리뷰터"로만. "제작"이라 쓰면 거짓 |
| 5 | `ansible-core` ≠ `ansible` | AWS/Azure/Docker/K8s 모듈은 `pip install ansible` |
| 6 | `shell`/`command` 모듈은 멱등하지 않음 | 전용 모듈 우선. 불가피하면 `creates:`/`removes:` 조건 |
| 7 | 상태 저장 없음 (Terraform과 다름) | state 관리가 필요하면 Terraform 병용 |

---

## 13. 12개월 실행 로드맵

```
┌─ 0~3개월: 기반 구축 (수익 거의 0, 자산 축적) ──────────────┐
│  Week 1-2   블로그 개설 + 글 4편 (아이디어 9)               │
│  Week 3-6   Ansible MCP 서버 MVP (아이디어 1)               │
│             → GitHub 공개 → Reddit/HN 공유                  │
│  Week 7-12  유튜브 시즌1 12편 제작 (아이디어 2)              │
│             → 주 2편 발행 시작                               │
│  수익: 0~50만원 | 얻는 것: 신뢰, 포트폴리오, 유입 경로       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─ 3~6개월: 첫 수익화 ───────────────────────────────────────┐
│  • MCP Pro 플랜 출시 ($29/월) → 첫 유료 고객                │
│  • 유튜브 구독자 500~1,500명 → 애드센스 시작                 │
│  • 템플릿 패키지 제작 (아이디어 7) → 149만원 판매 시작       │
│  • 블로그 유입으로 첫 외주 문의 발생                          │
│  수익: 월 50~300만원                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─ 6~9개월: 본격 수익 ───────────────────────────────────────┐
│  • 유료 강의 출시 (인프런, 15만원)                           │
│  • 첫 외주 계약 (아이디어 3) → 500~1,000만원                 │
│  • MCP Team 플랜 추가                                        │
│  • 플레이북 마켓 오픈 (아이디어 4)                            │
│  수익: 월 300~1,000만원                                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─ 9~12개월: 확장 & 선택 ────────────────────────────────────┐
│  • React 대시보드 개발 (아이디어 6) ← 본인 강점 활용         │
│    → MCP 서버와 통합 → Enterprise 플랜                       │
│  • 외주 → 운영 대행(리테이너) 전환 (반복 수익화)             │
│  • 기업 출강 시작                                            │
│  수익: 월 500~2,000만원                                      │
│  선택: 아이디어 5(AI SaaS) 또는 8(컴플라이언스)로 확장       │
└─────────────────────────────────────────────────────────────┘
```

### 가장 중요한 조언 3가지

**1. 아이디어 1(MCP) + 2(유튜브)를 동시에**

```
MCP 서버 = 기술 증명 + 선점
유튜브   = 유입 + 신뢰

"Ansible MCP 서버 만들기" 영상이
  → MCP 사용자를 늘리고
  → MCP가 유튜브 구독자를 늘리고
  → 둘 다 외주 문의를 만듭니다
```

**2. React/PHP 경험이 최대 무기**

```
Ansible 전문가 대부분 = 인프라 엔지니어, 프론트엔드 못 함
당신 = Ansible + React/PHP

→ "예쁘고 쓰기 쉬운 Ansible UI"는 아무도 못 만들고 있습니다
→ 아이디어 6(대시보드) 적합도 ★★★★★
→ "웹 개발자를 위한 Ansible" 포지셔닝도 경쟁자 없음
```

**3. 반복 수익(recurring)을 목표로**

```
일회성: 외주 1,000만원 → 끝 → 또 영업
반복:   MCP Pro $29/월 × 100명 = 월 $2,900 (자동)
       운영 대행 월 300만원 × 5개사 = 월 1,500만원 (자동)

→ 일회성으로 현금 확보 → 반복 수익 구조로 전환
```

### 피해야 할 함정

| 함정 | 왜 위험 | 대신 |
|---|---|---|
| 처음부터 아이디어 5·8 도전 | 난이도·투자 과대. 자금 소진 후 포기 | 1·2·9로 시작 |
| 완벽한 제품 만들고 공개 | 6개월 만들었는데 아무도 안 쓰는 경우 다수 | MVP 2주 → 공개 → 피드백 |
| Ansible 코드 수정해서 판매 | GPL 위반, 법적 리스크 | subprocess 호출 방식 고수 |
| "Ansible" 상표 사용 | Red Hat 상표권 | 서술적 사용만 |
| 한국 시장만 보기 | 시장 규모 한계 | 영문 README·자막으로 글로벌 |
| 무료로만 하다 지침 | 번아웃 후 중단 | 3개월 내 작게라도 과금 시작 |

---

## 14. 참고 자료

### 공식 링크

| 자료 | URL |
|---|---|
| **내 저장소** | **https://github.com/bmshin94/ansible** |
| 원본 저장소 | https://github.com/ansible/ansible |
| 공식 문서 | https://docs.ansible.com/ansible-core/devel/ |
| 모듈 색인 | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ |
| Galaxy 허브 | https://galaxy.ansible.com |
| 커뮤니티 포럼 | https://forum.ansible.com |
| 개발자 가이드 | https://docs.ansible.com/ansible-core/devel/dev_guide/ |
| 기여 가이드 | https://github.com/ansible/ansible/blob/devel/.github/CONTRIBUTING.md |
| CI (Azure Pipelines) | https://dev.azure.com/ansible/ansible/ |
| PyPI | https://pypi.org/project/ansible-core |

### 저장소 내 필독 파일

| 파일 | 왜 읽어야 하나 |
|---|---|
| `lib/ansible/modules/ping.py` | 가장 단순한 모듈. **LLM 도구 구조의 원형** |
| `lib/ansible/module_utils/basic.py` | `AnsibleModule` 클래스. **도구 검증 레이어 설계 교과서** |
| `lib/ansible/executor/module_common.py` | 모듈 코드를 원격 전송하는 핵심 로직 |
| `lib/ansible/plugins/loader.py` | 동적 플러그인 등록·발견 |
| `lib/ansible/plugins/strategy/debug.py` | 실패 시 대화형 개입 (HITL 패턴) |
| `.claude/skills/*/SKILL.md` | **권한 최소화(`allowed-tools`) 실제 사례 4건** |
| `context/*.md` (14개) | AI가 읽을 것을 전제로 쓴 프로젝트 규칙서 |
| `AGENTS.md` | AI 에이전트 작업 지침 + PR 리뷰 7단계 |

### 다음 단계 추천

```
1. pip install ansible        → 10분 실습 (5.3절) 따라하기
2. ansible-doc copy           → 모듈 문서 구조 체감
3. lib/ansible/modules/ping.py 읽기 → LLM 도구 구조 이해
4. .claude/skills/review/SKILL.md 읽기 → 권한 최소화 패턴 학습
5. MCP 서버 MVP 2주 제작      → GitHub 공개
```

---

*이 문서는 Claude Code를 통해 `bmshin94/ansible` 저장소를 전수조사하여 작성되었습니다.*
*분석 시점: 2026-10-07 / 대상 커밋: `21a07ef` / 버전: `ansible-core 2.23.0.dev0`*
