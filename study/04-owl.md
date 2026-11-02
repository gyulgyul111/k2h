# 4. W3C OWL 2: PLC와 장치의 의미 모델 만들기

## 학습 목표

- RDF, RDFS, OWL의 관계를 설명한다.
- 최소 TBox와 ABox를 작성한다.
- domain/range·존재 제약·동일성에 관한 대표적인 오해를 피한다.
- 온톨로지를 PLC 질의에 적용한다.

## 1. 왜 AAS 다음에 OWL인가?

AAS는 자산 정보를 교환하기 위한 공통 모델입니다. 그러나 다음 질문에는 PLC와 설치 구성의 도메인 의미가 추가로 필요합니다.

> StartCommand가 어떤 출력 채널에 매핑되며, 그 채널은 어떤 접촉기를 통해 어떤 모터를 구동하는가?

OWL은 클래스·속성·개체와 논리적 공리를 표현합니다. RDF는 그 지식을 그래프로 표현하는 바탕이고, RDFS는 기본적인 클래스·속성 관계를 제공합니다.

| 기술 | 기본 역할 |
|---|---|
| RDF | 사실의 그래프 표현 |
| RDFS | 클래스 계층, 속성 domain/range 등 기본 의미 |
| OWL | 더 풍부한 논리적 의미·동일성·제약 공리 |
| SPARQL | 그래프 질의 |

OWL은 자연어를 읽는 AI 모델이 아닙니다. 명시적으로 쓴 공리를 reasoner가 논리적으로 처리하는 언어입니다.

## 2. 먼저 질문을 정하기

온톨로지의 “유능함”을 확인할 질문을 **competency question**이라고 부릅니다.

초기에는 다음 다섯 개면 충분합니다.

1. Motor_01을 구동하는 접촉기는 무엇인가?
2. 접촉기로 연결되는 출력 채널은 무엇인가?
3. 그 채널에 매핑된 변수는 무엇인가?
4. 장치를 표현하는 AAS ID는 무엇인가?
5. 적용 가능한 매뉴얼 버전은 무엇인가?

클래스를 수백 개 만드는 것보다 이 질문에 출처·버전과 함께 답할 수 있는지가 중요합니다.

## 3. TBox와 ABox

```text
TBox: Motor는 Device의 하위 클래스다.
      actuates는 Device 간 관계다.

ABox: Motor_01은 Motor다.
      Contactor_K1은 Motor_01을 구동한다.
```

TBox와 ABox는 개념적 구분입니다. 반드시 서로 다른 데이터베이스여야 하거나 Mapper 다음에 TBox가 생성되어야 하는 것은 아닙니다.

## 4. 최소 모델: 한 파일로 읽기

다음 Turtle을 교육용 `model.ttl`로 저장하여 선택 실습에 사용할 수 있습니다.

```turtle
@prefix ex: <https://example.org/plant/> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

ex:PlantOntology a owl:Ontology .

ex:Device a owl:Class .
ex:Motor a owl:Class ; rdfs:subClassOf ex:Device .
ex:Contactor a owl:Class ; rdfs:subClassOf ex:Device .
ex:IOChannel a owl:Class .
ex:Variable a owl:Class .

ex:mappedTo a owl:ObjectProperty ;
    rdfs:domain ex:Variable ;
    rdfs:range ex:IOChannel .

ex:connectedTo a owl:ObjectProperty ;
    rdfs:domain ex:IOChannel ;
    rdfs:range ex:Device .

ex:actuates a owl:ObjectProperty ;
    rdfs:domain ex:Device ;
    rdfs:range ex:Device .

ex:assetId a owl:DatatypeProperty ;
    rdfs:domain ex:Device ;
    rdfs:range xsd:string .

ex:ratedCurrent a owl:DatatypeProperty ;
    rdfs:domain ex:Motor ;
    rdfs:range xsd:decimal .

ex:currentUnit a owl:DatatypeProperty ;
    rdfs:range xsd:string .

ex:Motor_01 a ex:Motor ;
    ex:assetId "urn:example:asset:motor-01" ;
    ex:ratedCurrent "12.5"^^xsd:decimal ;
    ex:currentUnit "A" .

ex:Contactor_K1 a ex:Contactor ;
    ex:actuates ex:Motor_01 .

ex:Channel_03 a ex:IOChannel ;
    ex:connectedTo ex:Contactor_K1 .

ex:StartCommand a ex:Variable ;
    ex:mappedTo ex:Channel_03 .
```

단위 문자열은 교육용 단순화입니다. 실무에서는 관리되는 단위 어휘와 수량 모델을 검토합니다. 관계 이름도 공식 AAS 어휘가 아니라 사내 도메인 모델 예시입니다.

## 5. 이 모델에서 추론되는 것

`Motor_01 a Motor`와 `Motor subClassOf Device`로부터 reasoner는 `Motor_01 a Device`를 추론할 수 있습니다.

명시하지 않았지만 공리에서 도출할 수 있는 지식을 얻는 것이 추론입니다. PLC의 실제 동작이나 물리 배선을 자동 관측한 것은 아닙니다.

## 6. domain/range는 입력 검사가 아닙니다

```turtle
ex:UnknownObject ex:mappedTo ex:Channel_03 .
```

`mappedTo`의 domain이 Variable이면, 이 트리플에서 UnknownObject가 Variable이라고 추론할 수 있습니다.

“UnknownObject에 Variable 타입이 명시되지 않았으므로 입력을 거절한다”는 뜻이 아닙니다.

domain/range만 설정하면 잘못된 배선이나 타입을 자동으로 거절한다는 이해는 위험합니다. 등록 조건 검사에는 SHACL을 사용합니다.

## 7. Open World: 기록이 없다고 부정은 아닙니다

Motor_02에 assetId가 없다고 OWL이 보통 “assetId가 존재하지 않는다”고 결론 내리지는 않습니다. 아직 모르는 정보일 수 있습니다.

OWL의 존재 제약도 “저장된 그래프에 구체적 값이 반드시 있는가”라는 검사와 다릅니다. 값이 명시되지 않았어도 존재한다고 해석할 수 있는 경우가 있습니다.

이 차이가 OWL과 SHACL을 함께 사용하는 이유입니다.

## 8. OWL의 주요 기능

| 기능 | 의미 | 초기 적용 시 주의 |
|---|---|---|
| subclass | 클래스의 상하 관계 | 분류와 물리 부품 관계를 혼동하지 않음 |
| equivalentClass | 동등한 클래스 의미 | 이름이 비슷하다고 선언하지 않음 |
| disjointWith | 동시 소속을 허용하지 않는 클래스 | 실제로 배타적인지 검토 |
| inverseOf | 관계의 역방향 | 의도한 의미가 성립하는지 확인 |
| TransitiveProperty | 관계의 추이성 | connectedTo를 무작정 추이적으로 선언하지 않음 |
| Restriction | 관계·값에 의한 클래스 조건 | 등록 검사와 동일시하지 않음 |
| sameAs | 동일한 개체 | 약한 연관이 아니라 강한 동일성 |

### connectedTo를 추이적으로 만들면 안 되는 예

채널이 접촉기에 연결되고 접촉기가 모터를 구동한다고, 채널이 모터에 직접 배선됐다는 뜻은 아닙니다.

“경로가 있다”와 “직접 연결됐다”는 다릅니다. 필요하면 SPARQL로 경로를 질의합니다.

### sameAs와 ID

같은 제품 모델의 모터 두 대를 `owl:sameAs`로 연결하면 안 됩니다. 두 모터의 사실을 동일한 개체의 사실로 취급하게 되기 때문입니다.

또한 OWL은 원칙적으로 서로 다른 이름이 반드시 서로 다른 개체를 가리킨다고 가정하지 않습니다. 차이를 논리적으로 다뤄야 한다면 서로 다른 개체임을 명시하는 설계를 검토합니다.

## 9. 선택 실습: SPARQL로 경로 찾기

아래 질문은 앞서 명시한 트리플로 답할 수 있습니다. OWL reasoner가 필요하지 않습니다.

```sparql
PREFIX ex: <https://example.org/plant/>

SELECT ?variable ?channel ?contactor
WHERE {
  ?variable ex:mappedTo ?channel .
  ?channel ex:connectedTo ?contactor .
  ?contactor ex:actuates ex:Motor_01 .
}
```

기대 결과:

```text
StartCommand | Channel_03 | Contactor_K1
```

### Python으로 읽는 경우

새로운 로컬 실습 폴더에서 앞의 Turtle을 `model.ttl`로, 다음 코드를 `query.py`로 저장합니다.

```python
from rdflib import Graph

graph = Graph().parse("model.ttl", format="turtle")
query = """
PREFIX ex: <https://example.org/plant/>
SELECT ?variable ?channel ?contactor
WHERE {
  ?variable ex:mappedTo ?channel .
  ?channel ex:connectedTo ?contactor .
  ?contactor ex:actuates ex:Motor_01 .
}
"""
rows = list(graph.query(query))
assert len(rows) == 1, f"Expected one path, got {len(rows)}"
for row in rows:
    print(*(str(value) for value in row), sep=" | ")
```

실습 폴더에서 실행합니다.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install rdflib
.\.venv\Scripts\python.exe query.py
```

RDFLib로 읽기만 한다고 전체 OWL DL 추론이 자동 실행되지는 않습니다. 추론 실습에는 별도 reasoner를 사용합니다.

## 10. 선택 실습: Protégé로 추론 보기

1. `model.ttl`을 엽니다.
2. Classes 화면에서 Motor의 상위 클래스 Device를 확인합니다.
3. Individuals 화면에서 Motor_01의 명시적 타입 Motor를 확인합니다.
4. 사용 가능한 호환 reasoner로 추론을 시작합니다.
5. 추론 결과에서 Motor_01이 Device에도 속하는지 확인합니다.
6. reasoner를 멈추고 명시적 기록과 추론 결과를 구분합니다.

실패하면 파싱 오류·미지원 기능·논리 모순을 구분해 확인합니다.

## 11. OWL 기능을 어디까지 사용할 것인가?

OWL 2 DL은 논리적 제약을 지키는 표현 범위, OWL 2 Full은 더 자유로운 RDF 조합을 허용하는 의미론으로 구분됩니다. 처음부터 모두 암기할 필요는 없습니다.

EL/QL/RL 등의 profile도 있지만, 우선 필요한 추론과 사용할 엔진이 대응하는지 확인합니다. Triple Store를 설치하면 모든 OWL 공리가 자동 추론되는 것은 아닙니다.

초기에는 클래스 계층·타입·명시적 관계·제한적 추론부터 시작하고, 추론이 실제 질문의 답을 개선하는지 측정합니다.

## 복습 문제와 해답

**Q1. Motor는 Device의 부품입니까?**  
`subClassOf`는 분류이며 부품 관계가 아닙니다.

**Q2. 배선이 기록되지 않은 채널은 미연결이라고 확정할 수 있습니까?**  
아닙니다. 미기록과 미연결을 구분하고, 근거 있는 상태를 별도로 기록합니다.

**Q3. 추론 결과와 원래 근거를 섞어 저장해도 됩니까?**  
출처를 잃지 않도록 명시적 사실·추론 결과·규칙/모델 버전을 구분합니다.

## 공식 문서 읽기 순서

[OWL 2 Primer](https://www.w3.org/TR/owl2-primer/)에서 Basic Notions → Classes, Properties, and Individuals → Advanced Class Relationships → Advanced Use of Properties 순으로 읽습니다.

이 자료의 PLC 모델과 원문의 예를 대응시키십시오. 문법 전체를 암기하기보다 무엇이 논리적으로 도출되는지 설명하는 것이 중요합니다.

다음: [SHACL과 종합 실습](05-shacl-integration.md).
