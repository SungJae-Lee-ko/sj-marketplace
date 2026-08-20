# shopping-shorts

쇼핑쇼츠(유튜브 쇼츠) 제작 풀세트 플러그인. 상품 정보 하나로 스크립트 → 스토리보드 → 발행 키트까지 만듭니다.

## 스킬 체인

```
shorts-script          shorts-storyboard        shorts-publish-kit
(후킹 스크립트)   →    (씬별 촬영/AI 콘티)   →   (제목·설명·해시태그)
```

| 스킬 | 하는 일 | 트리거 예시 |
|------|---------|------------|
| `shorts-script` | 30-45초 판매 스크립트 (훅→문제→해결→증거→CTA) | "이 상품으로 쇼츠 대본 써줘" |
| `shorts-storyboard` | 씬별 화면·앵글·자막·AI 생성 프롬프트 | "스토리보드 만들어줘" |
| `shorts-publish-kit` | 제목 3안·설명문·해시태그·고정댓글·체크리스트 | "업로드 준비해줘" |

각 스킬은 단독으로도 쓸 수 있고, 순서대로 체이닝하면 풀세트가 나옵니다.

## 설치 (Claude Code)

```bash
# 마켓플레이스 등록
claude plugin marketplace add <your-github-id>/<repo-name>

# 플러그인 설치
claude plugin install shopping-shorts@sj-marketplace
```

## 사용 예시

```
> 무선 미니 가습기로 쇼핑쇼츠 만들어줘. 타깃은 사무실 직장인
```

→ shorts-script가 발동해 스크립트를 만들고, 확정하면 자동으로 다음 단계를 제안합니다.

## 구조

```
shopping-shorts/
├── .claude-plugin/plugin.json
├── skills/
│   ├── shorts-script/
│   │   ├── SKILL.md
│   │   └── references/hook-patterns.md
│   ├── shorts-storyboard/SKILL.md
│   └── shorts-publish-kit/SKILL.md
└── README.md
```

## 라이선스

MIT
