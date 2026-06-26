# contacts-cli

macOS **연락처(Contacts / AddressBook)**를 읽는 읽기 전용 CLI. 이름·소속·번호·이메일로
사람을 찾고, 전화/이메일 핸들을 이름으로 역조회한다. Python 3 stdlib-only, 의존성 0.

`messages-cli`(`msg`)와 짝을 이루는 별도 도구다. `msg`가 메시지 표시용으로 연락처 *이름*만
해석하는 데 비해, `contacts`는 **전체 카드**(여러 번호·이메일·라벨·소속·직함·생일)를 노출한다.

## 설치
```sh
ln -sf "$PWD/contacts" ~/.local/bin/contacts   # PATH에 ~/.local/bin 가정
# Claude/Codex 스킬로도 쓰려면:
ln -sf "$PWD/.claude/skills/contacts" ~/.claude/skills/contacts
```

## 사용
```sh
contacts <query>                    # = search. 이름·소속·번호·이메일 부분일치
contacts search <query> [--json]
contacts show <query> [--json]      # 단일 인물 전체 카드(여러 매칭이면 후보 제시)
contacts lookup <handle> [--json]   # 전화/이메일 → 이름 역조회
contacts list [--limit N] [--json]  # 전체 간략 목록
```

## 설계 노트
- 데이터: `~/Library/Application Support/AddressBook/**/AddressBook-v22.abcddb`.
  실제 연락처는 대부분 `Sources/<UUID>/` 하위 DB들에 있다(메인 DB는 비어있을 수 있음).
  `**` glob으로 모두 읽어 합친다.
- **읽기 전용·비파괴**: 각 DB를 `file:…?mode=ro&immutable=1` URI로 직접 연다.
  복사도 쓰기도 없다. **Full Disk Access 불필요**(메시지 chat.db와 다른 점).
- **이름 조합**: CJK는 성+이름(`compose_name`), 서양식은 First Middle Last.
- **전화 매칭**: 국가코드 무관하게 끝 8자리. 이메일은 소문자.
- **멀티소스 병합**: 같은 이름끼리 묶은 뒤, 핸들(전화8/이메일)을 하나라도 공유하면
  union-find로 한 사람으로 병합. 핸들이 안 겹치는 동명이인은 따로 둔다(과병합 방지).

## 범위 밖
추가·수정·삭제(읽기 전용), vCard 내보내기, 탭 자동완성. 연락처를 *쓰는* 쪽은
`ji-google-workspace-admin/contacts/`(vCard→iCloud) 참조.
