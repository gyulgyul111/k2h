# 0. 공통 기초: 장치 정보를 “기계가 이해할 수 있게” 만든다는 것

## 학습 목표

이 장을 마치면 URI/IRI, JSON, RDF, 클래스, 개체, 속성, API, 스키마를 구분할 수 있습니다.

## 1. 엑셀의 장치 목록만으로 부족한 이유

다음 두 표를 생각해 보십시오.

| 설비 목록 | 값 |
|---|---|
| 장치명 | Motor_01 |
| 모델 | EM-100 |
| 전류 | 12.5 |

| PLC 목록 | 값 |
|---|---|
| 변수 | StartCommand |
| 주소 | 출력 채널 3 |
| 대상 | Motor_01 |

사람은 상황을 보고 연결할 수 있지만, 시스템은 다음을 모릅니다.

- `12.5`는 정격 전류인가, 현재 전류인가? 단위는 A인가 mA인가?
- 다른 공장의 `Motor_01`과 같은 장치인가?
- 출력 채널이 모터에 직접 연결되는가, 접촉기를 거치는가?
- 두 파일이 같은 시점의 설비 구성인가?

**구조화**는 필드를 정하는 일, **의미화**는 그 필드와 관계가 무엇을 뜻하는지 명시하는 일입니다. 구조와 의미에 더해 출처·버전·품질이 필요합니다.

## 2. JSON: 정보를 담는 표기법

```json
{
  "assetId": "urn:example:asset:motor-01",
  "model": "EM-100",
  "ratedCurrent": {
    "value": 12.5,
    "unit": "A"
  }
}
```

- `{}`는 이름–값 쌍을 담는 객체.
- `[]`는 순서가 있는 배열.
- `"EM-100"`은 문자열.
- `12.5`는 숫자.
- `true`/`false`는 Boolean.
- `null`은 값이 없음을 나타내는 표기지만, 업무상 결측의 이유까지 자동 설명하지는 않습니다.

위 JSON은 **임의로 만든 업무 데이터**입니다. JSON 형식이라고 AAS인 것은 아닙니다. AAS가 되려면 AAS 메타모델의 구조와 제약을 따라야 합니다.

## 3. 식별자: 이름과 ID를 분리하기

| 항목 | 예 | 의미 |
|---|---|---|
| 표시 이름 | Motor_01 | 사람이 읽는 이름 |
| 자산 ID | urn:example:asset:motor-01 | 실제 자산을 식별하는 값 |
| 제품 모델 | EM-100 | 제품 타입 식별 정보 |
| API 주소 | 서버의 자산 조회 URL | 정보를 가져오는 위치 |
| 의미 ID | https://example.org/concepts/RatedCurrent | 속성의 의미를 식별 |

URI는 자원을 식별하는 문자열 체계이고, IRI는 국제 문자를 사용할 수 있도록 확장한 체계입니다. 처음에는 “다른 것과 혼동하지 않도록 붙인 식별자”로 이해하면 충분합니다.

- `https://...` 식별자는 웹으로 조회 가능하도록 설계할 수 있습니다.
- `urn:...` 식별자는 곧바로 웹 주소가 되는 것은 아닙니다.
- 식별자가 있다고 해당 서버가 실제 존재하는 것은 아닙니다.
- 이름 변경·서버 이전 때문에 자산 ID까지 바뀌지 않게 설계합니다.

## 4. RDF: 정보를 문장 세 조각으로 표현하기

RDF는 Resource Description Framework입니다. 핵심은 **트리플(triple)**입니다.

```text
주어                술어                 목적어
Motor_01            모델을 가진다         "EM-100"
StartCommand        채널에 매핑된다        Channel_03
Contactor_K1        구동한다               Motor_01
```

주어와 술어는 보통 IRI로 식별합니다. 목적어는 개체 IRI 또는 문자열·숫자 같은 literal일 수 있습니다. RDF에서는 blank node도 쓸 수 있지만 초기 모델은 명시적 ID 위주로 시작하는 것이 쉽습니다.

### Turtle로 써 보기

Turtle은 RDF를 사람이 읽기 편하게 쓰는 문법입니다.

```turtle
@prefix ex: <https://example.org/plant/> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

ex:Motor_01 a ex:Motor ;
    ex:model "EM-100" ;
    ex:ratedCurrent "12.5"^^xsd:decimal .

ex:StartCommand ex:mappedTo ex:Channel_03 .
ex:Contactor_K1 ex:actuates ex:Motor_01 .
```

- `ex:Motor_01`은 긴 IRI의 축약입니다.
- `a`는 `rdf:type`의 축약입니다. “Motor_01은 Motor에 속하는 개체”라는 뜻입니다.
- `;`는 같은 주어에 대해 다른 술어를 이어 씁니다.
- `.`는 문장을 끝냅니다.
- `^^xsd:decimal`은 decimal 자료형임을 표시합니다.

이 예제의 `ex:model`과 `ex:actuates` 등은 학습용 도메인 어휘이며 공식 AAS 어휘가 아닙니다. 단위는 여기서 생략했지만 실무 모델에서는 반드시 별도로 표현해야 합니다.

**주의:** 데이터에서 edge 이름을 만들었다고 그 edge가 의미하는 물리적 사실이 자동 입증되지는 않습니다.

## 5. 클래스·개체·속성

| 용어 | 의미 | PLC 예 |
|---|---|---|
| Class, 클래스 | 사물을 분류하는 개념 | Motor, IOChannel, Variable |
| Individual, 개체 | 구체적인 대상 | Motor_01, Channel_03 |
| Object property | 개체와 개체의 관계 | 변수 mappedTo 채널 |
| Data property | 개체와 값의 관계 | 모터 ratedCurrent 12.5 |
| Annotation | 이름·설명 등 주석 정보 | 한국어 라벨 |

프로그래밍 언어의 클래스와 닮았지만 동일하지 않습니다. 특히 OWL은 객체 생성 코드가 아니라 지식을 표현하는 논리 언어입니다.

## 6. 용어 지도로 전체 보기

```text
JSON / XML / Turtle     어떻게 써서 전달할 것인가?
RDF                    사실을 그래프로 어떻게 표현할 것인가?
OWL / RDFS             개념과 관계의 의미는 무엇인가?
SHACL                  그래프가 등록 조건을 만족하는가?
SPARQL                 그래프에 무엇을 물어볼 것인가?
AAS                    자산 정보를 어떤 표준 모델로 교환할 것인가?
BaSyx                  AAS를 어떻게 저장하고 제공할 것인가?
```

메타모델은 모델을 만들기 위한 규칙입니다. 스키마는 형식·구조 제약을 기계적으로 검사하는 데 사용될 수 있습니다. 템플릿은 특정 업무를 위해 채울 항목을 더 구체적으로 정합니다. 이 셋은 연결되어 있지만 동의어는 아닙니다.

## 7. API: 다른 프로그램이 정보를 요청하는 창구

HTTP API에서는 일반적으로 다음 역할을 구분합니다.

| 동작 | 의미 | 주의 |
|---|---|---|
| GET | 조회 | 조회 권한 필요 |
| POST | 생성 또는 정해진 처리 요청 | 정확한 의미는 API 사양에 따름 |
| PUT/PATCH | 갱신 | 전체 대체와 부분 변경을 구분 |
| DELETE | 삭제 | 원장·검색·이력 정책 확인 |

HTTP 성공 코드가 실제 장치 사양의 정확성을 보증하지는 않습니다. 요청 실패·권한 거절·데이터 부재도 별도 처리해야 합니다.

## 8. 손으로 푸는 실습

다음을 트리플로 적어 보십시오.

1. Motor_01은 Motor다.
2. Motor_01의 모델은 EM-100이다.
3. StartCommand는 Channel_03에 매핑된다.
4. Channel_03은 Contactor_K1에 연결된다.
5. Contactor_K1은 Motor_01을 구동한다.

### 해답

```turtle
@prefix ex: <https://example.org/plant/> .

ex:Motor_01 a ex:Motor ; ex:model "EM-100" .
ex:StartCommand ex:mappedTo ex:Channel_03 .
ex:Channel_03 ex:connectedTo ex:Contactor_K1 .
ex:Contactor_K1 ex:actuates ex:Motor_01 .
```

### 확인 문제

**Q1. 같은 모델의 모터 두 대는 같은 개체인가?**  
아닙니다. 제품 타입은 공유해도 시리얼·설치 위치·배선이 다른 설치 개체입니다.

**Q2. `12.5`만 기록하면 충분한가?**  
아닙니다. 무엇의 값인지, 단위, 정격/실측 여부, 출처가 필요합니다.

**Q3. RDF를 쓰면 기존 DB를 모두 없애야 하나?**  
아닙니다. 관계 그래프, 업무 DB, 원문 저장소, 시계열 저장소를 목적에 맞게 연결합니다.

## 공식 문서 읽기

[W3C RDF 1.1 Primer](https://www.w3.org/TR/rdf11-primer/)의 RDF Data Model과 RDF Vocabularies를 먼저 읽으십시오. 문법 변형을 모두 암기하기보다 트리플·IRI·literal·타입을 설명하는 데 집중하십시오.

다음: [IDTA 사양을 읽는 법](01-idta-specifications.md).
