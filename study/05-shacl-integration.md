# 5. W3C SHACL: 그래프를 검사하고 안전하게 게시하기

## 학습 목표

- Data Graph와 Shapes Graph를 구분한다.
- 주요 SHACL 제약을 작성한다.
- 검사 결과에서 대상과 오류 원인을 찾는다.
- AAS·OWL·SHACL을 하나의 등록 흐름으로 연결한다.

## 1. SHACL의 역할

Shapes Constraint Language는 RDF 그래프에 대한 조건을 표현하는 언어입니다.

```text
Data Graph: 등록하려는 장치와 관계
Shapes Graph: 등록할 때 만족해야 하는 조건
                      │
                      ▼
               SHACL Validator
                      │
               conforms + 결과 보고서
```

OWL이 “무엇을 논리적으로 말할 수 있는가”를 다룬다면, SHACL은 “이 그래프가 지정한 조건을 만족하는가”를 검사합니다.

단, SHACL도 현실의 배선이나 제조사 사양을 관측하지 않습니다. 저장된 데이터에 대한 검사입니다.

## 2. 자주 쓰는 제약

| 표현 | 의미 | PLC 예 |
|---|---|---|
| `sh:targetClass` | 검사 대상 클래스 | Motor 개체 |
| `sh:path` | 검사할 속성/경로 | assetId |
| `sh:minCount` | 최소 개수 | ID 필수 |
| `sh:maxCount` | 최대 개수 | ID 하나 |
| `sh:datatype` | 값의 자료형 | decimal |
| `sh:class` | 참조 대상의 클래스 소속 | mappedTo 대상은 IOChannel |
| `sh:nodeKind` | IRI/literal 등 종류 | 대상은 IRI |
| `sh:minExclusive` | 지정 값보다 큼 | 정격 전류는 0 초과 |
| `sh:in` | 허용 값 집합 | 교육 예제에서는 단위 A만 허용 |
| `sh:message` | 사람이 읽는 설명 | 한국어 검사 메시지 |

**focus node**는 실제로 검사 중인 대상입니다. MotorShape가 모터 세 대를 대상으로 하면 각 모터가 차례로 focus node가 됩니다.

`minCount`와 `maxCount`는 해당 focus node에서 지정 path로 연결되는 값의 개수를 검사합니다. `maxCount 1`만으로 그래프 전체의 assetId 중복을 막지는 못합니다.

## 3. 교육용 Shapes Graph

[OWL 장](04-owl.md)의 `model.ttl`을 대상으로 다음을 `shapes.ttl`로 사용합니다.

```turtle
@prefix ex: <https://example.org/plant/> .
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

ex:MotorShape a sh:NodeShape ;
    sh:targetClass ex:Motor ;
    sh:property [
        sh:path ex:assetId ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:datatype xsd:string ;
        sh:message "모터에는 문자열 자산 ID가 정확히 하나 필요합니다."
    ] ;
    sh:property [
        sh:path ex:ratedCurrent ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:datatype xsd:decimal ;
        sh:minExclusive 0 ;
        sh:message "정격 전류는 0보다 큰 decimal 값이어야 합니다."
    ] ;
    sh:property [
        sh:path ex:currentUnit ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:in ("A") ;
        sh:message "교육용 모델은 정격 전류 단위 A만 허용합니다."
    ] .

ex:VariableShape a sh:NodeShape ;
    sh:targetClass ex:Variable ;
    sh:property [
        sh:path ex:mappedTo ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:nodeKind sh:IRI ;
        sh:class ex:IOChannel ;
        sh:message "변수는 IRI로 식별되는 I/O 채널 하나에 매핑되어야 합니다."
    ] .
```

이 규칙을 현장의 모든 변수에 적용하면 안 됩니다. 내부 변수는 I/O 매핑이 필요하지 않을 수 있습니다. 실무에서는 `IOBoundVariable` 같은 전용 클래스나 적절한 target을 사용합니다.

또한 정격 전류 0 초과 조건은 교육용 업무 규칙이지 AAS 공통 표준의 필수 조건이 아닙니다. 장치 종류·결측 상태·사양 정의에 맞춰 설계합니다.

## 4. 검사 결과 예상하기

| 변경 | 기대 결과 |
|---|---|
| 원래 model.ttl | conforms true |
| Motor_01의 assetId 삭제 | minCount 위반 |
| ratedCurrent를 문자열 `"12.5"`로 변경 | datatype 위반 |
| ratedCurrent를 decimal `-1.0`으로 변경 | minExclusive 위반 |
| currentUnit을 mA로 변경 | 이 자료의 in 제약 위반 |
| StartCommand의 매핑 대상을 Motor_01로 변경 | sh:class 위반 |
| 대상 클래스 개체가 한 건도 없음 | 해당 target에 위반이 없을 뿐, 필요한 대상이 존재한다는 증명은 아님 |

“검사 대상이 0건이라 합격”하는 사고를 막으려면 필요한 대상의 존재·예상 대상 개수도 게시 서비스에서 확인합니다.

같은 타입 선언 하나를 삭제했다고 항상 검사를 피할 수 있는 것은 아닙니다. SHACL이 보는 타입·클래스 계층과 선택한 target 방식을 이해하고 실제 케이스로 검증하십시오.

## 5. 선택 실습: 정상과 오류를 자동 확인하기

앞 장과 같은 실습 폴더에 `model.ttl`, `shapes.ttl`을 저장하고, 다음을 `validate_study.py`로 저장합니다.

```python
from rdflib import Graph, Literal, Namespace
from rdflib.namespace import XSD
from pyshacl import validate

EX = Namespace("https://example.org/plant/")
shapes = Graph().parse("shapes.ttl", format="turtle")

def check(graph):
    conforms, report, report_text = validate(
        data_graph=graph,
        shacl_graph=shapes,
        inference="none",
        abort_on_first=False,
    )
    assert isinstance(report, Graph), f"Validation failure: {report}"
    return conforms, report_text

good = Graph().parse("model.ttl", format="turtle")
conforms, report_text = check(good)
assert conforms, report_text
print("PASS: original graph conforms")

bad = Graph()
for triple in good:
    bad.add(triple)
bad.remove((EX.Motor_01, EX.ratedCurrent, None))
bad.add((EX.Motor_01, EX.ratedCurrent, Literal("-1.0", datatype=XSD.decimal)))
conforms, report_text = check(bad)
assert not conforms, "Expected rejection of negative rated current"
print("PASS: negative current is rejected")
print(report_text)
```

앞 장에서 만든 가상환경으로 실행합니다.

```powershell
.\.venv\Scripts\python.exe -m pip install pyshacl
.\.venv\Scripts\python.exe validate_study.py
```

기대 결과는 PASS 두 개와 음수 전류에 대한 위반 보고서입니다. 라이브러리 버전에 따라 보고서 출력 형식은 달라질 수 있습니다.

이 예제는 추론을 `none`으로 고정하고 필요한 타입을 명시합니다. SHACL의 클래스 소속 판정과 전체 OWL 추론은 같지 않습니다.

### 후속 실습

각 오류를 **한 번에 하나씩** 적용해 앞 표의 결과를 확인합니다. `model.ttl` 원본을 보존하고 새 그래프를 복사해서 바꾸면 비교하기 쉽습니다.

다른 제약이 동시에 실패하면 “한 가지 오류를 넣었으니 보고서도 한 줄”이라고 가정하지 마십시오.

## 6. 검사 보고서에서 볼 항목

| 항목 | 읽는 방법 |
|---|---|
| `sh:conforms` | 지정 검사에 적합한가 |
| `sh:focusNode` | 어느 대상에서 실패했는가 |
| `sh:resultPath` | 어느 속성/경로인가 |
| `sh:value` | 문제가 된 값이 있으면 해당 값 |
| `sh:sourceConstraintComponent` | 어떤 제약을 위반했는가 |
| `sh:resultSeverity` | Violation/Warning/Info 등 |
| `sh:resultMessage` | 설명 |

“데이터에 위반이 있다”와 “검사기가 실행되지 못했다”는 다릅니다. 잘못된 Shapes Graph나 미지원 기능을 합격으로 처리하지 않습니다.

severity에 따른 게시 허용 여부도 업무 정책으로 명시합니다. Warning이라고 항상 무시해도 되는 것은 아닙니다.

## 7. OWL 추론과의 상호작용

검사 전에 domain/range 추론을 적용하면 잘못된 참조 대상에도 기대 타입이 추가되어 타입 검사의 의도를 훼손할 수 있습니다.

초기 권장 방침:

1. Parser/Mapper가 타입과 출처를 명시한다.
2. 명시적 후보 데이터로 등록 조건을 검사한다.
3. 필요한 추론은 검사 조건과의 관계를 테스트한 뒤 적용한다.
4. 추론으로 추가한 사실과 원래 데이터를 구분한다.

SHACL의 클래스 대상·클래스 제약에는 RDFS 클래스 계층을 다루는 부분이 있지만, 이것을 전체 OWL 기능의 자동 추론으로 해석하지 않습니다.

또한 외부 웹에 타입이 정의되어 있다고 검사기가 반드시 그 웹을 찾아 읽는 것은 아닙니다. 검사에 사용할 데이터·스키마 범위를 명시해야 합니다.

## 8. 추가 과제: 그래프 전체의 ID 중복

`maxCount 1`은 각 장치의 ID가 하나임을 검사할 뿐입니다. 두 장치가 동일한 assetId를 가진 문제는 별도로 검사합니다.

학습용 SPARQL:

```sparql
PREFIX ex: <https://example.org/plant/>

SELECT ?assetId (COUNT(DISTINCT ?device) AS ?count)
WHERE {
  ?device ex:assetId ?assetId .
}
GROUP BY ?assetId
HAVING (COUNT(DISTINCT ?device) > 1)
```

실무에서는 제품 타입과 설치 개체, 여러 표현 객체, 허용 별칭 정책을 먼저 정합니다. 후보·승인 데이터 중 어느 범위를 유일성 검사 대상으로 삼을지도 명시합니다.

SHACL-SPARQL 제약으로 표현할 수도 있지만, 먼저 위 질의로 조건의 의미를 확인하십시오.

## 9. 종합 과제: AI 추출부터 게시까지

### 상황

AI가 데이터시트에서 다음을 추출했습니다.

```text
제품 모델: EM-100
정격 전류: 12.5 A
설치 위치: 미확인
PLC 출력: Channel_03인 것으로 추정
```

### 제출할 것

1. 제품 타입과 설치 개체를 분리한 ID 정책.
2. AAS 구조도와 채택할 템플릿 후보.
3. 필드별 근거와 미확인 상태.
4. BaSyx 저장 대상과 Registry 등록 정책.
5. RDF 매핑표.
6. SHACL 검사 조건과 오류 예제.
7. 게시 조건과 실패 처리.

### 해답 예시의 핵심

| 단계 | 처리 | 완료 조건 |
|---|---|---|
| AI 추출 | 값·단위·모델·출처 페이지를 후보화 | 근거 추적 가능 |
| 의미 확인 | 정격/실측과 모델 적용 범위 확인 | 개념·단위 확정 |
| AAS 구조 검사 | 메타모델·스키마·템플릿 제약 검사 | 채택 판본의 제약 적합 |
| 사람 승인 | 불확실 항목·신규 매핑 검토 | 미확인을 확정값으로 만들지 않음 |
| 저장 | 승인 버전을 AAS Repository에 보관 | 저장 버전 재조회 가능 |
| RDF 투영 | 대응 ID·정규화 값·출처 변환 | 매핑 버전 추적 가능 |
| SHACL 등 검사 | 후보 그래프의 타입·값·ID 검사 | 모든 게시 조건 충족 |
| 게시 | 승인 그래프 활성화 | 부분 실패·불일치 표시 가능 |

설치 위치가 미확인이면 필수 여부에 따라 미확인 또는 승인 대기로 둡니다. `Unknown`이라는 가상의 실제 장소를 확정 정보로 생성하지 않습니다.

“Channel_03 추정”은 승인 배선이 아닙니다. I/O 설정·배선표 등의 근거를 확인할 때까지 후보로 남깁니다. SHACL 통과만으로 배선이 맞다는 증거가 되지는 않습니다.

### 매핑표 예

| 출처 | 도메인 표현 | 함께 보존할 것 |
|---|---|---|
| AAS의 자산 ID | Motor_01의 assetId | 대응 AAS ID |
| 정격 전류 Property | ratedCurrent와 단위 | 원값·개념 ID·출처·AAS 버전 |
| PLC 심볼/I/O 설정 | Variable mappedTo IOChannel | 프로젝트 revision과 소스 위치 |
| 승인 배선표 | IOChannel connectedTo Contactor | 배선표 revision과 검토 상태 |
| 제조사 매뉴얼 | ProductType documentedBy Document | 적용 모델·문서 revision |

단위·의미 ID가 미확인이면 값을 조용히 추측하지 않고 매핑 검토 상태로 남깁니다.

## 10. 다섯 기술을 최종 연결하기

```text
IDTA 사양: 채택할 구조·API·교환 규칙
                    │
                    ▼
AAS 메타모델: 장치 정보 구조
                    │ 검증·승인
                    ▼
BaSyx: 저장·조회·서비스 탐색
                    │ 도메인 투영
                    ▼
OWL/RDFS: 개념·관계·추론의 의미
                    │
                    ▼
SHACL: 게시 후보 그래프의 등록 조건 검사
                    │
                    ▼
지식 그래프 → 관계 검색 / 문서 RAG / 시간 구간 분석
```

OWL/RDFS는 매핑 전부터 의미 모델의 기준으로 존재합니다. 위 그림은 책임의 연결이며, 저장 후에야 OWL을 처음 설계하라는 권장은 아닙니다.

## 11. 최종 구술 시험

다음 다섯 가지를 자신의 말로 설명할 수 있으면 설계의 기초가 갖춰진 것입니다.

1. AAS Property와 OWL data property는 왜 동일하지 않은가?
2. Registry에 등록됐는데도 왜 데이터를 못 가져올 수 있는가?
3. OWL domain이 왜 필수 입력 검사를 대체하지 못하는가?
4. SHACL에 적합해도 왜 실제 장치 사양을 보증하지 못하는가?
5. 고빈도 I/O 이력을 왜 별도 시계열 DB에 저장하는가?

### 해답 요점

1. AAS Property는 표준 모델 요소, OWL data property는 개체와 literal 사이의 관계이므로 투영 규칙이 필요합니다.
2. 오래된 endpoint, 저장소 장애, 참조 불일치, 권한 등이 원인일 수 있습니다.
3. domain은 타입을 도출하는 의미이지 값의 존재를 요구하는 입력 규칙이 아닙니다.
4. 지정 조건에 대한 저장 데이터 검사이므로 원문·배선·실물 사실 확인은 별도입니다.
5. 저장량·시간 검색·집계·보존 기간 요구가 다릅니다. 의미 모델의 ID와 의미로 연결합니다.

## 공식 문서 읽기 순서

[W3C SHACL](https://www.w3.org/TR/shacl/)에서 SHACL Example → Shapes and Constraints → Validation and Graphs → Core Constraint Components 순으로 읽습니다.

처음에는 Core 제약부터 시작합니다. SPARQL 제약과 복잡한 경로는 필요한 조건이 Core만으로 표현되지 않을 때 추가합니다.

돌아가기: [학습 로드맵](README.md).
