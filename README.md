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
| 담당 영역 | 클라이언트 구조 설계, 전투 및 타운 시스템, NPC 상호작용, 데이터 연동, 카메라, UI 기반 구조 |

GridRPG는 타일 위에서 유닛의 위치와 방향을 활용해 행동하는 RPG 프로토타입입니다.  
전투에서는 플레이어와 적이 턴을 주고받고, 타운에서는 자유 이동과 턴 모드를 전환하여 NPC 행동 및 상호작용을 확인할 수 있도록 개발하고 있습니다.

## 핵심 구현 기능

### 턴제 전투

- 플레이어 턴과 적 턴이 교대로 진행되는 전투 흐름
- 타일 기반 이동, 방향 전환, 일반 공격, 밀치기 공격
- 여러 액션을 예약하고 순차 실행하는 액션 큐
- 적이 다음 행동과 남은 턴을 표시하는 의도 시스템
- 유닛 사망 이벤트와 모든 적 처치 시 전투 종료 처리

### 타운 탐색 및 타운 턴

- 실시간 좌우 이동이 가능한 자유 탐색 모드
- 가까운 타일에 정렬한 뒤 행동하는 타운 턴 모드
- 플레이어 행동 이후 중립 NPC의 의도를 순차 실행
- NPC의 좌, 우, 정면, 후면 방향 상태 지원
- 중립 NPC와 플레이어가 타일을 공유할 수 있는 점유 규칙

### NPC 대화 및 상호작용

- 타운 모드 전용 `TownInteract` 입력을 추가하고 `E` 키로 상호작용 처리
- `PlayerInteractor`가 Trigger 범위 안의 `ITownInteractable` 대상을 감지하고, 가장 가까운 대상을 선택
- `NpcDialogue`가 NPC별 `DialogueId`와 말풍선 UI를 연결
- NPC 하위에 배치하는 `DialogueBubbleUI`로 월드 스페이스 말풍선 표시
- `BillboardToCamera`를 통해 말풍선이 카메라를 바라보도록 처리
- 대화 중 플레이어 이동을 잠그고, 대화 중 `E` 입력은 다음 대사 진행으로 사용
- 화면형 `DialogueUI`와 NPC 말풍선형 `DialogueBubbleUI`를 `IDialogueView` 인터페이스로 분리

### 데이터 기반 콘텐츠 구성

- Google Sheets 데이터를 CSV로 받아 ScriptableObject 또는 런타임 DB에 적용
- 타운별 타일 개수, 간격, 시작 위치를 데이터로 관리
- 프리팹 키와 타일 인덱스를 이용한 플레이어 및 NPC 스폰
- 스폰 데이터에서 유닛 방향과 위치 오프셋 설정
- 전투별 유닛과 복귀 포털 스폰 정보를 `BattleSpawnDatabase`로 관리
- `DialogueLines` 시트의 CSV를 `DialogueClip` / `DialogueLine`으로 파싱해 `DialogueDataBase`에 등록

### 타운-전투 코어 루프

- 타운 포털을 통해 전투 씬으로 진입
- 전투 승리 후 즉시 씬을 이동하지 않고, 전투 씬 안에서 타운 조작 모드로 전환
- 복귀 포털을 밟으면 이전 타운 씬으로 돌아가는 기본 흐름 구현
- 전투 중에는 포털이 작동하지 않도록 입력 모드와 전투 상태를 함께 검사

### 카메라 및 애니메이션

- 플레이어 추적 카메라와 전투 카메라 상태 전환
- 생성된 전투 그리드의 중앙 좌표를 기준으로 LookAt 타겟을 런타임 생성
- 이동, 공격, 피격, 사망 애니메이션 흐름 연결
- 사망한 유닛이 Idle 애니메이션으로 돌아가지 않도록 처리

## 클라이언트 구조

전역 서비스와 씬 전용 게임 흐름을 분리해, 각 클래스가 담당하는 책임을 명확하게 유지하려고 했습니다.

```mermaid
flowchart TD
    Managers["Managers<br/>전역 서비스 컨테이너"]
    Global["Input / Data / Camera / Grid / UnitRegistry / Dialogue"]
    Flow["GameFlowManager<br/>씬 흐름 관리"]
    BattleContext["BattleContext<br/>전투 조립 지점"]
    TownContext["TownContext<br/>타운 조립 지점"]
    BattleManagers["BattleSpawnManager / TurnManager / BattleManager"]
    TownManagers["TownSpawnManager / TownTurnManager"]
    Interaction["PlayerInteractor / ITownInteractable / NpcDialogue"]

    Managers --> Global
    Managers --> Flow
    Global --> BattleContext
    Global --> TownContext
    Flow --> BattleContext
    Flow --> TownContext
    BattleContext --> BattleManagers
    TownContext --> TownManagers
    TownContext --> Interaction
```

- `Managers`는 입력, 데이터, 카메라, 대화처럼 씬이 바뀌어도 유지되는 서비스를 관리합니다.
- `GameFlowManager`는 타운에서 전투로 진입하고, 전투 종료 후 돌아갈 타운을 기억하는 흐름을 담당합니다.
- `BattleContext`는 전투씬에서만 필요한 매니저를 생성하고 연결합니다.
- `TownContext`는 타운씬의 그리드 생성, 액터 스폰, 타운 턴 흐름을 관리합니다.
- `UnitRegistry`는 유닛 목록과 등록 상태만 관리하며, 승패 판단은 `BattleManager`가 담당합니다.
- `PlayerInteractor`는 타운 상호작용 입력과 대상 선택을 담당하고, 실제 대화 실행은 `NpcDialogue`가 담당합니다.

## 주요 문제 해결 경험

### 1. 전역 매니저에 집중되던 책임 분리

초기에는 전투 매니저 접근이 전역 `Managers`에 집중되어 타운 시스템을 추가할수록 의존성이 커질 가능성이 있었습니다.  
전투와 타운에 각각 `BattleContext`, `TownContext`를 도입하여 씬 전용 매니저의 생성과 생명주기를 분리했습니다.

**결과**

- 전투와 타운 코드가 서로의 매니저에 직접 의존하지 않게 됨
- 씬 종료 시 이벤트 구독 해제와 리소스 정리 위치가 명확해짐
- 새로운 씬 전용 시스템을 추가하기 쉬운 구조 확보

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

### 3. 씬 오브젝트 검색에 의존하지 않는 전투 카메라

전투 카메라가 이름 기반 오브젝트 검색에 의존하면 씬 구성 변경에 취약해집니다.  
`GridManager`가 생성한 첫 타일과 마지막 타일로 중앙 좌표를 계산하고, 카메라 LookAt 타겟을 런타임에 생성하도록 변경했습니다.

**결과**

- `GameObject.Find` 의존 제거
- 그리드 크기가 달라져도 전투 카메라가 중앙을 바라봄
- 씬에 별도 타겟 오브젝트를 수동 배치할 필요가 줄어듦

### 4. 타운 콘텐츠를 코드 수정 없이 배치

타운마다 플레이어와 NPC 위치를 코드에 직접 작성하면 콘텐츠 수정 비용이 커집니다.  
Google Sheets의 타운 맵 및 스폰 데이터를 읽어 그리드와 액터를 생성하도록 구성했습니다.

**결과**

- 타일 수, 스폰 위치, 방향을 시트에서 수정 가능
- 프리팹 키를 통해 데이터와 Unity 에셋 연결
- 타운 추가 시 코드 변경 범위 감소

### 5. 전투 진입과 복귀 흐름 분리

전투 승리 즉시 타운으로 씬을 전환하면 전투 종료 후 보상, 잔여 탐색, 복귀 선택 같은 흐름을 넣기 어렵습니다.  
`BattleManager`는 승리 조건과 전투 종료 상태를 판단하고, `GameFlowManager`가 씬 흐름을 관리하도록 분리했습니다.

**구현 내용**

- `BattlePortal`을 통해 타운에서 전투 씬으로 진입
- 전투 승리 시 `BattleState.BattleEnd`로 전환한 뒤 플레이어 조작을 타운 모드로 변경
- 전투 씬의 복귀 포털은 `InputMode.Town`이고 전투가 종료된 상태일 때만 작동
- `SceneManagerEx`를 일반 `IManager`로 정리하여 전역 매니저 컨테이너에서 안정적으로 등록

### 6. 죽은 유닛의 타일 점유 해제

유닛이 사망해도 타일 점유 정보가 남아 있으면 `MoveAction`이 해당 타일을 `Occupied`로 판단해 이동할 수 없습니다.  
사망 이벤트로 `UnitRegistry`에서 유닛을 등록 해제할 때 현재 타일 참조를 정리하고, 죽은 유닛은 이동 차단 대상으로 계산하지 않도록 변경했습니다.

**결과**

- 죽은 유닛이 있던 타일이 이동 가능한 상태로 전환
- `UnitRegistry`는 등록 해제와 이벤트 알림을 담당하고, 타일 참조 정리는 `Unit`이 수행
- 사망 애니메이션을 위해 오브젝트가 잠시 남아 있어도 이동 판정에는 영향을 주지 않음

### 7. NPC 대화 UI와 상호작용 책임 분리

NPC 대화 기능을 `PlayerTownController`에 직접 넣으면 이동, 입력, 대화 감지, UI 실행 책임이 한 클래스에 섞일 수 있습니다.  
`PlayerInteractor`, `ITownInteractable`, `NpcDialogue`, `DialogueBubbleUI`로 역할을 나누어 대화 상호작용을 구현했습니다.

**구현 내용**

- `PlayerInteractor`가 Trigger 범위 안의 상호작용 대상을 관리
- `NpcDialogue`가 NPC별 `DialogueId`와 말풍선 UI 실행을 담당
- `DialogueRunner`가 특정 UI 타입에 묶이지 않도록 `IDialogueView`를 사용
- `DialogueBubbleUI`는 NPC 하위 월드 스페이스 말풍선을 담당
- `BillboardToCamera`는 UI가 카메라를 바라보는 회전만 담당

**고려한 위험**

- NPC 몸 충돌용 Collider와 대화 감지용 Trigger Collider를 분리하지 않으면, 충돌 범위와 상호작용 범위를 독립적으로 조절하기 어렵습니다.
- 대화 중 전체 입력을 잠그면 다음 대사 진행 입력까지 막힐 수 있어, 현재는 플레이어 이동 컨트롤만 잠그고 `E` 입력은 다음 대사 진행으로 사용합니다.

## 현재 개발 상태

| 상태 | 기능 |
| --- | --- |
| 구현 완료 | 전투 그리드, 유닛 스폰, 플레이어 및 적 턴, 이동, 방향 전환, 일반 공격, 밀치기 공격 |
| 구현 완료 | 전투 카메라 전환, 적 의도, 유닛 사망, 적 전멸 승리 조건 |
| 구현 완료 | 타운 데이터 로드, 타운 그리드 및 액터 스폰, 자유 이동, 타운 턴, NPC 의도 |
| 구현 완료 | 타운 포털 전투 진입, 전투 승리 후 복귀 포털을 통한 타운 복귀 |
| 구현 완료 | `BattleSpawnDatabase` 기반 전투 유닛 및 포털 스폰 |
| 구현 완료 | NPC Trigger 상호작용, 말풍선 대화 UI, 카메라 빌보드, 대화 중 이동 제어 |
| 구현 완료 | `DialogueLines` CSV 파싱 및 `DialogueDataBase` 등록 코드 |
| 개발 중 | 찌르기 공격, 선택지 UI, 전투 결과 보상 흐름 |
| 개선 예정 | 플레이 영상과 스크린샷, 핵심 코드 샘플, 자동화 테스트, 빌드 설정 정리 |

## 개발 방향

- 전투 승리 후 보상, 결과 UI, 다음 목표 안내 흐름 추가
- 액션 종류와 적 행동 패턴 확장
- 대화 선택지 UI와 상호작용 대상 종류 확장
- 핵심 매니저와 턴 흐름에 대한 테스트 추가
- 플레이 영상과 구조도를 활용한 포트폴리오 문서 개선

## 저장소 공개 범위

실제 Unity 프로젝트 저장소는 유료 에셋과 전체 프로젝트 리소스를 포함하고 있어 비공개로 관리하고 있습니다.  
이 저장소에는 직접 구현한 클라이언트 시스템의 설계와 개발 과정을 중심으로 정리하며, 공개 가능한 코드 샘플과 플레이 자료를 순차적으로 추가할 예정입니다.
