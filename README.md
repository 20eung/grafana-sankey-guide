# Grafana Sankey 패널 구성 가이드: Network Traffic Flow

이 문서는 Grafana의 **Sankey Panel**을 사용하여 네트워크 장비의 트래픽 흐름(Source ➔ Target)을 시각화하는 방법을 정리합니다.
특히 **Nokia**(`ifDescr` 사용)와 **Arista/Cisco**(`ifAlias` 사용) 장비가 혼재된 환경에서 **벤더에 상관없이 인터페이스 설명을 표준화**하여 보여주는 기법을 포함합니다.

## 1. 사전 준비: Telegraf 설정 (ifAlias 수집)

Arista 등 일부 벤더는 인터페이스 설명(Description)을 `ifAlias` OID에 저장합니다. 따라서 Telegraf에서 이를 **Tag**로 수집해야 합니다.

**`telegraf.conf` 설정 예시:**

```toml
[[inputs.snmp.table]]
  name = "snmp"
  inherit_tags = [ "hostname" ]

  # [Tag] Nokia 등에서 사용
  [[inputs.snmp.table.field]]
    name = "ifDescr"
    oid = "IF-MIB::ifDescr"
    is_tag = true

  # [Tag] Arista/Cisco 등에서 사용 (필수 추가 ⭐)
  [[inputs.snmp.table.field]]
    name = "ifAlias"
    oid = "IF-MIB::ifAlias"
    is_tag = true
```

---

## 2. Grafana Query 설정 (InfluxQL)

Sankey 패널은 **Table** 형식의 데이터를 원하므로, 필요한 태그(`hostname`, `ifDescr`, `ifAlias`)를 모두 Group By에 포함해야 합니다.

- **Data Source:** InfluxDB (InfluxQL)
- **Format:** Time series (이후 Transform에서 Table로 변환)

```sql
SELECT non_negative_derivative("ifHCInOctets", 1s) * 8 AS "bandwidth"
FROM "snmp"
WHERE "hostname" =~ /^$host$/
  AND "ifDescr" =~ /To/          -- 연결 정보가 있는 포트만 필터링
  AND "ifOperStatus" = 'up'      -- Up 상태인 포트만
  AND $timeFilter
GROUP BY "hostname", "ifDescr", "ifAlias"
```

---

## 3. Transformations 설정 (핵심 로직) ⭐
벤더별로 다른 필드(`ifDescr` vs `ifAlias`)를 하나로 합치고, 긴 텍스트를 정규화하기 위해 **순서대로** 적용해야 합니다.

### Step 1: 데이터 요약

1. **Reduce**:
    - Mode: `Series to rows`
    - Calculations: `Last` (현재 트래픽 기준)
2. **Sort by**: `Last` (Reverse) - 트래픽 높은 순 정렬
3. **Limit**: `5` ~ `10` (상위 N개만 표시)

### Step 2: Nokia/General 장비 이름 추출 (1차)
Nokia처럼 `ifDescr`에 `"To-..."` 정보가 있는 경우를 먼저 추출합니다.

1. **Extract fields**:
    - **Source**: `ifDescr`
    - **Format**: `RegExp`
    - **RegExp**: `.*"To[-_](?<ConnectTo>[^"]+)".*`
    - 설명: `To-` 또는 `To_` 뒤에 오는 따옴표 안의 내용을 `ConnectTo` 필드로 추출.
    - 결과: Nokia는 값이 추출되고, Arista는 빈칸이 됨.

### Step 3: 벤더 정보 병합 (Hybrid Merge)
추출된 Nokia 정보(`ConnectTo`)와 Arista 정보(`ifAlias`)를 합칩니다.

1. **Add field from calculation**:
    - **Mode**: `Binary operation`
    - **Operation**: `ConnectTo` `+` `ifAlias`
    - **Alias**: `FinalTarget`
    - 원리: Nokia는 `ifAlias`가 비어있으므로 `ConnectTo`가 남고, Arista는 `ConnectTo`가 비어있으므로 `ifAlias`가 남습니다.

### Step 4: Source 이름 정규화 (선택사항)
호스트명(`Router_BB3`)을 짧게(`BB3`) 줄입니다.

1. **Extract fields**:
    - **Source**: `hostname`
    - **Format**: `RegExp`
    - **RegExp**: `/.*_(?<ShortSource>.*)/`

### Step 5: 최종 필드 정리

1. **Organize fields by name**:
    - `ShortSource` ➔ `Source` 로 이름 변경
    - `FinalTarget` ➔ `Target` 으로 이름 변경
    - `Last` ➔ `Value` 로 이름 변경
    - 나머지 불필요한 필드(`ifDescr`, `ifAlias`, `ConnectTo` 등)는 **Hide(눈동자 끄기)**
  
---

## 4. 패널 시각화 설정 (Visualization)

- **Visualization**: `Sankey`
- **Sankey Settings (Data Mapping)**:
    - **Source**: `Source`
    - **Target**: `Target`
    - **Weight**: `Value`
- **Standard options > Unit**: `Data rate` > `bits/sec(SI)` (bps)

---
## 💡 결과 예시
이 구성을 통해 아래와 같은 벤더 혼합 환경에서도 통일된 그래프를 얻을 수 있습니다.

|벤더|원본 데이터 위치|처리 과정|최종 결과 (Target)|
|:---|:---|:---|:---|
|Nokia|`ifDescr`: "To-Router_BB3"|Regex 추출|**Router_BB3**|
|Arista|`ifAlias`: "Router_Leaf_1"|Calculation 병합|**Router_Leaf_1**|
