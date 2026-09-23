---
name: contacts
description: macOS 연락처(Contacts/AddressBook) CLI. 이름·소속·번호·이메일로 사람을 찾고(읽기 전용), 전화/이메일 핸들을 이름으로 역조회하며, 새 연락처를 추가(add, Contacts.app 경유·미리보기+확인)한다. 어느 폴더·세션에서든 셸 명령 `contacts`로 실행. "OO 번호/이메일 뭐였지", "OO 소속이 어디", "이 번호 누구야", "OO 연락처 추가해줘", 메일·메시지 답장 전 상대 연락처 확인 등 연락처 작업이면 이 스킬을 먼저 본다. (수정·삭제는 범위 밖.)
---

# contacts — macOS 연락처 읽기 CLI

macOS **연락처(Contacts)** 앱의 로컬 DB(`~/Library/Application Support/AddressBook/**/AddressBook-v22.abcddb`)를
**읽기 전용**으로 읽는 Python CLI. **Claude 슬래시 스킬이 아니라 PATH에 설치된 셸 명령**이다
(`~/.local/bin/contacts`). 어느 폴더·세션에서든 `Bash`로 `contacts <subcommand>` 호출하면 된다.
소스: `~/dev/contacts-cli/contacts` (stdlib-only, 의존성 0).

`msg`(messages-cli)와 짝을 이루는 **별도 도구**다. `msg`도 연락처 이름 표시를 내부적으로 하지만,
이 CLI는 **전체 카드(여러 번호·이메일·라벨·소속·직함)**를 노출한다.

## 전제
- **읽기(search/show/lookup/list)**: Full Disk Access 불필요, AddressBook을 `mode=ro&immutable=1`로
  직접 open — 원본 DB는 절대 안 건드린다. 여러 소스(iCloud/Google/local)를 합치고 중복은 병합한다.
- **쓰기(add)**: abcddb를 직접 쓰지 않고 **Contacts.app을 osascript로 구동**해 만든다(공식 경로,
  이후 iCloud 동기화). **Automation 권한** 필요 — 첫 실행 시 "터미널이 Contacts 제어" 팝업 승인.
  헤드리스/에이전트 실행은 권한이 없으면 친절 안내 후 종료 → 그대로 사용자에게 전달.

## 명령
```
contacts <query>                    # = search. 이름·소속·번호·이메일 부분일치 카드 목록
contacts search <query> [--json]    # 위와 동일(명시형)
contacts show <query> [--json]      # 단일 인물 전체 카드 (여러 매칭이면 후보만 제시)
contacts lookup <handle> [--json]   # 역방향: 전화/이메일 → 이름
contacts list [--limit N] [--json]  # 전체 연락처 간략 목록
contacts add [표시이름] [--first F --last L] [--org O] [--title T] \
             [--phone N ...] [--email E ...] [--dry-run] [--force]
```

### add (연락처 생성) — ⚠️ 외부/영구 동작
- **사용자가 명시적으로 "추가/저장해줘"라고 할 때만** 실행한다. 추측해서 만들지 않는다.
- 기본 동작 = **미리보기(생성될 카드) + 기존 중복 검색 + 확인**. 확신이 없으면 먼저 **`--dry-run`**으로
  미리보기를 사용자에게 보여주고 확정받는다.
- 비슷한 기존 연락처가 있으면 **중단**하고 후보를 보여준다 → 그래도 만들려면 `--force`.
- 비대화형(에이전트 Bash)에선 확인 프롬프트를 못 받으므로, 사용자 승인이 분명할 때만 `--force`로 만든다.
- 이름: `--first`/`--last`가 우선. 표시이름만 주면 **CJK·단일토큰은 성(last)에 통째로**, 서양식(공백)만
  마지막 토큰을 성으로 분해. 애매하면 `--first/--last`를 명시.
- 전화 라벨은 mobile, 이메일 라벨은 home 고정(MVP). 대상은 **기본 계정(보통 iCloud)**.
- 생성 직후엔 동기화·색인 지연으로 `search`에 바로 안 보일 수 있다.

### 매칭 규칙 (느슨함)
- **이름·별명·소속·직함·부서**: 부분 일치, 대소문자 무시(`현석` → 정현석·성현석 …).
- **전화번호**: 숫자만 추출해 **끝 8자리** suffix 매칭. `010…`이든 `+8210…`이든 무관.
- **이메일**: 소문자 정규화 매칭.
- `lookup`은 핸들 1개로 정확 조회(전화 끝8자리 / 이메일 소문자). 메시지·메일의 발신자
  핸들을 사람 이름으로 바꿀 때 쓴다(= `msg`의 이름 해석과 동일 기준).

## 사용 패턴 (에이전트용)
- **기계 파싱이 필요하면 항상 `--json`.** 사람용 출력은 라벨/이모지가 섞인다.
- 카드 JSON 스키마:
  ```
  { "name", "first", "last", "nickname", "org", "title", "department",
    "phones": [{"label","number"}], "emails": [{"label","address"}],
    "birthday": "YYYY-MM-DD"|null, "sources": ["<소스8>", …] }
  ```
  - `label`은 정규화됨(`mobile`/`home`/`work`/…), 커스텀 라벨은 소문자 원문.
  - `sources`는 이 카드가 합쳐진 소스 DB 식별자(중복 병합 흔적). 2개 이상이면 여러 계정에 있던 것.
- 메일/메시지 답장 전 상대 이메일·번호 확인: `contacts show "<이름>" --json`.
- "이 번호 누구야": `contacts lookup "+8210…" --json` 또는 `contacts lookup user@host --json`.
- 여러 명 매칭돼 애매하면 `show`가 후보 목록만 준다 → 더 구체적 쿼리로 좁혀라.

## 주의 / 함정
- **이름 표시는 소스 데이터 품질에 좌우된다.** 원본 연락처가 성/이름 칸을 잘못 채웠으면
  (예: 이름 칸에 풀네임, 소속 칸에 직함) 그대로 반영된다 — CLI 버그가 아님.
- **중복 병합은 같은 이름끼리만** 핸들 공유로 합친다. 이름 표기가 다른 같은 사람(예: 소속이
  이름에 붙은 변형)은 별도 카드로 남을 수 있다. 동명이인 과병합을 피하려는 안전한 기본값이다.
- **프라이버시**: 사용자의 개인 연락처다. 요청 범위를 넘어 무단으로 전체를 덤프하지 말 것.

## 범위 밖
기존 연락처 **수정·삭제**(추가 add는 지원). vCard 내보내기, 탭 자동완성, 라벨/계정 지정은 향후 후보.
(연락처를 iCloud로 **대량** 넣는 작업은 별개 파이프라인 `ji-google-workspace-admin/contacts/`(조직 명부 CardDAV 동기화) 참조 — 이 CLI의 add는 1건 대화형 추가용.)
