# AzerothCore 연속 공격 및 위치 판정 분석

## 1. 문제 제기

틱 기반 시스템에서 연속 공격(예: 쌍수 무기, 질풍)의 첫 타격과 마지막 타격 사이에 타겟이 이동할 경우, 타겟의 변경된 위치가 후속 타격에 반영되지 않아 공격이 빗나가는 현상이 발생할 수 있습니다. 본 문서는 AzerothCore가 이러한 상황을 어떻게 처리하는지 분석합니다.

## 2. 분석 요약

결론부터 말하면, **AzerothCore는 이러한 문제를 겪지 않습니다.** 시스템은 모든 개별 타격(주무기, 보조무기, 추가 공격 등)이 발생하기 직전에 매번 타겟의 현재 위치를 기준으로 사거리와 시야(LOS)를 다시 확인합니다. 모든 공격 판정은 단일 맵 스레드 내에서 순차적으로 처리되므로, 공격 시퀀스 중간에 위치 정보가 누락될 가능성이 없습니다.

## 3. 핵심 로직 상세 분석

### 3.1. 스레딩 모델: 맵 기반 멀티스레딩

-   **`MapMgr::Update`**: 서버는 여러 개의 워커 스레드를 사용하여 활성화된 각 맵(`Map` 객체)의 업데이트를 병렬로 처리합니다.
-   **`Map::Update`**: 그러나, **하나의 맵 인스턴스 내의 모든 로직(모든 플레이어, NPC, 오브젝트의 `Update` 함수 호출)은 항상 단일 스레드에서 순차적으로 실행됩니다.**
-   **결론**: 이 구조는 특정 맵(예: 스톰윈드) 내에서 한 유닛의 공격 계산이 진행되는 동안, 다른 유닛의 이동 패킷이 동시에 처리되는 것을 원천적으로 방지합니다. 모든 이벤트는 해당 맵의 업데이트 틱 안에서 순서대로 처리됩니다.

### 3.2. 공격 시퀀스

공격의 전체 흐름은 `Unit.cpp`의 `_UpdateSpells` 함수와 `Update` 함수에서 관리됩니다.

#### 3.2.1. 주무기 및 보조무기 공격 (쌍수 무기)

주무기와 보조무기는 `_UpdateSpells` 함수 내에서 각각 독립적인 타이머(`m_attackTimer`)에 의해 관리됩니다.

```cpp
// Unit.cpp 내의 _UpdateSpells 함수 (간소화된 의사코드)
void Unit::_UpdateSpells(uint32 time_diff)
{
    // 주무기 공격 타이머 확인
    if (주무기 공격 타이머 <= 0)
    {
        if (대상이 있고 && IsWithinMeleeRange(대상)) // 1. 주무기 공격 직전 거리 재확인
        {
            AttackerStateUpdate(대상, 주무기);      // 2. 주무기 공격 실행 (피해 계산 및 적용)
            resetAttackTimer(주무기);
        }
    }

    // 보조무기 공격 타이머 확인
    if (보조무기 공격 타이머 <= 0)
    {
        if (대상이 있고 && IsWithinMeleeRange(대상)) // 3. 보조무기 공격 직전 거리 재확인
        {
            AttackerStateUpdate(대상, 보조무기);    // 4. 보조무기 공격 실행
            resetAttackTimer(보조무기);
        }
    }
}
```

-   각 무기의 공격 시점이 되면, `IsWithinMeleeRange` 함수를 호출하여 타겟의 **현재 위치**를 기준으로 거리를 다시 계산합니다.
-   거리가 유효할 경우에만 `AttackerStateUpdate` 함수를 호출하여 실제 공격(빗나감, 회피, 피해량 계산 등)을 처리합니다.
-   주무기 공격과 보조무기 공격은 별개의 이벤트이므로, 주무기 공격 후 타겟이 범위를 벗어나면 보조무기 공격은 발동하지 않습니다.

#### 3.2.2. 추가 공격 (질풍 등)

질풍(Windfury)과 같은 추가 공격(Extra Attack)은 즉시 실행되지 않고, 다음 `Unit::Update` 틱에서 처리되도록 예약됩니다.

```cpp
// Unit.cpp 내의 Update 함수 (간소화된 의사코드)
void Unit::Update(uint32 p_time)
{
    // ...
    // 추가 공격 대기열 확인
    if (추가 공격이 예약되어 있음)
    {
        while (처리할 추가 공격이 남아있음)
        {
            // ...
            if (대상이 있고 && IsWithinMeleeRange(대상)) // 1. 추가 공격 직전 거리 재확인
            {
                HandleProcExtraAttackFor(대상, 공격 횟수); // 2. 추가 공격 실행
            }
        }
        // ...
    }
    // ...
}
```

-   추가 공격이 발동하면, `extraAttacksTargets`라는 대기열에 타겟 정보와 공격 횟수가 저장됩니다.
-   다음 서버 틱의 `Unit::Update`에서 이 대기열을 확인하고, 실제 공격을 실행하기 직전에 **다시 `IsWithinMeleeRange`를 호출하여** 타겟의 최신 위치를 기준으로 거리를 검사합니다.
-   따라서, 주무기 공격으로 질풍이 발동된 후 타겟이 즉시 거리를 벗어나면, 다음 틱에서 처리될 질풍 공격은 거리 판정 실패로 취소됩니다.

### 3.3. 위치 확인 로직: `IsWithinMeleeRange`

이 함수는 단순한 거리 계산을 넘어 여러 요소를 복합적으로 고려합니다.

```cpp
// Unit.cpp
bool Unit::IsWithinMeleeRange(Unit const* obj, float dist) const
{
    // ... 3D 좌표를 이용한 거리 제곱 계산 ...
    float distsq = ...;

    // 최종 공격 가능 거리 계산
    float maxdist = dist + GetMeleeRange(obj);

    // Leeway 보너스: 공격자와 대상이 모두 플레이어이고 이동 중일 때 추가 거리 보너스
    if ((IsPlayer() || obj->IsPlayer()) && HasLeewayMovement() && obj->HasLeewayMovement())
        maxdist += LEEWAY_BONUS_RANGE;

    return distsq < maxdist * maxdist;
}

float Unit::GetMeleeRange(Unit const* target) const
{
    // (내 모델 크기 + 대상 모델 크기 + 기본 여유 거리)와 최소 사거리 중 큰 값
    float range = GetCombatReach() + target->GetCombatReach() + 4.0f / 3.0f;
    return std::max(range, NOMINAL_MELEE_RANGE); // NOMINAL_MELEE_RANGE = 5.0f
}
```

-   **3D 거리**: X, Y, Z 좌표를 모두 사용하여 정확한 3차원 거리를 계산합니다.
-   **모델 크기(`GetCombatReach`)**: 공격자와 대상의 캐릭터 모델 크기를 반영하여, 큰 유닛은 더 먼 거리에서 공격할 수 있습니다.
-   **기본 여유 거리**: 모든 근접 공격에 `4.0/3.0` 야드의 고정된 추가 거리가 부여됩니다.
-   **Leeway 보정**: 클라이언트-서버 간 지연 시간(lag)을 보상하기 위해, 양측이 모두 이동 중일 경우 공격 거리를 늘려주는 '리웨이' 시스템이 적용됩니다.

## 4. 최종 결론

AzerothCore의 전투 로직은 연속 공격의 각 타격이 독립적인 이벤트로 처리되도록 설계되었습니다. 모든 타격은 실행 직전, 해당 맵 스레드의 현재 틱에서 타겟의 최신 위치 정보를 바탕으로 정교한 거리 및 시야 검사를 다시 수행합니다. 이로 인해 연속 공격 도중 타겟이 이동하더라도 그 위치 변화가 즉시 다음 타격 판정에 반영되어, '틱'으로 인한 판정 누락 현상은 발생하지 않습니다.
