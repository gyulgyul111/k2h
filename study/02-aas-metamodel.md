# 2. AAS Metamodel Part 1: 장치 정보를 실제로 표현하기

## 학습 목표

- AAS·AssetInformation·Submodel·SubmodelElement를 연결한다.
- 네 가지 주요 식별자를 구분한다.
- Reference를 읽고 교육용 AAS JSON을 설명한다.

## 1. 전체 구조부터 보기

```text
Environment: 교환·직렬화 시 모델들을 모아 담는 컨테이너
  ├─ AssetAdministrationShell
  │      ├─ id
  │      ├─ assetInformation
  │      └─ submodels: Submodel을 가리키는 Reference 목록
  ├─ Submodel
  │      ├─ id / idShort / semanticId
  │      └─ submodelElements
  │             ├─ Property
  │             ├─ MultiLanguageProperty
  │             ├─ File
  │             └─ Collection / List / 기타 요소
  └─ ConceptDescription
         └─ 개념 설명과 관련 데이터 사양
```

AAS가 모든 데이터를 내부에 직접 중첩해서 갖는다고 생각하지 마십시오. AAS는 서브모델을 Reference로 가리킬 수 있고, 서비스 배포에서는 서브모델 저장소가 따로 있을 수도 있습니다.

Environment는 여기서 함께 담아 전달하는 구조이며, 반드시 하나의 DB 테이블 또는 서버를 의미하지 않습니다.

## 2. 주요 구성 요소

| 구성 요소 | 역할 | 모터 예 |
|---|---|---|
| AssetAdministrationShell | 자산의 디지털 대표 구조 | Motor_01의 AAS |
| AssetInformation | 어떤 자산을 나타내는지 | 설치 모터 자산 ID와 Instance 구분 |
| Submodel | 특정 측면의 정보 묶음 | 학습용 TechnicalData |
| Property | 자료형이 있는 값 | 정격 전류 12.5 |
| MultiLanguageProperty | 언어별 문자열 값 | 한국어·영어 제품 설명 |
| File | 파일 위치와 콘텐츠 유형 | 매뉴얼 PDF 참조 |
| SubmodelElementCollection | 이름 있는 요소들의 그룹 | 전기 사양 그룹 |
| SubmodelElementList | 리스트 의미와 공통 제약을 가진 요소 묶음 | 정해진 타입의 항목 목록 |
| ReferenceElement | 다른 모델/자원을 가리키는 값 | 관련 요소 참조 |
| RelationshipElement | 두 참조 대상 간 관계 표현 | 구성품 관계 후보 |
| ConceptDescription | 속성 등의 개념 설명 | 정격 전류의 정의 |

Collection과 List는 단순히 “둘 다 배열처럼 쓰면 되는 것”이 아닙니다. 특히 List의 항목 타입·자료형·의미 관련 제약은 해당 판본을 확인해야 합니다.

Operation 같은 요소를 표현할 수도 있지만, 모델에 Operation이 있다는 이유로 PLC 제어를 허용해서는 안 됩니다. 실제 실행 API·권한·안전 정책은 별도입니다.

## 3. 네 가지 식별자

| 필드 | 무엇을 식별하나? | 예 |
|---|---|---|
| `id` | AAS·Submodel·ConceptDescription처럼 Identifiable인 모델 객체 | urn:example:aas:motor-01 |
| `idShort` | 문맥 내 요소를 사람이 다루기 위한 짧은 이름 | RatedCurrent |
| `globalAssetId` | AAS가 나타내는 자산 | urn:example:asset:motor-01 |
| `semanticId` | 모델/요소가 나타내는 의미 | 정격 전류 개념의 ID |

### 반드시 이해할 차이

```text
Motor_01이라는 실제 모터
   ↕ globalAssetId
이를 표현하는 AAS
   ↕ id
AAS에 담긴 정격 전류 속성
   ↕ semanticId
"정격 전류"라는 개념
```

`idShort`는 전역 식별자가 아닙니다. 다른 장치에서도 `RatedCurrent`를 쓸 수 있습니다. 필수 여부와 문맥 내 제약은 요소 종류에 따라 확인합니다.

`semanticId`는 값 자체가 아닙니다. 같은 정격 전류 의미를 나타내는 두 Property는 semanticId를 공유할 수 있어도 각 장치의 값은 다를 수 있습니다.

## 4. Type과 Instance

- **제품 타입:** EM-100이라는 제품 모델의 공통 사양.
- **설치 인스턴스:** 시리얼 SN-DEMO-001을 가진 Motor_01.

같은 제품 20대의 공통 사양과 개별 설치 정보를 분리하면 중복이 줄어듭니다. 단, 타입·인스턴스 관계와 사양 상속 정책을 명확히 해야 합니다.

`assetInformation.assetKind`는 표현하는 자산의 구분입니다. `Submodel.kind`는 서브모델이 템플릿인지 인스턴스인지 구분하는 다른 필드입니다. 이름이 비슷하다고 같은 의미로 읽지 마십시오.

## 5. Reference: 문자열 링크보다 구조가 있습니다

Reference는 `type`과 `keys` 등으로 구성됩니다.

### ModelReference

AAS 모델 객체를 가리키는 참조입니다.

```json
{
  "type": "ModelReference",
  "keys": [
    {
      "type": "Submodel",
      "value": "urn:example:submodel:motor-01:technical"
    }
  ]
}
```

“이 ID를 가진 Submodel을 가리킨다”는 뜻입니다. 내부 요소를 가리키려면 후속 key를 사용하지만 순서·타입 제약을 따라야 합니다.

### ExternalReference

외부 자원이나 의미 정의 등을 가리키는 참조입니다.

```json
{
  "type": "ExternalReference",
  "keys": [
    {
      "type": "GlobalReference",
      "value": "https://example.org/concepts/RatedCurrent"
    }
  ]
}
```

웹 URL 모양이어도 대상의 존재·접근 권한·정의 내용이 자동 검증되는 것은 아닙니다.

## 6. 교육용 AAS Environment JSON

다음은 공통 구조를 배우기 위한 최소 사례입니다. **공식 Technical Data 템플릿 준수 예제가 아닙니다.** 단위·개념 정의를 완성한 운영 데이터도 아닙니다.

```json
{
  "assetAdministrationShells": [
    {
      "modelType": "AssetAdministrationShell",
      "id": "urn:example:aas:motor-01",
      "idShort": "Motor01",
      "assetInformation": {
        "assetKind": "Instance",
        "globalAssetId": "urn:example:asset:motor-01"
      },
      "submodels": [
        {
          "type": "ModelReference",
          "keys": [
            {
              "type": "Submodel",
              "value": "urn:example:submodel:motor-01:technical"
            }
          ]
        }
      ]
    }
  ],
  "submodels": [
    {
      "modelType": "Submodel",
      "id": "urn:example:submodel:motor-01:technical",
      "idShort": "TechnicalData",
      "kind": "Instance",
      "semanticId": {
        "type": "ExternalReference",
        "keys": [
          {
            "type": "GlobalReference",
            "value": "https://example.org/concepts/TechnicalData"
          }
        ]
      },
      "submodelElements": [
        {
          "modelType": "Property",
          "idShort": "RatedCurrent",
          "valueType": "xs:decimal",
          "value": "12.5",
          "semanticId": {
            "type": "ExternalReference",
            "keys": [
              {
                "type": "GlobalReference",
                "value": "https://example.org/concepts/RatedCurrent"
              }
            ]
          }
        }
      ]
    }
  ],
  "conceptDescriptions": []
}
```

### 읽는 순서

1. AAS의 `id`와 자산 `globalAssetId`가 서로 다름을 찾습니다.
2. AAS의 submodels 참조 값과 실제 Submodel의 id가 일치하는지 찾습니다.
3. Submodel의 의미와 Property의 의미가 다름을 찾습니다.
4. Property의 `valueType`은 decimal인데 JSON의 `value`는 문자열임을 확인합니다. AAS 직렬화에서는 이를 보고 실제 값의 lexical 형식을 해석합니다.
5. `conceptDescriptions`가 비어 있으므로 이 패키지 안에서 모든 의미 정의를 읽을 수는 없음을 확인합니다.

### 왜 `unit`을 Property에 임의로 넣지 않았나?

AAS Property의 임의 필드로 `"unit": "A"`를 추가하면 표준 구조를 벗어날 수 있습니다. 단위는 적용 템플릿과 개념 설명의 데이터 사양 등 정해진 표현을 따릅니다.

이 예제는 구조에만 집중합니다. 운영에서는 **정격 전류의 정의·단위 A·표준 개념 매핑·출처**를 적합한 템플릿과 데이터 사양으로 완성해야 합니다.

## 7. AAS와 RDF는 두 단계로 생각하기

```text
AAS 구조를 보존한 공식 RDF 교환 표현
          │
          └─ 필요 정보를 도메인 의미로 투영
                  ├─ Motor_01 a Motor
                  ├─ Motor_01 ratedCurrent ...
                  └─ Channel_03 connectedTo Contactor_K1
```

AAS의 구조를 RDF로 표현하는 것과 PLC 관계를 정의하는 것은 다릅니다. 도메인 투영에서는 다음이 필요합니다.

- AAS ID와 도메인 개체 IRI의 대응표.
- semanticId와 도메인 속성의 매핑 규칙.
- 단위 정규화 및 원래 값의 보존.
- 데이터 revision·출처·검토 상태.
- 물리 배선의 별도 근거.

`globalAssetId`와 RDF IRI가 다르다고 무조건 `owl:sameAs`로 연결하지 않습니다. 자산과 그 표현 객체가 정말 동일한 대상인지 먼저 판단해야 합니다.

## 8. 실습과 해답

**과제 A:** Motor_02를 추가할 때 무엇을 바꿉니까?

**해답:** 실제 자산 ID, AAS ID, 인스턴스 Submodel ID, 시리얼 등 개별 정보를 바꿉니다. 동일한 의미의 Property semanticId는 유지할 수 있습니다. 복사한 데이터가 실제 장치 사양과 일치하는지 별도 검토합니다.

**과제 B:** RatedCurrent를 NominalCurrent라는 idShort로 바꿨는데 의미는 그대로입니다. 의미 ID도 반드시 바꿉니까?

**해답:** 의미가 같다면 바꿀 이유가 없습니다. 다만 공식 템플릿의 명명 제약과 기존 요소 참조 경로의 영향을 확인합니다.

**과제 C:** Submodel 참조는 있지만 해당 Submodel을 찾을 수 없습니다. 정상으로 처리해도 됩니까?

**해답:** 외부 서비스에 존재할 수 있으므로 즉시 “존재하지 않는다”고 단정하지 않습니다. 그러나 소비에 필요한 참조를 해석할 수 없는 상태는 명시적으로 보고하고 정상 완성 데이터로 취급하지 않습니다.

## 공식 문서 읽기 순서

1. [Part 1](https://industrialdigitaltwin.io/aas-specifications/IDTA-01001/v3.1/index.html)의 메타모델 구조.
2. [Core](https://industrialdigitaltwin.io/aas-specifications/IDTA-01001/v3.1/spec-metamodel/core.html): AssetAdministrationShell, AssetInformation, Submodel, Property.
3. [Common](https://industrialdigitaltwin.io/aas-specifications/IDTA-01001/v3.1/spec-metamodel/common.html): Referable, Identifiable, HasSemantics.
4. [Referencing](https://industrialdigitaltwin.io/aas-specifications/IDTA-01001/v3.1/spec-metamodel/referencing.html): Reference, Key, 타입별 제약.

다음: [BaSyx](03-basyx.md).
