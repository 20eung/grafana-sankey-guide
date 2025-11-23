# Grafana Sankey 패널 구성 가이드: Network Traffic Flow

이 문서는 Grafana의 **Sankey Panel**을 사용하여 네트워크 장비의 트래픽 흐름(Source ➔ Target)을 시각화하는 방법을 정리합니다.
특히 **Nokia**(`ifDescr` 사용)와 **Arista/Cisco**(`ifAlias` 사용) 장비가 혼재된 환경에서 **벤더에 상관없이 인터페이스 설명을 표준화**하여 보여주는 기법을 포함합니다.

## 1. 사전 준비: Telegraf 설정 (ifAlias 수집)

Arista 등 일부 벤더는 인터페이스 설명(Description)을 `ifAlias` OID에 저장합니다. 따라서 Telegraf에서 이를 **Tag**로 수집해야 합니다.

**`telegraf.conf` 설정 예시:**

```toml
  version = 2
  community = "private"

  interval = "30s"
  timeout = "2s"
  retries = 1

  # 1. 시스템 이름 (Hostname)
  [[inputs.snmp.field]]
    name = "hostname"
    oid = "RFC1213-MIB::sysName.0"
    is_tag = true

  [[inputs.snmp.field]]
    name = "uptime"
    oid = "DISMAN-EVENT-MIB::sysUpTimeInstance"

  # 2. 인터페이스 정보 테이블(최적화)
  [[inputs.snmp.table]]
    name = "snmp"
    inherit_tags = [ "hostname" ]

    # 필드 정의: 필요한 것만 정확히 가져옴.

    # [태그] 인터페이스 이름
    [[inputs.snmp.table.field]]
      name = "ifName"
      oid = "IF-MIB::ifName"
      is_tag = true

    # [태그] 인터페이스 설명 (Description)
    # Nokia나 일부 장비는 Description을 여기에 저장합니다.
    [[inputs.snmp.table.field]]
      name = "ifDescr"
      oid = "IF-MIB::ifDescr"
      is_tag = true

    # [태그] 인터페이스 별칭 (Alias)
    # Arista/Cisco나 일부 장비는 Description을 여기에 저장합니다.
    [[inputs.snmp.table.field]]
      name = "ifAlias"
      oid = "IF-MIB::ifAlias"
      is_tag = true

    # [태그] 필터링용 상태값 (매우 중요)
    [[inputs.snmp.table.field]]
      name = "ifOperStatus"
      oid = "IF-MIB::ifOperStatus"
      is_tag = true

    # [데이터] 64-bit 카운터 (트래픽 In)
    [[inputs.snmp.table.field]]
      name = "ifHCInOctets"
      oid = "IF-MIB::ifHCInOctets"

    # [데이터] 64-bit 카운터 (트래픽 Out)
    [[inputs.snmp.table.field]]
      name = "ifHCOutOctets"
      oid = "IF-MIB::ifHCOutOctets"

  # 3. 데이터 다이어트 (필터링)
  # 중요: 상태가 'up'(1)인 인터페이스의 데이터만 InfluxDB로 보냄.
  # down된 포트나 관리 목적이 아닌 포트의 불필요한 0 데이터를 버려서 DB 부하를 줄임.
  [inputs.snmp.tagpass]
    ifOperStatus = ["1", "up"]

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
  AND "ifDescr" =~ /To/          -- 특정 문자를 포함하는 포트만 포함하는 필터링
  AND "ifDescr" !~ /^p/          -- 특정 문자로 시작하는 포트는 제거하는 필터링
  AND "ifName" !~ /system/       -- 특정 문자로 시작하는 포트는 제거하는 필터링
  AND $timeFilter
GROUP BY "hostname", "ifName", "ifDescr", "ifAlias"
```

---

## 3. Transformations 설정 (핵심 로직) ⭐
벤더별로 다른 필드(`ifDescr` vs `ifAlias`)를 하나로 합치고, 긴 텍스트를 정규화하기 위해 **순서대로** 적용해야 합니다.

### Step 1: 데이터 요약

1. **Reduce**:
    - Mode: `Series to rows`
    - Calculations: `Last` (현재 트래픽 기준)
    - Labels to fields: `On`
2. **Sort by**:
    - Field: `Last`
    - Reverse: `On` (트래픽 높은 순 정렬)
3. **Limit**: `5` ~ `10` (상위 N개만 표시)

### Step 2: Source 이름 정규화 (선택사항)
호스트명(`Korea_Cisco_Seoul_BB3`)을 짧게(`BB3`) 줄입니다.

4. **Extract fields**:
    - **Source**: `hostname`
    - **Format**: `RegExp`
    - **RegExp**: `/.*_(?<ShortSource>.*)/`
    - 설명: `_` 문자 앞의 모든 문자열을 제외하고 이후 내용을 `ShortSource` 필드로 추출.

### Step 3: 최종 필드 정리

5. **Organize fields by name**:
    - `ShortSource` ➔ `Source` 로 이름 변경
    - `ifName` ➔ `Interface` 으로 이름 변경
    - `ifAlias` ➔ `Target` 으로 이름 변경
    - `Last` ➔ `Value` 로 이름 변경
    - 나머지 불필요한 필드(`hostname`, `ifDescr`)는 **Hide(눈동자 끄기)**
  
---

## 4. 패널 시각화 설정 (Visualization)

- **Visualization**: `Sankey Panel`
- **Sankey Settings (Data Mapping)**:
    - **Value Field**: `Value`
- **Standard options > Unit**: `Data rate` > `bits/sec(SI)` (bps)

