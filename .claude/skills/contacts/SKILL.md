---
name: contacts
description: macOS 연락처(Contacts/AddressBook)를 읽는 읽기 전용 CLI. 이름·소속·번호·이메일로 사람을 찾고, 전화/이메일 핸들을 이름으로 역조회한다. 어느 폴더·세션에서든 셸 명령 `contacts`로 실행. "OO 번호/이메일 뭐였지", "OO 소속이 어디", "이 번호 누구야", 메일·메시지 답장 전 상대 연락처 확인 등 연락처 조회 작업이면 이 스킬을 먼저 본다. (추가·수정은 범위 밖.)
---

# contacts — macOS 연락처 읽기 CLI

macOS **연락처(Contacts)** 앱의 로컬 DB(`~/Library/Application Support/AddressBook/**/AddressBook-v22.abcddb`)를
**읽기 전용**으로 읽는 Python CLI. **Claude 슬래시 스킬이 아니라 PATH에 설치된 셸 명령**이다
(`~/.local/bin/contacts`). 어느 폴더·세션에서든 `Bash`로 `contacts <subcommand>` 호출하면 된다.
소스: `~/dev/contacts-cli/contacts` (stdlib-only, 의존성 0).

`msg`(messages-cli)와 짝을 이루는 **별도 도구**다. `msg`도 연락처 이름 표시를 내부적으로 하지만,
이 CLI는 **전체 카드(여러 번호·이메일·라벨·소속·직함)**를 노출한다.

## 전제
- **Full Disk Access 불필요** — AddressBook은 그냥 읽힌다(메시지 chat.db와 다름).
- 원본 DB는 절대 안 건드린다(각 DB를 `mode=ro&immutable=1`로 직접 open). 디스트럭티브 동작 없음.
- 여러 소스(iCloud/Google/local)를 합치고 중복은 병합한다.

## 명령
```
contacts <query>                    # = search. 이름·소속·번호·이메일 부분일치 카드 목록
contacts search <query> [--json]    # 위와 동일(명시형)
contacts show <query> [--json]      # 단일 인물 전체 카드 (여러 매칭이면 후보만 제시)
contacts lookup <handle> [--json]   # 역방향: 전화/이메일 → 이름
contacts list [--limit N] [--json]  # 전체 연락처 간략 목록
```

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
연락처 추가·수정·삭제(읽기 전용). vCard 내보내기, 탭 자동완성은 향후 후보.
(연락처를 iCloud로 *쓰는* 작업은 별개 파이프라인 `ji-google-workspace-admin/contacts/` 참조.)
