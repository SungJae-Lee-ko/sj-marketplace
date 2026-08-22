# sj-marketplace

SJ의 Claude 플러그인 마켓플레이스입니다.

## 설치

```bash
# 1. 마켓플레이스 등록
claude plugin marketplace add SungJae-Lee-ko/sj-marketplace

# 2. 플러그인 설치
claude plugin install shopping-shorts@sj-marketplace
```

## 플러그인 목록

| 플러그인 | 버전 | 설명 |
|---|---|---|
| [shopping-shorts](plugins/shopping-shorts/) | 0.6.0 | 쇼핑쇼츠(유튜브 쇼츠) 풀세트 제작 — 스크립트 → 스토리보드 → AI 영상 생성(인물 일관성 체인) → 자동 조립(완성 MP4) → 발행 키트 5스킬 체인 + 에이전트 2종 |

### shopping-shorts 구성

**스킬 체인 (5종)**

```
shorts-script ─→ shorts-storyboard ─→ shorts-video-gen ─→ shorts-assemble ─→ shorts-publish-kit
 (스크립트·진단)   (씬 설계·프롬프트)     (AI 생성·검수)      (자동 조립·렌더)     (발행 텍스트 자산)
```

**에이전트 (2종)**

- `shorts-producer` — "○○ 쇼츠 처음부터 끝까지 만들어줘" 한 마디로 5스킬 체인 전체 주행
- `shorts-auditor` — 발행 전 읽기 전용 검수: 스크립트 채점 · 정합성 · 영상 규격 · 인물/제품 일관성 · 법규 표현 PASS/FAIL

**핵심 설계**

- 인물 등장 씬은 스틸 고정 → 영상 변환 필수 체인 (identity switching 방지, 실패 사례 기반)
- 조립은 ffmpeg 자동 렌더 — 편집 앱 없이 완성 MP4 산출
- Higgsfield 등 영상 생성 MCP 자동 연동, 직접 전송 우선 (egress 허용 시 완전 자동)
- 타인 영상은 분석 전용, 자사 소재만 생성·조립 소스로 사용

변경 이력: [plugins/shopping-shorts/CHANGELOG.md](plugins/shopping-shorts/CHANGELOG.md)

## 사용 시작

설치 후 Claude Code 또는 Cowork에서:

```
> 무선 미니 가습기로 쇼핑쇼츠 만들어줘. 타깃은 사무실 직장인
```

레퍼런스(제품 사진·벤치마킹 쇼츠) 확인부터 완성 MP4·발행 키트까지 체인으로 진행됩니다.
