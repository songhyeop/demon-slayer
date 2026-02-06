# Build & Test Framework Guide

이 문서는 Demon Slayer 프로젝트의 빌드 및 테스트 전략을 정의합니다.

---

## 1. 핵심 철학

### 1.1 빌드 철학

- **재현 가능한 빌드**: 누가 언제 빌드해도 동일한 결과물이 나와야 한다
- **원클릭 빌드**: 빌드 과정에 수동 개입이 없어야 한다
- **빌드 전 검증**: 테스트 통과 없이 빌드 배포 금지
- **버전 추적**: 모든 빌드 결과물은 버전과 빌드 시점을 명확히 표시

### 1.2 테스트 철학

- **테스트 먼저**: 기능 구현 전에 테스트 케이스부터 작성 (TDD)
- **빠른 피드백**: EditMode 테스트로 즉각적인 피드백 확보
- **Critical Path 우선**: 핵심 게임 루프를 먼저 테스트
- **모듈화된 코드**: 테스트 가능한 코드 = 좋은 설계

---

## 2. 테스트 전략

### 2.1 테스트 피라미드

```
          /\
         /  \        E2E Tests (PlayMode)
        /    \       - 전체 게임 플로우
       /------\      - 실제 Unity 런타임 필요
      /        \     - 느림, 최소한만 작성
     /          \
    /  통합 테스트  \   Integration Tests
   /              \  - 여러 시스템 연동 검증
  /----------------\
 /                  \
/    단위 테스트      \  Unit Tests (EditMode)
/--------------------\  - 개별 클래스/함수
                        - 빠름, 많이 작성
```

### 2.2 테스트 유형별 특징

| 유형 | 속도 | 범위 | 사용 시점 |
|------|------|------|-----------|
| **EditMode (Unit)** | 빠름 (ms) | 단일 클래스/함수 | 로직 검증 |
| **PlayMode (Integration)** | 느림 (초) | 여러 시스템 연동 | 게임 플로우 검증 |
| **Manual QA** | 매우 느림 | 전체 게임 | 릴리스 전 최종 확인 |

### 2.3 테스트 대상 우선순위

1. **필수 (Critical)**
   - 전투 시스템: 데미지 계산, 턴 순서, HP/MP 관리
   - 맵 생성기: DAG 구조 검증, 경로 유효성
   - 침공 게이지: 증가 조건, 게임오버 트리거
   - StateMachine: 상태 전환 로직

2. **권장 (Important)**
   - 상점 시스템: 구매/판매 로직
   - 인벤토리: 아이템 추가/사용
   - 경험치/레벨업 계산

3. **선택 (Optional)**
   - UI 상호작용
   - 사운드/이펙트 트리거

---

## 3. 테스트 작성 방법

### 3.1 AAA 패턴 (Arrange-Act-Assert)

```csharp
[Test]
public void PlayerAttack_ReducesEnemyHP()
{
    // Arrange (준비): 테스트에 필요한 객체 생성
    var player = new PlayerCharacter { AttackPower = 15 };
    var enemy = new Enemy { HP = 100 };

    // Act (실행): 테스트할 동작 수행
    player.Attack(enemy);

    // Assert (검증): 예상 결과 확인
    Assert.AreEqual(85, enemy.HP);
}
```

### 3.2 테스트 네이밍 규칙

```
[테스트대상]_[시나리오]_[예상결과]
```

예시:
- `PlayerAttack_WhenEnemyHasArmor_DamageIsReduced`
- `MapGenerator_WithDepth10_Creates12Layers`
- `InvasionGauge_ReachesMax_TriggersGameOver`
- `Flee_DuringCombat_IncreasesGaugeBy1`

### 3.3 EditMode 테스트 예시

```csharp
// Assets/Scripts/Tests/EditMode/Combat/CombatManagerTests.cs
using NUnit.Framework;
using DemonSlayer.Combat;

[TestFixture]
public class CombatManagerTests
{
    private CombatManager _combat;
    private PlayerCharacter _player;
    private Enemy _enemy;

    [SetUp]
    public void SetUp()
    {
        _player = new PlayerCharacter { HP = 100, AttackPower = 10 };
        _enemy = new Enemy { HP = 50, AttackPower = 5 };
        _combat = new CombatManager(_player, _enemy);
    }

    [Test]
    public void TurnOrder_PlayerActsFirst()
    {
        Assert.AreEqual(TurnOwner.Player, _combat.CurrentTurn);
    }

    [Test]
    public void ProcessPlayerTurn_Attack_DamagesEnemy()
    {
        _combat.ProcessPlayerTurn(PlayerAction.Attack);

        Assert.AreEqual(40, _enemy.HP);
    }

    [Test]
    public void ProcessPlayerTurn_AfterAction_EnemyTurnStarts()
    {
        _combat.ProcessPlayerTurn(PlayerAction.Attack);

        Assert.AreEqual(TurnOwner.Enemy, _combat.CurrentTurn);
    }

    [Test]
    public void EnemyDefeated_CombatEnds_WithVictory()
    {
        _enemy.HP = 5;

        _combat.ProcessPlayerTurn(PlayerAction.Attack);

        Assert.IsTrue(_combat.IsFinished);
        Assert.AreEqual(CombatResult.Victory, _combat.Result);
    }
}
```

### 3.4 PlayMode 테스트 예시

```csharp
// Assets/Scripts/Tests/PlayMode/Integration/FullGameFlowTests.cs
using System.Collections;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;
using UnityEngine.SceneManagement;

[TestFixture]
public class FullGameFlowTests
{
    [UnitySetUp]
    public IEnumerator SetUp()
    {
        yield return SceneManager.LoadSceneAsync("Main");
    }

    [UnityTest]
    public IEnumerator GameFlow_StartToFirstCombat_TransitionsCorrectly()
    {
        // Arrange
        var gameManager = Object.FindFirstObjectByType<GameManager>();
        yield return null;

        // Act: 전투 노드 선택
        var mapUI = Object.FindFirstObjectByType<MapUI>();
        mapUI.SimulateNodeClick(NodeType.NormalCombat);
        yield return new WaitForSeconds(0.5f);

        // Assert
        Assert.IsInstanceOf<CombatState>(gameManager.CurrentState);
    }

    [UnityTest]
    public IEnumerator InvasionGauge_ReachesMax_TriggersGameOver()
    {
        var gameManager = Object.FindFirstObjectByType<GameManager>();

        // 침공 게이지를 최대치까지 증가
        for (int i = 0; i < 30; i++)
        {
            gameManager.IncrementInvasionGauge();
            yield return null;
        }

        Assert.IsTrue(gameManager.IsGameOver);
        Assert.IsInstanceOf<GameOverState>(gameManager.CurrentState);
    }
}
```

### 3.5 맵 생성 테스트 (중요)

프로시저럴 맵 생성은 버그가 발생하기 쉬우므로 철저히 테스트해야 합니다.

```csharp
[TestFixture]
public class MapGeneratorTests
{
    [Test]
    public void Generate_WithDepth10_Creates12Layers()
    {
        var config = ScriptableObject.CreateInstance<MapConfig>();
        config.mapDepth = 10;

        var map = MapGenerator.Generate(config);

        // Start(1) + Middle(10) + Boss(1) = 12
        Assert.AreEqual(12, map.Layers.Count);
    }

    [Test]
    public void Generate_FirstLayer_IsStartNode()
    {
        var map = MapGenerator.Generate(CreateDefaultConfig());

        Assert.AreEqual(1, map.Layers[0].Nodes.Count);
        Assert.AreEqual(NodeType.Start, map.Layers[0].Nodes[0].Type);
    }

    [Test]
    public void Generate_LastLayer_IsBossNode()
    {
        var map = MapGenerator.Generate(CreateDefaultConfig());

        var lastLayer = map.Layers[^1];
        Assert.AreEqual(1, lastLayer.Nodes.Count);
        Assert.AreEqual(NodeType.Boss, lastLayer.Nodes[0].Type);
    }

    [Test]
    public void Generate_AllMiddleNodes_HaveAtLeastOneParent()
    {
        var map = MapGenerator.Generate(CreateDefaultConfig());

        for (int i = 1; i < map.Layers.Count; i++)
        {
            foreach (var node in map.Layers[i].Nodes)
            {
                Assert.IsTrue(node.Parents.Count > 0,
                    $"Node at layer {i} has no parent");
            }
        }
    }

    [Test]
    public void Generate_AllMiddleNodes_HaveAtLeastOneChild()
    {
        var map = MapGenerator.Generate(CreateDefaultConfig());

        for (int i = 0; i < map.Layers.Count - 1; i++)
        {
            foreach (var node in map.Layers[i].Nodes)
            {
                Assert.IsTrue(node.Children.Count > 0,
                    $"Node at layer {i} has no children");
            }
        }
    }

    [Test]
    public void Generate_HasValidPathFromStartToBoss()
    {
        var map = MapGenerator.Generate(CreateDefaultConfig());

        Assert.IsTrue(map.HasValidPath());
    }
}
```

---

## 4. 프로젝트 구조

### 4.1 Assembly Definition 구조

```
Assets/
├── Scripts/
│   ├── Runtime/
│   │   └── DemonSlayer.Runtime.asmdef     # 게임 코드
│   ├── Tests/
│   │   ├── EditMode/
│   │   │   └── DemonSlayer.Tests.EditMode.asmdef
│   │   └── PlayMode/
│   │       └── DemonSlayer.Tests.PlayMode.asmdef
│   └── Editor/
│       └── DemonSlayer.Editor.asmdef      # 에디터 도구
```

### 4.2 테스트 폴더 구조

```
Assets/Scripts/Tests/
├── EditMode/
│   ├── DemonSlayer.Tests.EditMode.asmdef
│   ├── Combat/
│   │   ├── CombatManagerTests.cs
│   │   ├── PlayerCharacterTests.cs
│   │   └── EnemyTests.cs
│   ├── Map/
│   │   ├── MapGeneratorTests.cs
│   │   ├── MapNodeTests.cs
│   │   └── WeightTableTests.cs
│   ├── Core/
│   │   ├── StateMachineTests.cs
│   │   └── GameManagerTests.cs
│   └── Data/
│       └── ScriptableObjectValidationTests.cs
│
└── PlayMode/
    ├── DemonSlayer.Tests.PlayMode.asmdef
    └── Integration/
        ├── FullGameFlowTests.cs
        ├── CombatFlowTests.cs
        └── MapNavigationTests.cs
```

### 4.3 Assembly Definition 설정 예시

**DemonSlayer.Tests.EditMode.asmdef**
```json
{
    "name": "DemonSlayer.Tests.EditMode",
    "rootNamespace": "DemonSlayer.Tests.EditMode",
    "references": [
        "DemonSlayer.Runtime"
    ],
    "includePlatforms": [
        "Editor"
    ],
    "excludePlatforms": [],
    "defineConstraints": [
        "UNITY_INCLUDE_TESTS"
    ],
    "optionalUnityReferences": [
        "TestAssemblies"
    ]
}
```

**DemonSlayer.Tests.PlayMode.asmdef**
```json
{
    "name": "DemonSlayer.Tests.PlayMode",
    "rootNamespace": "DemonSlayer.Tests.PlayMode",
    "references": [
        "DemonSlayer.Runtime",
        "DemonSlayer.Editor"
    ],
    "includePlatforms": [],
    "excludePlatforms": [],
    "defineConstraints": [
        "UNITY_INCLUDE_TESTS"
    ],
    "optionalUnityReferences": [
        "TestAssemblies"
    ]
}
```

---

## 5. 빌드 자동화

### 5.1 빌드 스크립트

```csharp
// Assets/Scripts/Editor/BuildTools/BuildScript.cs
using UnityEditor;
using UnityEditor.Build.Reporting;
using UnityEngine;
using System;
using System.IO;

public static class BuildScript
{
    private static readonly string[] Scenes = { "Assets/Scenes/Main.unity" };
    private static readonly string BuildFolder = "Builds";

    [MenuItem("Build/Build Windows (Development)")]
    public static void BuildWindowsDev()
    {
        Build(BuildTarget.StandaloneWindows64, BuildOptions.Development);
    }

    [MenuItem("Build/Build Windows (Release)")]
    public static void BuildWindowsRelease()
    {
        Build(BuildTarget.StandaloneWindows64, BuildOptions.None);
    }

    [MenuItem("Build/Build macOS")]
    public static void BuildMac()
    {
        Build(BuildTarget.StandaloneOSX, BuildOptions.None);
    }

    [MenuItem("Build/Build All Platforms")]
    public static void BuildAll()
    {
        BuildWindowsRelease();
        BuildMac();
    }

    private static void Build(BuildTarget target, BuildOptions options)
    {
        string version = Application.version;
        string timestamp = DateTime.Now.ToString("yyyyMMdd_HHmm");
        string targetName = target.ToString();
        string extension = target == BuildTarget.StandaloneWindows64 ? ".exe" : ".app";

        string buildPath = Path.Combine(
            BuildFolder,
            targetName,
            $"DemonSlayer_{version}_{timestamp}{extension}"
        );

        // 빌드 폴더 생성
        Directory.CreateDirectory(Path.GetDirectoryName(buildPath));

        BuildPlayerOptions buildOptions = new BuildPlayerOptions
        {
            scenes = Scenes,
            locationPathName = buildPath,
            target = target,
            options = options
        };

        Debug.Log($"Building for {targetName}...");
        BuildReport report = BuildPipeline.BuildPlayer(buildOptions);

        if (report.summary.result == BuildResult.Succeeded)
        {
            Debug.Log($"Build succeeded: {buildPath}");
            Debug.Log($"Build size: {report.summary.totalSize / (1024 * 1024):F2} MB");
            Debug.Log($"Build time: {report.summary.totalTime.TotalSeconds:F1} seconds");
        }
        else
        {
            Debug.LogError($"Build failed with {report.summary.totalErrors} errors");
            foreach (var step in report.steps)
            {
                foreach (var message in step.messages)
                {
                    if (message.type == LogType.Error)
                        Debug.LogError(message.content);
                }
            }
        }
    }
}
```

### 5.2 빌드 전 테스트 검증

```csharp
// Assets/Scripts/Editor/BuildTools/PreBuildValidation.cs
using UnityEditor;
using UnityEditor.TestTools.TestRunner.Api;
using UnityEngine;

public static class PreBuildValidation
{
    [MenuItem("Build/Build with Test Validation")]
    public static void BuildWithTests()
    {
        Debug.Log("Running EditMode tests before build...");

        var testRunnerApi = ScriptableObject.CreateInstance<TestRunnerApi>();
        var filter = new Filter { testMode = TestMode.EditMode };

        testRunnerApi.RegisterCallbacks(new TestCallbacks());
        testRunnerApi.Execute(new ExecutionSettings(filter));
    }

    private class TestCallbacks : ICallbacks
    {
        public void RunStarted(ITestAdaptor testsToRun) { }

        public void RunFinished(ITestResultAdaptor result)
        {
            if (result.FailCount > 0)
            {
                Debug.LogError($"Tests failed: {result.FailCount} failures. Build aborted.");
            }
            else
            {
                Debug.Log($"All {result.PassCount} tests passed. Proceeding with build...");
                BuildScript.BuildWindowsRelease();
            }
        }

        public void TestStarted(ITestAdaptor test) { }
        public void TestFinished(ITestResultAdaptor result) { }
    }
}
```

---

## 6. CI/CD Pipeline (선택)

### 6.1 GitHub Actions 워크플로우

```yaml
# .github/workflows/unity-ci.yml
name: Unity CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  UNITY_VERSION: 6000.3.6f1

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          lfs: true

      - name: Cache Library
        uses: actions/cache@v3
        with:
          path: Library
          key: Library-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-

      - name: Run EditMode Tests
        uses: game-ci/unity-test-runner@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          unityVersion: ${{ env.UNITY_VERSION }}
          testMode: EditMode
          githubToken: ${{ secrets.GITHUB_TOKEN }}

      - name: Run PlayMode Tests
        uses: game-ci/unity-test-runner@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          unityVersion: ${{ env.UNITY_VERSION }}
          testMode: PlayMode
          githubToken: ${{ secrets.GITHUB_TOKEN }}

      - name: Upload Test Results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: artifacts/

  build:
    name: Build ${{ matrix.targetPlatform }}
    needs: test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        targetPlatform:
          - StandaloneWindows64
          - StandaloneOSX
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          lfs: true

      - name: Cache Library
        uses: actions/cache@v3
        with:
          path: Library
          key: Library-${{ matrix.targetPlatform }}-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-${{ matrix.targetPlatform }}-

      - name: Build
        uses: game-ci/unity-builder@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
        with:
          unityVersion: ${{ env.UNITY_VERSION }}
          targetPlatform: ${{ matrix.targetPlatform }}
          versioning: Semantic

      - name: Upload Build
        uses: actions/upload-artifact@v3
        with:
          name: Build-${{ matrix.targetPlatform }}
          path: build/${{ matrix.targetPlatform }}
```

### 6.2 Unity License 설정

1. Unity 계정으로 라이센스 활성화
2. GitHub Secrets에 `UNITY_LICENSE` 추가
3. [GameCI 문서](https://game.ci/docs/github/activation) 참조

---

## 7. 테스트 커버리지 목표

| 시스템 | EditMode | PlayMode | 비고 |
|--------|----------|----------|------|
| StateMachine | 90% | 10% | 상태 전환 로직 필수 |
| CombatManager | 80% | 20% | 턴 흐름, 데미지 계산 |
| MapGenerator | 90% | 10% | 프로시저럴 생성 철저히 |
| PlayerCharacter | 80% | - | HP, MP, 레벨업 |
| InvasionGauge | 90% | 10% | 핵심 메카닉 |
| Shop | 70% | - | 구매/판매 로직 |
| UI | 10% | 40% | 통합 테스트 위주 |

---

## 8. 체크리스트

### 8.1 프로젝트 초기 설정

- [ ] Assembly Definition 파일 생성
  - [ ] `DemonSlayer.Runtime.asmdef`
  - [ ] `DemonSlayer.Editor.asmdef`
  - [ ] `DemonSlayer.Tests.EditMode.asmdef`
  - [ ] `DemonSlayer.Tests.PlayMode.asmdef`
- [ ] Test Framework 패키지 확인
- [ ] 첫 테스트 작성 및 실행 확인
- [ ] 빌드 스크립트 작성
- [ ] `.gitignore`에 `Builds/` 폴더 추가

### 8.2 Phase별 테스트 추가

**Phase 2 (Vertical Slice) 완료 시:**
- [ ] `StateMachineTests.cs`
- [ ] `CombatManagerTests.cs`
- [ ] `FullGameFlowTests.cs` (E2E)

**Phase 3 (Procedural Map) 완료 시:**
- [ ] `MapGeneratorTests.cs`
- [ ] `MapNodeTests.cs`
- [ ] `MapNavigationTests.cs`

**Phase 4 (Full Combat) 완료 시:**
- [ ] `SkillSystemTests.cs`
- [ ] `InventoryTests.cs`
- [ ] `CombatFlowTests.cs`

---

## 9. 참고 자료

- [Unity Test Framework 공식 문서](https://docs.unity3d.com/Packages/com.unity.test-framework@1.4/manual/index.html)
- [GameCI GitHub Actions](https://game.ci/docs/github/getting-started)
- [Assembly Definitions 가이드](https://docs.unity3d.com/Manual/ScriptCompilationAssemblyDefinitionFiles.html)
- [NUnit 공식 문서](https://docs.nunit.org/)
