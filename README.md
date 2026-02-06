# Demon Slayer

맵 탐험형 턴제 로그라이크 게임

## 게임 소개

마을에서 출발하여 마왕성까지 도달해 마왕을 처치하는 것이 목표입니다.
침공 게이지가 최대치에 도달하기 전에 보스를 물리쳐야 합니다.

## 개발 환경

- **Engine**: Unity 6 (6000.3.6f1)
- **Language**: C#
- **IDE**: Visual Studio / Rider / VS Code

## 프로젝트 설정

### 필수 요구사항

- Unity Hub 설치
- Unity 6000.3.6f1 버전 설치

### 프로젝트 열기

1. 이 저장소를 클론합니다:
   ```bash
   git clone <repository-url>
   cd demon-slayer
   ```

2. Unity Hub에서 프로젝트를 추가합니다:
   - Unity Hub 실행
   - `Add` → `Add project from disk` 선택
   - `demon-slayer` 폴더 선택

3. Unity 6000.3.6f1 버전으로 프로젝트를 엽니다.

## 문서

| 문서 | 설명 |
|------|------|
| [plan.md](plan.md) | 게임 디자인 문서 (GDD) |
| [dev-plan.md](dev-plan.md) | 기술 스택 및 개발 계획 |
| [build-test-guide.md](build-test-guide.md) | 빌드 및 테스트 가이드 |
| [CLAUDE.md](CLAUDE.md) | AI 어시스턴트 가이드 |

## 브랜치 전략

| 브랜치 | 용도 |
|--------|------|
| `main` | 안정 버전 (릴리스) |
| `develop` | 개발 통합 브랜치 |
| `feature/*` | 기능 개발 |
| `bugfix/*` | 버그 수정 |

## 테스트 실행

Unity Editor에서:
1. `Window` → `General` → `Test Runner` 열기
2. `EditMode` 또는 `PlayMode` 탭 선택
3. `Run All` 클릭

## 빌드

Unity Editor에서:
- `Build` → `Build Windows` (개발용)
- `Build` → `Build All Platforms` (전체 빌드)

## 라이선스

Private - All rights reserved
