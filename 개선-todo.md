# 저장소 소프트웨어 공학 분석 — Qurious

| 항목 | 내용 |
|---|---|
| **문서 버전** | v0.1 |
| **작성일** | 2026-11-10 (KST) |
| **구분 (산출물 분류)** | 8. 개발 · 형상 관리 |
| **기준** | `main` `9725f0b7` (PR #634 머지) · 작업 트리 기준 실측 |
| **목적** | 코드 · 설정 · 테스트 · 형상 · 문서(산출물) 구조를 소프트웨어 공학 관점에서 점검하고, 보완할 점을 우선순위와 함께 적는다 |
| **관련 문서** | [코딩표준](conventions/코딩표준.md) · [CI/CD 명세](../05-deployment/CI-CD명세.md) · [로컬실행안내서](../05-deployment/로컬실행안내서.md) · [산출물목록](../../산출물목록.md) · [docs 폴더 지도](../../README.md) |

> **한 줄** — 의존성 고정 · 시험 · 문서 추적성은 평균을 크게 웃돈다. 반면 **환경(local / dev / prod) 구분이 없고, 자동으로 도는 검사(CI)가 없고, 품질 도구가 문서로만 있다.** 이 셋은 발표 뒤 하루 안에 메울 수 있는 크기다.

---

## 1. 전체 모습

Qurious 는 FastAPI 모놀리스 + Celery(worker · beat) 를 PostgreSQL · Redis · Neo4j · Qdrant · Ollama 와 함께 docker compose 한 장으로 띄우는 구조다. 브라우저는 8966 포트 하나로 닿는다.

**<표 1> 규모 (2026-11-10 실측)**

| 항목 | 값 | 셈한 방법 |
|---|---|---|
| Python 코드 | 약 108,000줄 | `app` · `collector` · `scripts` · `tools` 의 `*.py` 합 |
| 라우트 모듈 / 서비스 모듈 | 44 / 약 200 | `app/routes` · `app/services` 파일 수 (services 하위 패키지는 `brokers` 하나) |
| 500줄 넘는 서비스 | 21개 | 최대 `paper_trading.py` 1,748줄 · `auto_trade.py` 1,554줄 |
| 프런트 | `app.html` 255KB · JS 96파일 26,700줄 | Vanilla JS · 번들러 없음 |
| 테스트 함수 | 3,283개 · 354파일 | `def test_` 수 |
| Alembic 리비전 | 59개 · merge 6개 · head 1개 | `revision` / `down_revision` 파싱 |
| 커밋 / 기여자 | 1,148 / 5명 | `git log` |
| 커밋 종류 | docs 225 · feat 177 · fix 165 · test 25 | Conventional Commits 접두 |
| docs 폴더 | 534MB · md 2,293개 · 추적 파일 약 2,800개 | `du` · `git ls-files` |
| `.git` 크기 | 468MB | — |

### 1.1 잘 되어 있는 것

| 영역 | 내용 | 근거 |
|---|---|---|
| 의존성 고정 | `requirements.lock`(uv) 으로 간접 의존까지 판 고정 · 바탕 이미지와 compose 외부 이미지는 「태그@다이제스트」 | `Dockerfile` · `docker-compose.yml` · `tests/test_deps_lock.py` 가 규칙을 시험으로 지킴 |
| 안전 가드 | 실거래 승인 환경변수를 테스트마다 지움 · 외부 사이트 · 알림 채널 호출을 테스트에서 차단 | `tests/conftest.py` autouse 고정 |
| 문서 ↔ 코드 동기화 | 라우트 · 스키마 · RTM · 문서 링크를 스캐너가 세고, 어긋나면 `--check` 가 멈춤 | `scripts/*_scan.py` 13개 · `tests/test_*_scan.py` |
| 마이그레이션 | 리비전 59개가 head 하나로 모임 · 기동 시 자동 적용과 배포 단계 수동 적용을 설정으로 고를 수 있음 | `RUN_MIGRATIONS_ON_STARTUP` · `scripts/migrate.sh` |
| 비밀값 | 증권사 · 알림 키는 저장소 밖 키 파일로 암호화 · JWT 자리표시자는 토큰 발급 거부 | `app/lib/broker_secrets.py` · `app/lib/jwt_auth.py` |
| 의사결정 기록 | ADR · 결정 대장 · RTM · 정본 인덱스(산출물목록) | `docs/common/02-design/adr/` 등 |
| 개발 모드 | 코드 폴더 bind mount + `--reload` · 일회용 시험 DB 로 전체 시험 | `scripts/personal/compose.dev.yml` · `test.ps1` |

---

## 2. 보완할 점

### 2.1 환경(local / dev / prod) 구분이 없다 — 가장 큰 구조적 결함

**현상**

| # | 현상 | 위치 |
|---|---|---|
| 1 | compose 는 한 장이고 `env_file: .env` 를 읽는데, 앱 설정은 `ENV_FILE` 이 없으면 `.env.dev` 를 읽는다. `.env.prod` 는 `.gitignore` 에만 있고 어디서도 쓰지 않는다. `.env.example` 은 「`.env` 또는 `.env.dev` 로 복사」 라고 안내한다 | `docker-compose.yml` · `app/config.py:261` · `.gitignore` · `.env.example` |
| 2 | 그래서 컨테이너 안과 호스트 스크립트가 서로 다른 설정 파일을 본다. 예약 작업이 `.env.dev` 만 읽어 알림 채널이 비는 사고가 있었고, 우회 코드(`fill_channels_from_dotenv`)가 들어갔다 | `scripts/collect_report.py:30 · 116` |
| 3 | 설정 기본값이 기초 코드의 `lumina:lumina@localhost` 라 compose 의 `fin_user:fin_pass@postgres` 와 다르다. 일일 갱신 러너는 「다른 DB 에 말없이 쓸 수 있어 돌리지 않는다」 는 방어 코드를 둔다 | `app/config.py` `DATABASE_URL` · `scripts/daily_update.py:1027` |
| 4 | DB 비밀번호 `fin_pass` · Neo4j `finagent123` 이 compose · config.py 기본값 · `dev.ps1` 에 하드코딩. `SESSION_SECRET` 기본값은 거부 로직 없이 남아 있다(config.py 밖에 참조가 없어 사실상 죽은 설정) | `docker-compose.yml` · `app/config.py:8 · 78` |
| 5 | `COOKIE_SECURE=false` · `TRADINGVIEW_ENFORCE_IP=false` · `RUN_MIGRATIONS_ON_STARTUP=true` 처럼 개발용 기본값을 운영에서 뒤집어 주는 프로필이 없다 | `app/config.py` |
| 6 | 환경별 덧씌우기(`compose.dev.yml` · `compose.qurious.yml` · `compose.local.yml`)가 `scripts/personal/` 아래 PowerShell 전용으로 있다. 루트 `start.sh` 는 외부 `redis` 컨테이너와 외부 `shared-net` 네트워크를 전제로 하는 별도 기동 경로다. 기동 방식이 OS 별로 둘로 갈라졌다 | `scripts/personal/start.ps1` · `start.sh` |
| 7 | `networks.shared-net.external: true` 라 `docker compose up` 전에 네트워크를 손으로 만들어야 한다. 숨은 선행 조건이다 | `docker-compose.yml` 끝 |

**제안**

1. `Settings` 에 `APP_ENV: Literal["local","dev","prod"]` 한 칸을 둔다. `prod` 이면 자리표시자 비밀값 · `COOKIE_SECURE=false` · `JWT_ALLOW_INSECURE_SECRET=true` 를 **기동 거부**로 처리한다(지금 JWT 에만 있는 `_require_real_secret` 을 일반화).
2. compose 를 셋으로 나눈다 — `compose.yaml`(공통) · `compose.override.yaml`(로컬 · docker 가 자동으로 겹침 · 포트 노출 · 개발 비밀번호) · `compose.prod.yaml`(포트 비노출 · 비밀값은 `secrets:` 또는 환경변수 · `restart` · 리소스 상한). `scripts/personal/compose.*.yml` 의 내용이 그대로 재료다.
3. 설정 파일 이름을 `.env` 하나로 통일하고 `.env.dev` 기본값을 없앤다. 환경은 파일 이름이 아니라 `APP_ENV` 로 가른다.
4. `start.sh` 와 `start.ps1` 을 OS 중립 러너 하나(Makefile 또는 `python -m tools.run`)로 합친다. 네트워크 생성 · `.env` 점검 · 프로필 선택을 거기서 한다.
5. config.py 의 기초 코드 잔재(`KIS_SECRETS_NAME` 등 AWS 칸 · `SESSION_SECRET` · `lumina` 주소)를 걷어낸다.

### 2.2 CI 가 없다

- `.github/` 가 루트에 없다. ADR-0003 으로 **의도적으로** 걷어낸 상태인데, 그 뒤 시험이 23건에서 3,283건으로 늘었다(CI/CD 명세 v1.2 의 「시험 23건」 은 그때 숫자).
- 지금은 한 사람 PC 의 `test.ps1` 결과를 테스트계획서에 손으로 옮긴다. 브랜치 보호(룰셋)도 없다.
- 최근 커밋 30개가 전부 docs 다. 코드 변경이 눈에 띄지 않는 데에는 자동 검증이 없는 구조도 한몫한다.
- **제안** — PR 마다 (a) DB 없이 도는 시험 부분집합 (b) `test_deps_lock` (c) ruff 만 돌리는 최소 워크플로. DB 시험은 `services: postgres` 로 한 단계 더. 형태는 `rag-lab/.github/workflows/ci.yml` 이 이미 갖고 있다. ADR-0003 의 「논의 먼저」 는 존중하되, 논의의 재료로 이 숫자를 올린다.

### 2.3 품질 도구가 문서로만 존재한다

| 현상 | 위치 |
|---|---|
| `pyproject.toml` · ruff · flake8 · mypy · pre-commit · `.editorconfig` 가 하나도 없다. 코딩표준은 문서로만 있다 | 루트 |
| `.prettierignore` 는 있는데 prettier 설정 · `package.json` 이 없다 | `.prettierignore` |
| `app` 안에 `print()` 38곳과 `getLogger` 59모듈이 섞여 있다. 기동 lifespan 이 `print` 로 로그를 쓴다 | `app/main.py` |
| 백업 파일 6개가 추적된다 | 아래 |

```
app/routes/tradingview.py.bak_fills
app/routes/tradingview.py.bak_syntax
app/services/lean_backtest.py.bak_fills
app/services/lean_backtest.py.bak_total_fees
app/services/tradingview.py.bak_fills
tests/test_tradingview.py.bak_fills
```

**제안** — `pyproject.toml` 에 ruff(lint + format) · 대상 파이썬 3.12 를 적고, pre-commit 으로 ruff · 큰 파일 · 비밀값 검사를 건다. `.bak*` 는 지우고 `.gitignore` 에 `*.bak*` 를 더한다. `print` 는 `logging` 으로 바꾸고 포맷을 한 곳에서 정한다.

### 2.4 모듈 경계가 약하다

| 현상 | 수치 | 제안 |
|---|---|---|
| `app/services` 가 단층 폴더 | 약 200파일 · 하위 패키지는 `brokers` 하나 · 500줄 넘는 파일 21개 | 파일 접두(`paper_` · `rebalance_` · `kis_` · `kb_` · `data_` · `strategy_`)가 이미 도메인을 말한다. 그대로 패키지로 묶는다 |
| 라우트가 SQL 을 직접 실행 | 44개 중 23개가 `select(` · `session.execute` 사용 | 업무 규칙을 서비스로 내리고 라우트는 요청 · 응답 변환만 |
| `main.py` 가 라우터 43개를 한 줄로 임포트 | — | 도메인 패키지마다 `router` 를 모아 `include_router` 를 패키지 쪽으로 |
| 전역 `settings = Settings()` 를 60모듈이 임포트 | — | 지금은 괜찮으나 2.1 의 환경 분리 때 걸림돌. 함수 주입 또는 `lru_cache` 로 바꿀 여지 |
| 프런트 한 파일 | `app.html` 255KB · JS 96파일 · 번들러 · JS 단위시험 없음 · prettier 가 `app.html` · `core.js` 를 제외(diff 충돌) | 제외 자체가 파일이 너무 크다는 신호. 최소한 화면 단위로 HTML 조각을 나눈다 |
| e2e 재현 불가 | Playwright 스크립트 33개가 자동 수집되지 않고 `playwright` 가 `requirements-dev.txt` 에 없다 | 의존을 적고 `tests/e2e` 를 마커로 선택 실행 |
| 운영 스크립트 비대 | `daily_update.py` 2,584줄 · `hf_dataset.py` 2,534줄이 `app` 밖 · Dockerfile 이 스크립트 8개를 이름으로 골라 복사 | 컨테이너가 쓰는 스크립트는 `app/cli/` 같은 패키지로 옮겨 통째로 복사 |
| Alembic 이름 규칙 혼재 | `0001_…` 번호 · 해시 이름 · `crypto_order_costs_20261109` 날짜 이름 · 번호 겹침 4건(0012 · 0037 · 0040 · 0044) | head 는 하나라 동작은 문제없음. 규칙을 하나로 정하고 `alembic revision` 템플릿에 반영 |
| 컨테이너 권한 | root 로 실행 · `chmod 777 data` · `/var/run/docker.sock` 마운트(LEAN 실행용) | 로컬 시연에는 허용. 운영 전 차단 요건으로 배포계획서에 명시 |

### 2.5 저장소가 무겁다

- 발표자료 HTML 한 파일 판 50MB + 29MB, webm · mp4 8 ~ 16MB 가 Git LFS 없이 커밋돼 `.git` 이 468MB 다. 2차 영상은 HF 비공개로 보내고 3차 영상은 git 에 넣기로 결정이 바뀌어 기준이 일관되지 않다.
- `rag-lab/`(59MB · 299파일)은 자체 Dockerfile · compose · CI · tests 를 가진 다른 프로젝트의 복사본이다. 반입대장이 있으니 서브모듈로 바꾸거나 반입한 파일만 남긴다.
- `data/lean-workflows/` 결과 JSON · `screenshots/` 같은 생성물이 추적된다.
- **제안** — 바이너리(mp4 · webm · 한 파일 HTML · png 5MB 이상)는 LFS 또는 GitHub Release 로 옮기고 `.gitattributes` 로 규칙을 적는다.

---

## 3. 산출물(docs) 구조

### 3.1 잘 된 점

- `phase-N/00-management … 06-retrospective` 10구분 · `common/` 공통 · 정본 인덱스(산출물목록) 의 세 층은 명확하고, 스캐너가 링크 · 수치를 지킨다.
- ADR · 결정 대장 · RTM · 테스트계획서가 실제 실행 결과 숫자를 담는다.

### 3.2 보완할 점

| # | 현상 | 제안 |
|---|---|---|
| 1 | 산출물목록이 1,557줄이고 「S119 에 바뀐 칸 …」 식 세션 변경 내역이 본문의 절반이다. **인덱스가 로그가 됐다** | 인덱스(무엇이 · 어디에 · 어떤 상태)와 변경 이력을 파일로 나눈다. 이력은 `CHANGELOG` 성격의 별도 문서로 |
| 2 | `docs/지난판/` 에 산출물목록 v3.9 ~ v3.22 열두 판이 파일로 남아 있다 | git 이 이력을 가진다. 파일명 판본은 걷어내고 필요하면 태그로 대체 |
| 3 | `common/00-management/github-archive/` 1,010파일이 GitHub 이슈 · PR 을 저장소 안에 복제한다. 추적 파일의 약 3분의 1 | 보존 의도는 이해되나, 별도 아카이브 저장소나 export 산출물로 분리 |
| 4 | phase-3 의 `03-development` 는 3파일, `02-design` 은 254파일 | 설계 · 시험 대비 개발 규약 · 코드 구조 문서가 비었다. 2.4 의 모듈 경계 문제가 문서에도 드러난다. 모듈 지도 · 의존 방향 · 환경 프로필 문서를 여기에 |
| 5 | phase-2 → phase-3 복사(사본 76)와 `common/` 사이의 정본이 어디인지 docs/README 를 읽어야 안다 | 2차는 태그 `phase2-final-2026-10-21` 로 얼리고 3차만 살아 있는 문서로 |
| 6 | 문서 머리 표의 「판 · 상태」 칸이 한 문단이다 | 메타데이터는 한 줄, 이력은 부록으로 |
| 7 | 커밋 비율 docs 225 : feat 177 : fix 165 · 최근 30커밋 전부 docs | 교육 과정 산출물로는 자연스러우나 유지 비용이 크다. 스캐너가 세는 숫자는 문서에 손으로 적지 않고 생성하게 하면 docs 커밋이 줄어든다 |

---

## 4. 우선순위

| 순위 | 할 일 | 크기 | 효과 |
|---|---|---|---|
| 1 | `APP_ENV` + compose 셋 분리 + 비밀값을 compose 밖으로 + `.env` 이름 통일 (2.1) | 반나절 | 환경 사고 재발 차단 · 운영 전환 가능 |
| 2 | 최소 CI (시험 부분집합 · deps-lock · ruff) (2.2) | 반나절 | 다섯 사람의 PR 이 같은 기준으로 검사됨 |
| 3 | `pyproject.toml` · ruff · pre-commit · `.bak` 삭제 · `print` → `logging` (2.3) | 반나절 | 코딩표준이 도구로 집행됨 |
| 4 | LFS · rag-lab 분리 (2.5) | 하루 | clone · PR 가벼워짐 |
| 5 | services 도메인 패키지화 · 라우트의 SQL 을 서비스로 (2.4) | 며칠 | 변경 영향 범위가 읽힘 |
| 6 | 산출물목록 인덱스 · 이력 분리 · 지난판 정리 (3.2) | 반나절 | 인덱스가 다시 인덱스가 됨 |

1 ~ 3 은 발표(2026-11-12) 뒤 하루 안에 끝낼 수 있는 크기다.

---

## 부록 A. 만든 방법

작업 트리에서 다음을 직접 셌다 — `git ls-files` · `git log` · `wc -l` · `du` · `find` · `grep`(`print(` · `getLogger` · `select(` · `from app.config import settings`) · Alembic `revision` / `down_revision` 파싱 스크립트. 사람이 적은 숫자는 쓰지 않았다.

## 부록 B. 개정 이력

| 판 | 날짜 | 내용 |
|---|---|---|
| v0.1 | 2026-11-10 | 첫 판 |
