# sj-marketplace

SJ의 Claude 플러그인 마켓플레이스입니다.

## 설치

```bash
# 1. 마켓플레이스 등록 (GitHub 아이디/저장소명으로 교체)
claude plugin marketplace add <github-id>/sj-marketplace

# 2. 플러그인 설치
claude plugin install shopping-shorts@sj-marketplace
```

## 플러그인 목록

| 플러그인 | 버전 | 설명 |
|---|---|---|
| [shopping-shorts](plugins/shopping-shorts/) | 0.3.0 | 쇼핑쇼츠(유튜브 쇼츠) 풀세트 제작 — 스크립트·스토리보드·AI 영상 생성·발행 키트 4스킬 체인 + 스크립트 진단 모드 |

## 사용 시작

설치 후 Claude Code에서:

```
> 무선 미니 가습기로 쇼핑쇼츠 만들어줘. 타깃은 사무실 직장인
```

레퍼런스(제품 사진·벤치마킹 쇼츠)를 물어보는 것부터 시작해 발행용 해시태그까지 체인으로 진행됩니다.

## 업데이트

```bash
claude plugin marketplace update sj-marketplace
```
