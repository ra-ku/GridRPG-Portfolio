# GridRPG

> Unity 기반 1차원 그리드 RPG 개인 프로젝트  
> 턴제 전투, 타운 탐색, NPC 상호작용, 데이터 기반 콘텐츠 구성을 구현하며 게임 클라이언트 구조를 학습하고 있습니다.

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 개발 형태 | 개인 프로젝트 |
| 개발 기간 | 2026.04 ~ 개발 중 |
| 엔진 | Unity 6 `6000.3.5f2` |
| 언어 | C# |
| 주요 기술 | Unity Input System, Cinemachine, Animancer, ScriptableObject, Google Sheets CSV |
| 담당 영역 | 클라이언트 구조 설계, 전투 및 타운 시스템, 씬 전환, NPC 상호작용, 데이터 연동, 카메라, UI 기반 구조 |

GridRPG는 타일 위에서 유닛의 위치와 방향을 활용해 행동하는 RPG 프로토타입입니다.  
전투에서는 플레이어와 적이 턴을 주고받고, 타운에서는 자유 이동과 상호작용을 통해 NPC 대화, 문/포탈 이동, 전투 진입 흐름을 확인할 수 있도록 개발하고 있습니다.

## 핵심 구현 기능

### 턴제 전투

- 플레이어 턴과 적 턴이 교대로 진행되는 전투 흐름
- 입력 1회당 1턴을 소비하는 규칙 기반 전투
- 타일 기반 이동, 방향 전환, 일반 공격, 밀치기 공격
- `MoveAction`, `SwitchStanceAction`은 즉시 실행하고, 공격형 액션은 큐에 예약 후 순차 실행
- 예약된 액션은 실행 시점의 위치와 방향을 기준으로 판정
- 적이 다음 행동과 남은 턴을 표시하는 의도 시스템
- `UnitHealth.OnDied` 이벤트 기반 사망 처리
- `UnitRegistry`는 유닛 등록/해제와 목록 관리만 담당하고, 전투 종료 판단은 `BattleManager`가 담당

### 타운 탐색 및 상호작용

- 실시간 좌우 이동이 가능한 자유 탐색 모드
- 가까운 타일에 정렬한 뒤 행동하는 타운 턴 모드
- 플레이어 행동 이후 중립 NPC의 의도를 순차 실행
- `PlayerInteractor`가 Trigger 범위 안의 `ITownInteractable` 대상을 감지하고 가장 가까운 대상을 선택
- `NpcDialogue`를 통한 NPC별 대화 실행
- `DoorSceneTransition`을 통한 상호작용 기반 문 이동
- `ScenePortal`을 통한 트리거 기반 자동 포탈 이동

### 데이터 기반 콘텐츠 구성

- Google Sheets 데이터를 CSV로 받아 ScriptableObject 또는 런타임 DB에 적용
- `TownMapDatabase`, `BattleMapDatabase`로 씬별 그리드 생성 데이터 관리
- `TownSpawnDatabase`, `BattleSpawnDatabase`로 플레이어, NPC, 적, 포탈, 오브젝트 스폰 관리
- `PrefabDatabase`의 키를 통해 스폰 데이터와 Unity 프리팹 연결
- `PortalDestinationDatabase`로 포탈/문 목적지 데이터를 스폰 데이터와 분리 관리
- `DialogueLines` 시트의 CSV를 `DialogueClip` / `DialogueLine`으로 파싱해 대화 DB에 등록

`PortalDestinationData` 시트 예시:

```csv
PortalId,MoveType,TargetSceneId,TargetTileIndex
BattlePortal,StartBattle,BattleProto,-1
Door_To_Home,SceneMove,Jjamba_Low_Home,3
Battle_Return_Portal,ReturnToTown,Jjamba_Low,29
```

### 씬 전환 구조

최근 씬 이동 로직을 `SceneTransitionManager` 중심으로 정리했습니다.

- `ScenePortal`: 플레이어가 트리거에 닿았을 때 자동 이동 요청
- `DoorSceneTransition`: 플레이어가 상호작용 키를 눌렀을 때 이동 요청
- `SceneTransitionManager`: `PortalId`로 목적지 DB를 조회하고 `MoveType`에 따라 이동 처리
- `GameFlowManager`: 전투 시작, 마을 복귀, 다음 타운 스폰 타일 기억, 실제 씬 이동 요청 관리
- `SceneManagerEx`: Unity 씬 로드와 씬 초기화 완료 알림 담당

```mermaid
flowchart TD
    A["TownSpawnData / BattleSpawnData"] --> B["ObjectId 주입"]
    B --> C["ScenePortal 또는 DoorSceneTransition"]
    C --> D["SceneTransitionManager"]
    D --> E["PortalDestinationDatabase 조회"]
    E --> F{ "MoveType" }
    F -->|SceneMove| G["GameFlowManager.MoveToScene"]
    F -->|StartBattle| H["GameFlowManager.StartBattle"]
    F -->|ReturnToTown| I["GameFlowManager.ReturnToTown"]
    G --> J["SceneManagerEx.LoadScene"]
    H --> J
    I --> J
```

이 구조로 자동 포탈, 상호작용 문, 전투 진입, 전투 종료 후 마을 복귀를 같은 데이터 흐름에서 처리합니다.  
전투 종료 후 복귀할 때는 `PortalDestinationData.TargetTileIndex`를 `GameFlowManager`가 임시 저장하고, 새 타운 씬에서 `TownSpawnManager`가 플레이어 스폰 타일을 덮어씁니다.

### 타운-전투 코어 루프

- 타운 포탈을 통해 전투 씬으로 진입
- 전투 씬 진입 시 `BattleMapDatabase` 기준으로 전투 그리드 생성
- 전투 씬 진입 시 `BattleSpawnDatabase` 기준으로 유닛과 오브젝트 스폰
- 전투 승리 후 즉시 씬을 이동하지 않고, 전투 씬 안에서 타운 조작 모드로 전환
- 복귀 포탈을 밟으면 이전 타운 씬으로 돌아가는 흐름 구현
- 전투 중에는 포탈이 동작하지 않도록 입력 모드와 전투 상태를 함께 검사

### 조건 기반 대화 시스템

`DialogueLines` 시트의 `ConditionType` 값을 통해 같은 `DialogueId`라도 상황에 따라 다른 대사를 선택하도록 구현했습니다.

| ConditionType | 의미 |
| --- | --- |
| `Always` | 조건 대사가 없을 때 사용하는 기본 대사 |
| `FirstTalk` | 해당 `DialogueId`를 처음 완료하기 전 출력 |
| `AfterFirstTalk` | 첫 대화를 끝낸 뒤 출력 |
| `AfterBattleVictory` | 마지막 전투 결과가 승리일 때 우선 출력 |

- 첫 대화 완료 기록은 씬 이동 후에도 유지되도록 `DialogueManager`에 보관
- 씬 이동 시에는 현재 씬 UI 참조만 정리하고, 진행 기록은 유지
- NPC는 전투 결과를 직접 판단하지 않고 `DialogueManager`가 `GameFlowManager.LastBattleResult`를 조회

### 카메라 및 애니메이션

- 플레이어 추적 카메라와 전투 카메라 상태 전환
- 생성된 전투 그리드의 중앙 좌표를 기준으로 전투 카메라 LookAt 타겟을 런타임 생성
- 타운 씬에서 플레이어 스폰 직후 Cinemachine 플레이어 카메라를 타겟 위치로 즉시 스냅
- 씬 복귀 시 카메라가 먼 위치에서 플레이어를 따라오는 현상을 줄이도록 `PreviousStateIsValid`를 무효화하고 즉시 갱신
- 이동, 공격, 피격, 사망 애니메이션 흐름 연결
- 사망한 유닛이 Idle 애니메이션으로 돌아가지 않도록 처리

### 사운드 시스템 기반 구조

- 전역 `SoundManager` 추가
- `SoundDatabase` ScriptableObject로 사운드 ID, 타입, 클립, 볼륨, 반복 여부 관리
- BGM용 `AudioSource`와 SFX용 `AudioSource`를 런타임에 생성
- `PlayBgm(SoundId id)`, `PlaySfx(SoundId id)` 호출 구조 추가
- 같은 BGM을 다시 요청하면 재시작하지 않도록 현재 BGM ID를 검사

현재는 사운드 재생 구조를 추가한 상태이며, 실제 오디오 클립 연결과 씬/지역 단위 BGM 확장은 이후 작업으로 남아 있습니다.

## 클라이언트 구조

전역 서비스와 씬 전용 게임 흐름을 분리해, 각 클래스가 담당하는 책임을 명확하게 유지하려고 했습니다.

```mermaid
flowchart TD
    Managers["Managers<br/>전역 서비스 컨테이너"]
    Global["Input / Data / Camera / Sound / Grid / UnitRegistry / Dialogue"]
    Flow["GameFlowManager<br/>게임 흐름 상태 관리"]
    Transition["SceneTransitionManager<br/>포탈/문 이동 해석"]
    BattleContext["BattleContext<br/>전투 조립 지점"]
    TownContext["TownContext<br/>타운 조립 지점"]
    BattleManagers["BattleSpawnManager / TurnManager / BattleManager"]
    TownManagers["TownSpawnManager / TownTurnManager"]
    Interaction["PlayerInteractor / ITownInteractable / NpcDialogue / DoorSceneTransition"]

    Managers --> Global
    Managers --> Flow
    Managers --> Transition
    Global --> BattleContext
    Global --> TownContext
    Flow --> BattleContext
    Flow --> TownContext
    Transition --> Flow
    BattleContext --> BattleManagers
    TownContext --> TownManagers
    TownContext --> Interaction
```

- `Managers`는 입력, 데이터, 카메라, 사운드, 대화처럼 씬이 바뀌어도 유지되는 서비스를 관리합니다.
- `GameFlowManager`는 타운에서 전투로 진입하고, 전투 종료 결과와 돌아갈 타운을 기억하는 흐름을 담당합니다.
- `SceneTransitionManager`는 포탈/문 ID를 목적지 데이터와 연결하고, 실제 이동은 `GameFlowManager`에 요청합니다.
- `BattleContext`는 전투 씬에서만 필요한 매니저를 생성하고 연결합니다.
- `TownContext`는 타운 씬의 그리드 생성, 액터 스폰, 타운 턴 흐름을 관리합니다.
- `UnitRegistry`는 유닛 목록과 등록 상태만 관리하며, 승패 판단은 `BattleManager`가 담당합니다.

## 주요 문제 해결 경험

### 1. 전역 매니저에 집중되던 책임 분리

초기에는 전투 매니저 접근이 전역 `Managers`에 집중되어 타운 시스템을 추가할수록 의존성이 커질 가능성이 있었습니다.  
전투와 타운에 각각 `BattleContext`, `TownContext`를 도입하여 씬 전용 매니저의 생성과 생명주기를 분리했습니다.

**결과**

- 전투와 타운 코드가 서로의 매니저에 직접 의존하지 않게 됨
- 씬 종료 시 이벤트 구독 해제와 리소스 정리 위치가 명확해짐
- 새 씬 전용 시스템을 추가하기 쉬운 구조 확보

### 2. 유닛 목록 관리와 전투 종료 판단 분리

`UnitRegistry`가 전투 종료까지 판단하면 목록 관리와 게임 흐름이라는 두 책임이 섞이게 됩니다.  
`UnitRegistry`는 유닛 등록, 해제, 분류만 담당하고, 적 사망 후 승리 조건은 `BattleManager`가 판단하도록 구성했습니다.

```mermaid
sequenceDiagram
    participant Health as UnitHealth
    participant Registry as UnitRegistry
    participant Battle as BattleManager

    Health->>Registry: OnDied 이벤트
    Registry->>Registry: 죽은 유닛 목록에서 제거
    Registry->>Battle: OnUnitUnRegistered 이벤트
    Battle->>Battle: 남은 적 수 확인 및 전투 종료
```

### 3. 씬 이동 방식 분리와 중앙화

처음에는 전투 포탈처럼 특정 목적에 맞춘 스크립트가 씬 이동을 직접 처리했습니다.  
맵, 문, 포탈, 전투 복귀 흐름이 늘어나면서 이동 목적지 조회와 이동 실행을 `SceneTransitionManager`로 모았습니다.

**결과**

- 자동 포탈과 상호작용 문이 같은 목적지 데이터를 사용
- `BattlePortal`에 의존하지 않고 `ScenePortal` / `DoorSceneTransition`으로 이동 발동 방식만 분리
- `PortalDestinationData` 시트에서 `PortalId`, `MoveType`, `TargetSceneId`, `TargetTileIndex`를 수정해 이동 흐름 변경 가능
- 전투 복귀 시 목표 타일 스폰을 데이터로 지정 가능

### 4. 콘텐츠를 코드 수정 없이 배치

타운과 전투의 타일 수, 스폰 위치, 방향을 코드에 직접 작성하면 콘텐츠 수정 비용이 커집니다.  
Google Sheets의 맵 및 스폰 데이터를 읽어 그리드와 액터를 생성하도록 구성했습니다.

**결과**

- 타일 수, 스폰 위치, 방향을 시트에서 수정 가능
- 프리팹 키를 통해 데이터와 Unity 에셋 연결
- 타운과 전투 추가 시 코드 변경 범위 감소

### 5. 조건 기반 대화 상태 유지

NPC 첫 대화 여부를 씬 오브젝트에 저장하면 씬 이동 시 상태가 초기화될 수 있습니다.  
`DialogueManager`가 완료된 `DialogueId`를 전역 매니저 생명주기 동안 보관하고, 씬 이동 시에는 UI 참조만 정리하도록 수정했습니다.

**결과**

- 씬 이동 후에도 `FirstTalk`가 반복 출력되지 않음
- 전투 승리 후 `AfterBattleVictory` 대사로 전환 가능
- NPC는 전투 결과나 진행 상태를 직접 알 필요 없이 대화 요청만 담당

### 6. 죽은 유닛의 타일 점유 해제

유닛이 사망해도 타일 점유 정보가 남아 있으면 `MoveAction`이 해당 타일을 `Occupied`로 판단해 이동할 수 없습니다.  
사망 이벤트로 `UnitRegistry`에서 유닛을 등록 해제할 때 현재 타일 참조를 정리하고, 죽은 유닛은 이동 차단 대상으로 계산하지 않도록 변경했습니다.

**결과**

- 죽은 유닛이 있던 타일이 이동 가능한 상태로 전환
- 사망 애니메이션을 위해 오브젝트가 잠시 남아 있어도 이동 판정에는 영향을 주지 않음

## 현재 개발 상태

| 상태 | 기능 |
| --- | --- |
| 구현 완료 | 전투 그리드, 유닛 스폰, 플레이어 및 적 턴, 이동, 방향 전환, 일반 공격, 밀치기 공격 |
| 구현 완료 | 전투 카메라 전환, 적 의도, 유닛 사망, 적 전멸 승리 조건 |
| 구현 완료 | 타운 데이터 로드, 타운 그리드 및 액터 스폰, 자유 이동, 타운 턴, NPC 의도 |
| 구현 완료 | NPC Trigger 상호작용, 말풍선 대화 UI, 카메라 빌보드, 대화 중 이동 제어 |
| 구현 완료 | 조건 기반 대화 선택: `FirstTalk`, `AfterFirstTalk`, `AfterBattleVictory`, `Always` |
| 구현 완료 | `SceneTransitionManager` 기반 포탈/문/전투 진입/전투 복귀 흐름 |
| 구현 완료 | `PortalDestinationDatabase` 기반 이동 목적지 데이터 분리 |
| 구현 완료 | `BattleSpawnDatabase` 기반 전투 유닛 및 복귀 포탈 스폰 |
| 구현 완료 | `BattleMapDatabase` 기반 전투 그리드 생성 |
| 구현 완료 | `SoundManager`와 `SoundDatabase` 기반 사운드 재생 구조 |
| 구현 완료 | 타운 플레이어 스폰 직후 Cinemachine 카메라 스냅 처리 |
| 개발 중 | 찌르기 공격, 선택지 UI, 전투 결과 보상 흐름 |
| 개선 예정 | 실제 사운드 리소스 연결, 페이드 인/아웃 기반 씬 전환, 플레이 영상과 스크린샷, 핵심 코드 샘플, 자동화 테스트, 빌드 설정 정리 |

## 개발 방향

- 전투 승리 후 보상, 결과 UI, 다음 목표 안내 흐름 추가
- 액션 종류와 적 행동 패턴 확장
- 대화 선택지 UI와 상호작용 대상 종류 확장
- 실제 BGM/SFX 리소스를 `SoundDatabase`에 연결하고 전투 액션에 SFX 적용
- 페이드 인/아웃 또는 로딩 화면을 통한 씬 전환 시각 보완
- 핵심 매니저와 턴 흐름에 대한 테스트 추가
- 플레이 영상과 구조도를 활용한 포트폴리오 문서 개선

## 저장소 공개 범위

실제 Unity 프로젝트 저장소는 유료 에셋과 전체 프로젝트 리소스를 포함하고 있어 비공개로 관리하고 있습니다.  
이 저장소에는 직접 구현한 클라이언트 시스템의 설계와 개발 과정을 중심으로 정리하며, 공개 가능한 코드 샘플과 플레이 자료를 순차적으로 추가할 예정입니다.
