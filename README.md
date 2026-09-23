# contacts-cli

macOS **연락처(Contacts / AddressBook)**를 읽는 읽기 전용 CLI. 이름·소속·번호·이메일로
사람을 찾고, 전화/이메일 핸들을 이름으로 역조회한다. Python 3 stdlib-only, 의존성 0.

`messages-cli`(`msg`)와 짝을 이루는 별도 도구다. `msg`가 메시지 표시용으로 연락처 *이름*만
해석하는 데 비해, `contacts`는 **전체 카드**(여러 번호·이메일·라벨·소속·직함·생일)를 노출한다.

## 설치
```sh
ln -sf "$PWD/contacts" ~/.local/bin/contacts   # PATH에 ~/.local/bin 가정
# 단일 SKILL.md(SSOT)를 Claude·Codex 양쪽에 심링크:
ln -sf "$PWD/.claude/skills/contacts" ~/.claude/skills/contacts
ln -sf "$PWD/.claude/skills/contacts" ~/.codex/skills/contacts
```

## 사용
```sh
contacts <query>                    # = search. 이름·소속·번호·이메일 부분일치
contacts search <query> [--json]
contacts show <query> [--json]      # 단일 인물 전체 카드(여러 매칭이면 후보 제시)
contacts lookup <handle> [--json]   # 전화/이메일 → 이름 역조회
contacts list [--limit N] [--json]  # 전체 간략 목록

# 연락처 추가 (Contacts.app 경유, 미리보기+확인. Automation 권한 필요)
contacts add "홍길동" --phone 010-1234-5678 --email a@b.com --org "회사" [--title 팀장]
contacts add --first John --last Smith --phone ... [--dry-run] [--force]
```

## 설계 노트
- 데이터: `~/Library/Application Support/AddressBook/**/AddressBook-v22.abcddb`.
  실제 연락처는 대부분 `Sources/<UUID>/` 하위 DB들에 있다(메인 DB는 비어있을 수 있음).
  `**` glob으로 모두 읽어 합친다.
- **읽기는 read-only·비파괴**: 각 DB를 `file:…?mode=ro` URI로 직접 연다(-wal 사이드카까지 보여
  방금 add 한 연락처가 즉시 조회됨). 열기 실패 시 `immutable=1`로 폴백. 복사도 쓰기도 없다.
  **Full Disk Access 불필요**(메시지 chat.db와 다른 점).
- **이름 조합**: CJK는 성+이름(`compose_name`), 서양식은 First Middle Last.
- **전화 매칭**: 국가코드 무관하게 끝 8자리. 이메일은 소문자.
- **멀티소스 병합**: 같은 이름끼리 묶은 뒤, 핸들(전화8/이메일)을 하나라도 공유하면
  union-find로 한 사람으로 병합. 핸들이 안 겹치는 동명이인은 따로 둔다(과병합 방지).
- **add(쓰기)는 abcddb를 직접 안 건드린다**: Core Data+CardDAV로 동기화되는 DB라 직접 쓰면
  손상·덮어쓰기 위험. 대신 **Contacts.app을 osascript로 구동**(공식 경로). 이름/번호/이메일은
  AppleScript 문자열 보간이 아니라 **argv로 전달**해 인젝션을 원천 차단(messages-cli send와 동일).
  기본 = 미리보기 + 중복검색 + 확인, `--force`로 스킵, 비대화형은 `--force` 필수. **Automation 권한** 필요.

## 범위 밖
기존 연락처 **수정·삭제**(추가 add는 지원), vCard 내보내기, 탭 자동완성, 라벨/계정 지정.
연락처를 **대량**으로 iCloud에 넣는 쪽은 `ji-google-workspace-admin/contacts/`(조직 명부 CardDAV) 참조.
