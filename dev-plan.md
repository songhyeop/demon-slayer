# Development Plan — Demon Slayer

> 참조 문서: [plan.md](plan.md) (GDD), [CLAUDE.md](CLAUDE.md) (개발 가이드라인)

---

## Tech Stack (확정)

| 항목 | 선택 | 근거 |
|---|---|---|
| 엔진 | Unity (LTS) | 2D 타일 + UI 조합이 간단, C# 생태계 |
| 언어 | C# | Unity 표준 |
| 아키텍처 패턴 | State Machine | 맵/전투/상점 등 명확한 상태 전이 구조 |
| 정적 데이터 관리 | ScriptableObject | Inspector에서 디자인 조정 가능, 코드 변경 불필요 |
| 씨앤 구성 | 단일 씨앤 (Single Scene) | 상태 전환이 자주 일어나고 규모가 작아서 씨앤 전환 부담 불필요 |

---

## Architecture Overview

### State Machine (중앙 구조)

```
                    ┌──────────────┐
                    │  GameManager │  ← 글로벌 상태 + 상태 전이 통제
                    └──────┬───────┘
                           │ owns
              ┌────────────▼────────────┐
              │      StateMachine       │
              └─┬──────┬──────┬────────┘
                ▼      ▼      ▼
          MapState  CombatState  ShopState
                │      ↕          │
                ▼      ▼          ▼
          RestState  GameOverState
                        ▼
                   VictoryState
```

- `IGameState` 인터페이스: `Enter()`, `Update()`, `Exit()` 세 메서드만 정의
- `GameManager`가 현재 활성 상태를 가지고, 전이 시 `Exit()` → 교체 → `Enter()` 호출
- UI도 상태별로 활성화/비활성화 (InvasionGaugeUI는 항상 활성)

### ScriptableObject 구조

```
ScriptableObjects/
├── Enemies/          # 적 종류별 스탯 (HP, Attack, Gold, XP)
├── Items/            # 포션, 장비 (효과, 가격)
├── Skills/           # 스킬 정의 (MP 비용, 데미지, 효과)
└── Map/
    └── MapConfig     # 맵 생성 전체 설정 (Depth, 노드 수, 가중치 테이블)
```

### 맵 생성 (Layer-based Procedural)

```
레이어 구성:
  Layer 0     = [Start]           ← 고정
  Layer 1..N  = 랜덤 노드 1~3개  ← 타입은 가중치로 결정
  Layer N+1   = [Boss]            ← 고정

엣지 연결 규칙:
  1. nextLayer 모든 노드에 최소 1개의 부모 보장
  2. currentLayer 모든 노드에 최소 1개의 자식 보장
  3. 추가 엣지를 확률적으로 생성 (분기 형성)
```

**노드 타입 가중치 테이블 (기본값)**

| 구간 | NormalCombat | EliteCombat | Shop | Rest |
|---|---|---|---|---|
| 초반 (Layer 1~3) | 50% | 10% | 25% | 15% |
| 중반 (Layer 4~6) | 35% | 20% | 25% | 20% |
| 후반 (Layer 7~9) | 25% | 35% | 20% | 20% |

### 전투 턴 흐름

```
CombatState.Enter()
  └─→ PlayerTurn (입력 대기: 공격 / 스킬 / 아이템 / 도망)
        └─→ 액션 실행
              ├─ 적 HP ≤ 0 → 보상 획득 → MapState로 복귀
              └─ 적 생존 → EnemyTurn
                    └─→ 적 공격 실행
                          ├─ 플레이어 HP ≤ 0 → GameOverState
                          └─ 플레이어 생존 → PlayerTurn으로 루프
```

---

## Folder Structure (Unity Project)

```
Assets/
├── Scripts/
│   ├── Core/
│   │   ├── GameManager.cs              # 글로벌 상태, Singleton
│   │   └── StateMachine/
│   │       ├── IGameState.cs           # Enter / Update / Exit
│   │       ├── MapState.cs
│   │       ├── CombatState.cs
│   │       ├── ShopState.cs
│   │       ├── RestState.cs
│   │       ├── GameOverState.cs
│   │       └── VictoryState.cs
│   ├── Map/
│   │   ├── MapNode.cs                  # 노드 데이터 클래스
│   │   ├── MapGenerator.cs             # 프로시저널 맵 생성
│   │   └── MapRenderer.cs             # 맵을 화면에 렌더링
│   ├── Combat/
│   │   ├── CombatManager.cs           # 턴 흐름 관리
│   │   ├── PlayerCharacter.cs         # HP, MP, 공격력, 인벤토리
│   │   └── Enemy.cs                   # 적 인스턴스 (EnemyData SO 참조)
│   ├── Shop/
│   │   └── ShopManager.cs             # 상품 목록, 구매 로직
│   ├── UI/
│   │   ├── MapUI.cs                   # 맵 화면
│   │   ├── CombatUI.cs               # 전투 화면 (커맨드 버튼, HP/MP 바)
│   │   ├── ShopUI.cs                  # 상점 화면
│   │   └── InvasionGaugeUI.cs         # 항상 표시, GameManager 구독
│   └── Data/
│       ├── EnemyData.cs               # ScriptableObject
│       ├── ItemData.cs                # ScriptableObject
│       ├── SkillData.cs               # ScriptableObject
│       └── MapConfig.cs               # ScriptableObject (맵 생성 설정)
├── ScriptableObjects/                 # SO 인스턴스 (Inspector 편집용)
│   ├── Enemies/
│   ├── Items/
│   ├── Skills/
│   └── Map/
├── Sprites/
└── Scenes/
    └── Main.unity                     # 단일 씨앤
```

---

## 개발 단계 (Phase)

### Phase 1 — Project Setup
**목표:** Unity 프로젝트 초기화 + 기본 구조 세팅

- [ ] Unity LTS 프로젝트 생성
- [ ] 위의 폴더 구조 생성
- [ ] ScriptableObject 클래스 정의: `EnemyData`, `ItemData`, `SkillData`, `MapConfig`
- [ ] `GameManager` skeleton (글로벌 상태 변수만, 아직 로직 없음)
- [ ] `Main.unity` 씨앤 생성

---

### Phase 2 — Vertical Slice (최소 플레이 가능한 루프)
**목표:** Start → 전투 1회 → Boss → Victory/GameOver까지 경험 가능한 루프

이 단계에서는 완벽한 구현보다 **루프가 돌아가는 것**이 핵심입니다. UI는 텍스트 기반이어도 됩니다.

- [ ] `IGameState` 인터페이스 정의
- [ ] `StateMachine` 기본 구현 (활성 상태 교체, Enter/Exit 호출)
- [ ] `MapState` — 하드코딩된 선형 맵 (Start → 전투 → 보스, 3노드)
- [ ] `CombatState` — 공격만 가능한 기본 전투 (플레이어 공격 → 적 공격 → 반복)
- [ ] `Enemy.cs` — EnemyData SO에서 스탯 읽기
- [ ] `PlayerCharacter.cs` — HP, 공격력 기본 구현
- [ ] Invasion Gauge: 맵 이동 시 +1, maxGauge 도달 시 GameOver
- [ ] `GameOverState`, `VictoryState` — 기본 종료 화면
- [ ] `InvasionGaugeUI` — 항상 표시

**Phase 2 완료 기준:** 게임을 실행하여 Start에서 출발 → 적과 싸우고 이기거나 지거나 → 보스까지 가거나 게이지로 패배하는 것까지 직접 확인 가능

---

### Phase 3 — Procedural Map
**목표:** 하드코딩된 맵을 프로시저널 생성으로 교체

- [ ] `MapNode` 클래스 (Layer, Index, Type, Next 목록)
- [ ] `MapConfig` ScriptableObject 완성 (Depth, 노드 수 범위, 가중치 테이블, 엣지 확률)
- [ ] `MapGenerator` — 레이어별 생성, 가중치로 타입 배정, 엣지 연결 규칙 적용
- [ ] `MapRenderer` — 레이어별 노드를 씨앤에 배치, 엣지를 선으로 연결
- [ ] `MapState` — 클릭 가능한 다음 노드 강조, 선택 시 이동 처리
- [ ] 테스트: 여러 번 실행하여 맵이 매번 다르게 생성되는지 확인

---

### Phase 4 — Full Combat
**목표:** 전투에 Skill, Item, Flee 추가

- [ ] `SkillData` ScriptableObject 완성 + Skill 시스템 (MP 관리)
- [ ] `ItemData` ScriptableObject 완성 + 인벤토리 기본 구현
- [ ] CombatState에 커맨드 추가: Skill, Item, Flee
- [ ] Flee: 전투 종료 + invasion gauge +1 페널티
- [ ] 적 AI: 기본은 항상 공격 (추후 확장 가능한 구조로)
- [ ] 전투 승리 시 보상: gold, XP 획득
- [ ] `CombatUI` — 커맨드 버튼, 플레이어/적 HP·MP 바

---

### Phase 5 — Shop & Rest
**목표:** 상점과 휴식 노드 기능 구현

- [ ] `ShopManager` — 상품 목록 생성 (ItemData SO에서), 구매 로직 (gold 차감)
- [ ] `ShopState` + `ShopUI`
- [ ] `RestState` — HP 회복량 정의 및 적용
- [ ] 인벤토리 시스템 완성 (아이템 추가/사용/표시)

---

### Phase 6 — UI & Visual Polish
**목표:** 전체 UI를 완성도 높이기

- [ ] `MapUI` — 노드 타입별 아이콘, 현재 위치 강조, 이동 가능 경로 표시
- [ ] `CombatUI` — 스킬/아이템 드롭다운, 적 스프라이트, ダメージ 연출
- [ ] `ShopUI` — 아이템 카드, 구매 확인, 소지금 표시
- [ ] `InvasionGaugeUI` — 게이지 바로 시각화
- [ ] 스프라이트 작업 및 배치

---

### Phase 7 — Balance & Polish
**목표:** 실제로 재미있는 게임이 되도록 튜닝

- [ ] `maxInvasionGauge` / `mapDepth` 균형 튜닝
- [ ] 적 스탯 (HP, Attack, Gold, XP) 튜닝
- [ ] 노드 타입 가중치 테이블 조정
- [ ] 엘리트 전투 보상 (유물 시스템은 미정)
- [ ] Edge case 검토: 모든 경로가 막힌 경우, 극단적인 맵 구성 등
- [ ] 전체 게임 플레イ 테스트 및 난이도 조정

---

## 핵심 균형 파라미터

이 값들은 모두 ScriptableObject 또는 Inspector에서 조정 가능하게 만들어야 합니다.

| 파라미터 | 초기값 | 영향 |
|---|---|---|
| `maxInvasionGauge` | 30 | 클을수록 시간 압박 적음 |
| `mapDepth` | 10 | 클을수록 경로가 길어짐. 최단 경로 이동 횟수와 동일 |
| `minNodesPerLayer` | 1 | 선택지 최소 폭 |
| `maxNodesPerLayer` | 3 | 선택지 최대 폭 |
| `extraEdgeProbability` | 0.3 | 분기의 밀도 |
| 초기 gold | 100 | 시작 자금 |

**균형 공식:**
```
남은 게이지 여유 = maxInvasionGauge - mapDepth
                 = 30 - 10 = 20

이 값이 클수록 우회와 도망의 여유가 생김.
너무 작으면 최단 경로만 강제됨.
```

---

## 미정 사항 (Future Decisions)

아래 항목은 Phase 4~7 진행 중에 결정하면 됩니다.

1. **엘리트 보상의 "유물" 시스템** — plan.md에 언급되었나 구체적 정의 없음. 패시브 효과 아이템으로 구현할지, 별도 카테고리로 할지 미정.
2. **적 AI** — Phase 4에서는 항상 공격만. 추후 방어/회복/특수 공격 등 확장 여부 미정.
3. **경험치 (XP) 시스템** — plan.md에서 언급된 개념이지만 레벨업 메커닉은 정의되지 않음.
4. **보스 전투의 특이점** — 일반 전투와 동일한 구조인지, 특별한 페이즈/패턴이 있는지 미정.
