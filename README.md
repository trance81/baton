<div align="center">

<img src="assets/logo-tagline.png" alt="Baton" width="180">

# Baton

**세션이 끊겨도 하던 일이 이어지는 AI 코딩 에이전트 스킬**

작업 하나당 마크다운 한 장. 끝나도 지우지 않습니다.

[![License: 0BSD](https://img.shields.io/badge/License-0BSD-black.svg)](LICENSE)
[![Agent skill](https://img.shields.io/badge/AI%20agent-skill-d97757)](SKILL.md)
[![Zero deps](https://img.shields.io/badge/dependencies-0-brightgreen)](templates/baton-stop-hook.mjs)

한국어 · [English](README.en.md)

</div>

---

## 설치

```bash
npx skills add trance81/baton -g
```

그리고 쓰는 에이전트에게 한마디면 됩니다.

```
이 프로젝트에 baton 추가해줘
```

## 무엇이 달라지나

CLI 세션은 꺼지면 기억을 잃습니다. `resume` 목록이 있지만 며칠 쌓이면 제목과 시각만 보고는
어느 게 뭐였는지 알 수 없습니다.

<table>
<tr><th width="50%">baton 없이</th><th width="50%">baton 있을 때</th></tr>
<tr valign="top"><td>

```
"결제 웹훅 마저 하자"

→ diff를 훑어 어디까지
  했는지 추론

→ 왜 그렇게 했는지는
  코드에 없음

→ 사람이 다시 설명
```

</td><td>

```
"결제 웹훅 마저 하자"

→ .baton/payment-webhook.md
  (running) 읽음

→ "서명 검증까지 됐고 다음은
   retry 정책입니다. 큐는 안
   쓰기로 하셨죠 — 하루 수백
   건이라 과하다고."
```

</td></tr>
</table>

diff는 **무엇이 바뀌었는지** 알려줍니다. **무엇을 하려던 중이었는지, 왜 그 선택을 했는지**는
알려주지 않습니다. 그 둘이 배턴에 있습니다.

배턴은 끝나도 남습니다. 그래서 몇 달 뒤 이런 게 됩니다.

```
"스테이징 배포 어떻게 하더라?"
   → .baton/ 검색 → deploy-pipeline.md (passed)
   → 절차, 왜 그 순서인지, 그때 막혔던 것까지 그대로
```

## 4대 원칙

```
1. .baton/<task>.md 가 작업 상태의 유일한 정본이다.
2. 모든 턴을 기록하지 않는다 — 다음 세션에 필요한 상태만 적는다.
3. 사이드카는 로컬 복구 단서일 뿐이다.
4. 과거 배턴은 git에 영구히 남기되, 컨텍스트에는 필요할 때만 올린다.
```

## 어떻게 쓰나

**1 · 설치** — 스킬이 `.baton/README.md`(운영 규칙)를 만들고, 진입 파일(`CLAUDE.md`·`AGENTS.md`
등 그 프로젝트가 쓰는 것)에 포인터 한 줄을 넣고, 턴이 끝날 때 도는 훅을 등록하고, `.gitignore`를
손봅니다. 이미 있는 조각은 건드리지 않으니 다시 실행해도 안전합니다.

**2 · 시작** — 작업이 오늘 안에 안 끝날 것 같으면 배턴이 생깁니다. 오타 수정 같은 건 만들지
않습니다.

```markdown
---
status: running
updated: 2026-08-19T09:05:00Z
branch: feature/payment-webhook
---

# Payment webhook

## 지금 상태
엔드포인트 뼈대만. 서명 검증 없음.

## 다음 할 일
서명 검증 붙이기.
```

필수 필드는 `status`와 `updated` 둘뿐입니다.

**3 · 재개** — 새 세션은 `.baton/` 최상위의 `running`·`waiting`만 읽고 그 맥락에서
이어갑니다. 과거 배턴을 전부 올리지는 않습니다. 그러면 baton이 스스로 컨텍스트를
오염시킵니다.

**4 · 완료** — `status: passed`로 두고 남깁니다. 몇 달 뒤 "그거 어떻게 했더라" 싶을 때
검색하는 자리가 여기입니다.

## 폴더 구조

```
.baton/
├─ README.md        규칙                      git
├─ <slug>.md        작업·기능 단위 배턴         git
├─ done/            아카이브 (선택)            git
├─ local/           로컬 전용 메모 (선택)       git 제외
└─ .session/        훅이 쓰는 런타임 기록       git 제외
```

## 무엇을 적고 무엇을 안 적나

| | 적는다 | 안 적는다 |
|---|---|---|
| 접속 | `ssh prod-web-01` (`~/.ssh/config` alias) | 비밀번호·개인키 값 |
| 키 | 1Password `"prod-server-ssh"` 항목 | `AWS_SECRET_ACCESS_KEY=...` |
| 절차 | `scripts/deploy.sh` 실행 후 nginx reload | 토큰 문자열 |

> [!WARNING]
> `.baton/*.md`는 git에 올라갑니다. 팀이 있으면 팀 전체가, 공개 저장소면 누구나 봅니다.
> **어디에 있는지, 뭐라고 부르는지만 적습니다.** 그것만으로 다음 세션이 접속 방법을 알아내는
> 데 충분하고, 값이 git 이력에 박히는 일은 없습니다.

기기에만 있는 설정값이나 팀에 공유 못 할 개인 자료는 `local/`에 두면 git에 올라가지
않습니다. 다만 **참고용**이라 언제든 지워질 수 있으니, 거기 있는 내용에 의존하는 절차를
만들지 않습니다.

<details>
<summary><b>에이전트 내장 memory와 뭐가 다른가</b></summary>

<br>

요즘 CLI 에이전트는 대개 자체 memory 기능을 갖고 있습니다. 공통점은 **기기 로컬**에 저장되고,
무엇을 기억할지 모델 판단에 맡긴다는 점입니다. 예를 들어 Claude Code의
[Auto memory](https://code.claude.com/docs/en/memory)는 `~/.claude/projects/<repo>/memory/`에
쌓이고, 공식 문서에도 "기기·클라우드 환경 간 공유 안 됨"이라고 적혀 있습니다.

| | 내장 memory | baton |
|---|---|---|
| 저장 위치 | 기기 로컬 | `.baton/*.md` — git 커밋 |
| 팀 공유 | 안 됨 | git push하면 공유됨 |
| 뭘 남기나 | 모델이 판단한 일반 학습 | 작업 하나의 확정된 상태 |
| 형태 | 모델이 관리하는 요약 노트 | 사람이 그대로 읽는 작업 문서 |

겹치지 않습니다 — 내장 memory가 안 하는 일(git으로 기기·사람 간 공유)을 baton이 메웁니다.

</details>

<details>
<summary><b>사이드카가 하는 일</b></summary>

<br>

턴이 끝날 때마다 훅이 `.baton/.session/<session-id>.json`을 다시 씁니다. 모델을 부르지
않으니 토큰이 0입니다.

```json
{
  "observedAt": "2026-08-19T17:54:00Z",
  "event": "Stop",
  "transcriptPath": "C:/Users/.../a1b2c3d4-....jsonl",
  "lastMessage": "서명 검증 끝, 다음은 retry backoff."
}
```

`observedAt`은 **훅이 이 세션을 마지막으로 관찰한 시각**입니다. 배턴이 최신인지까지는
말해주지 않습니다. 배턴이 미심쩍거나 세션이 비정상적으로 끝났을 때 열어보는 **복구
단서**입니다. `event`에는 훅을 건 도구의 이벤트 이름이 그대로 들어갑니다 — Claude Code에서
턴이 API 오류로 끝났으면 `StopFailure`가 되고 `error`가 함께 실립니다.

</details>

<details>
<summary><b>알아둘 제약</b></summary>

<br>

- **모든 종료를 관찰하지는 못합니다.** 사용자가 중간에 끊거나(Ctrl+C/ESC), 프로세스가 강제로
  죽거나, 전원이 나가면 훅이 아예 돌지 않습니다. baton은 이걸 잡으려 하지 않습니다. 정본은
  어차피 마크다운 파일이고, 사이드카는 최선을 다한 단서일 뿐입니다.
- **`.baton/` 파일 규약은 도구에 매이지 않습니다.** 파일을 읽고 쓰는 에이전트면 무엇이든
  진입 파일(`CLAUDE.md`·`AGENTS.md` 등)에 규칙을 적어두는 것으로 그대로 씁니다.
  사이드카 훅 스크립트도 마찬가지로 도구 중립이라, 턴 종료 시점에 JSON을 stdin으로 넘기는
  도구면 붙습니다. 바로 쓸 수 있는 설정 스니펫은 **Claude Code · Codex · Gemini CLI** 세 가지가
  들어 있고, 그 밖의 도구는 그 도구의 훅 설정에서 같은 스크립트를 직접 가리켜 주면 됩니다.
- git 없는 프로젝트에선 `branch`·`assignee` 필드가 빠집니다. 배턴 자체는 그대로 동작하지만
  **이력 계층이 통째로 없습니다** — 본문을 덮어쓰면 이전 내용을 되찾을 방법이 없습니다.
  아래 `git log` 항목이 그 자리를 메우는 걸 전제하기 때문입니다. 그런 프로젝트라면 배턴에
  "왜 이 방향" 절을 조금 더 성실히 채워두는 편이 낫습니다.
- `local/`은 git에 안 올라가므로 **다른 PC에 따라가지 않습니다.** 기기마다 다른 값이라
  그게 정상이지만 팀과 나눠야 할 내용이면 git 레인에 (값 말고 참조로) 적어야 합니다.
- 시간순 변경 이력은 baton이 갖지 않습니다. 본문은 덮어쓰기 때문에 그건 `git log`의 몫입니다.
  일부러 이렇게 뒀습니다 — 파일 안에 이력을 쌓으면 다음 세션이 읽을 양이 계속 늘어나고,
  정본이 둘로 갈립니다.
- **읽는 범위는 규약이지 강제가 아닙니다.** 훅은 셸 명령이라 모델이 무엇을 읽을지 통제하지
  못합니다. 원칙 4는 진입 파일과 `.baton/README.md`의 지시로 지켜지고, 어기면 컨텍스트가
  낭비됩니다. 규약 대신 `PreToolUse` 훅으로 `done/` 읽기를 차단할 수도 있지만 그러면 "예전에
  어떻게 했더라"를 찾는 길까지 같이 막힙니다. 막고 싶은 것과 살리고 싶은 것이 같은 동작이라
  강제하지 않습니다. 대신 `done/`을 하위 폴더로 둬서, 기본 경로가 곧 싼 경로가 되게 했습니다.

</details>

<details>
<summary><b>다른 설치 방법</b></summary>

<br>

이 프로젝트에만:

```bash
npx skills add trance81/baton
```

`.agents/skills/baton-init/`에 풀리고, 설치된 도구들의 스킬 폴더(`.claude/skills/` 등)에
심볼릭 링크가 걸립니다. 원본 하나만 갱신하면 링크된 도구 전부에 반영됩니다.

수동으로 — 쓰는 도구의 스킬 폴더에 그대로 복사합니다:

```bash
git clone https://github.com/trance81/baton.git
cp -r baton <스킬-폴더>/baton-init   # 예: ~/.claude/skills/baton-init
```

</details>

---

<div align="center">

0BSD · `main`이 항상 현재 규약입니다

</div>
