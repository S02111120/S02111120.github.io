# [UE5/C++] 탑뷰 로그라이크 개발기: 인벤토리 컴포넌트 설계, Git 충돌 병합 및 보스 보상 드롭 파이프라인 구축

    > **개발 환경**: Unreal Engine 5, C++, Visual Studio 2022, Git
    > **협업 구성**: 3인 (UI 담당 1, 보스 AI/패턴 담당 1, 시스템/인벤토리 담당 본인)

    ---

    ## 1. 인벤토리 & 아이템 백엔드 시스템 설계

    UI 작업과의 병렬 개발을 위해, UI 의존성을 배제하고 독립적으로 구동되는 C++ 비즈니스 로직과 데이터 파이프라인을
  먼저 구축했다.

    ### 1-1. Data-Driven 아이템 설계 (`UPrimaryDataAsset`)

    하드코딩을 방지하고 에디터 친화적인 확장성을 위해 `URLItemDataAsset`을 구현했다.

    * `FPrimaryAssetId`를 활용한 에셋 관리
    * 아이템 ID, 이름, 설명, 인벤토리 텍스처, 최대 중첩 개수(`MaxStackCount`), 소모품/장비 타입 열거형(`EItemType`)
  정의
    * 1차 동작 검증용 테스트 데이터 에셋(`DA_Potion_Health`) 제작

    ```cpp
    // RLItemDataAsset.h
    UCLASS(BlueprintType)
    class ROGUELIKETOPVIEW_API URLItemDataAsset : public UPrimaryDataAsset
    {
        GENERATED_BODY()

    public:
        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item|Info")
        FName ItemID;

        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item|Info")
        FText DisplayName;

        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item|Display")
        TObjectPtr<UTexture2D> Icon;

        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item|Data")
        int32 MaxStack;
    };

  ### 1-2. 컴포넌트 기반 아키텍처 (URLInventoryComponent)

  플레이어 캐릭터(ARLCharacter)의 비대화(Fat Actor)를 방지하고 단일 책임 원칙(SRP)을 준수하기 위해 인벤토리 기능을
  UActorComponent로 격리했다.

  • 인터페이스 분리: AddItem(), UseItem(), RemoveItem() 등의 핵심 API 노출
  • 느슨한 결합(Loose Coupling): 상태 변화 시 호출되는 Dynamic Multicast Delegate를 선언하여, UI 위젯이 컴포넌트를
  직접 참조하지 않고 이벤트 구독 형태로 동작하도록 설계
  • 협업 문서화: UI 작업자가 C++ 코드를 분석하지 않고도 바인딩할 수 있도록 함수 시그니처와 델리게이트 이벤트 목록을
  UI_INVENTORY_GUIDE.md로 문서화하여 공유함
  ──────
  ## 2. Git Merge Conflict 분석 및 수동 병합

  ### 2-1. 충돌 원인 분석

  플레이어 캐릭터(ARLCharacter.h / RLCharacter.cpp)에 인벤토리 컴포넌트를 통합하는 과정에서, 보스 패턴 담당 팀원의
  공격 예고 데칼(Decal Indicator) 커밋이 원격 브랜치에 먼저 머지되며 코드 충돌이 발생함.
  ### 2-2. 해결 과정

  1. 헤더 병합 (RLCharacter.h):
      • 보스 공격 예고용 데칼 컴포넌트 선언과 인벤토리 컴포넌트 포인터 선언부를 각각 보존
      • 전방 선언(Forward Declaration) 중복 정리 및 인클루드 최소화
  2. 소스 병합 (RLCharacter.cpp):
      • BeginPlay() 내 초기화 루틴의 실행 우선순위를 정리
      • 컴포넌트 등록 및 델리게이트 바인딩 시점이 겹치지 않도록 호출 순서 재배열
  3. 로컬 빌드 검증:
      • 언리얼 엔진 핫리로드 대신 에디터 종료 후 Rider/VS 기반 클린 빌드 수행하여 이상 없음 확인


  │ Engineering Note:
  │ 공용 코어 클래스(Character)를 여러 작업자가 동시에 수정하면 충돌 비용이 급증한다. 핵심 액터는 서브시스템이나
  │ 컴포넌트의 컨테이너 역할만 수행하도록 제한하고, 기능 구현은 철저히 컴포넌트로 캡슐화해야 충돌을 최소화할 수 있음을
  │ 확인했다.
  ──────
  ## 3. 몬스터 사망 및 보상 드롭 파이프라인 리팩토링

  ### 3-1. 기존 문제점 (Issue)

  • 일반 몬스터 처치 시에도 보상 상자(MyRLRewardChest)가 100% 드롭되는 현상
  • 보스 몬스터(BP_Boss) 처치 시에는 정작 상자가 스폰되지 않는 로직 누락 발생

  ### 3-2. 원인 파악 (RCA)

  ARLEnemyCharacter::Die() 함수 내부에서 사망 액터의 타입을 식별하지 않고 일괄 처리되고 있었으며, 상자 스폰 로직이
  보스 전용 분기에 태워지지 않은 상태였음.
  ### 3-3. 해결 구현 (RLEnemyCharacter.cpp)

  • 보스 런타임 판별: 사망 시점의 액터가 보스(BP_Boss)인지 클래스/태그를 통해 자동 식별
  • 드롭 테이블 분기:
      • 일반 몬스터: 보상 상자 드롭 로직 제거, 필드 드롭 액터(ARLItemDrop) 확률 계산만 수행
      • 보스 몬스터: 보스 판정 시 위치 벡터를 계산하여 MyRLRewardChest를 100% 확정 스폰하도록 동적 로드 연동
    // RLEnemyCharacter.cpp
    void ARLEnemyCharacter::Die()
    {
        Super::Die();

        // 보스 여부 런타임 판정
        if (IsBoss())
        {
            // 보스 처치 시 보상 상자 100% 확정 스폰
            SpawnBossRewardChest();
        }
        else
        {
            // 일반 몬스터: 필드 아이템 드롭 확률 연산
            TryDropFieldItem();
        }
    }
  ──────
  ## 4. 의존성 관리 및 결함 방어 (Defensive Coding)

  • UI 미완성 브랜치에 대한 방어:
  보상 상자 인터랙션 시 호출될 삼지선다 UI(WBP_RewardSelect)가 타 작업자의 로컬 브랜치에 위치한 상태였음.
  에셋 로드 실패로 인한 에디터 크래시를 방지하기 위해 소프트 클래스 경로 검증 및 IsValid() 널 체크 방어 코드를 적용함.
  ──────
  ## 5. Next Steps

  [ ] 보스 전용 경험치 분기 처리: 현재 기본값(30 EXP)으로 고정되어 일반 몬스터와 동일한 EXP를 지급하는 상태. 보스 판정
  시 대량 경험치(350 EXP)를 즉시 부여하도록 C++ 분기 추가
  [ ] UI 병합 후 엔드투엔드 검증: WBP_RewardSelect 머지 후 [보스 처치 ➡️ 상자 오픈 ➡️ 삼지선다 선택 ➡️ 인벤토리
  반영]의 전체 파이프라인 루프 테스트
