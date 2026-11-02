# 3. Eclipse BaSyx: AAS를 저장하고 서비스하는 방법

## 학습 목표

- 표준과 구현을 구분한다.
- Repository·Registry·Discovery·UI의 책임을 설명한다.
- 공식 Starter Kit로 조회 흐름을 확인하고 실패를 해석한다.

## 1. BaSyx를 한 문장으로

Eclipse BaSyx는 AAS를 포함한 Industrie 4.0 관련 기능을 구현하는 오픈소스 구성 요소와 SDK입니다.

비유하면 AAS 사양은 “어떤 계약을 따를지”, BaSyx는 “그 계약을 구현한 프로그램”입니다. 따라서 사양을 읽는 일과 특정 BaSyx 배포판을 운영하는 일은 구분해야 합니다.

## 2. 주요 역할

```text
사용자 / 애플리케이션
       │
       ├─ Registry에서 서비스 위치 확인
       │             │
       │             ▼
       ├─ AAS Repository에서 AAS 조회
       │             │ 서브모델 참조
       │             ▼
       └─ Submodel Repository에서 속성 조회
```

| 역할 | 담당하는 것 | 오해하지 말 것 |
|---|---|---|
| AAS Repository | AAS 데이터 제공·관리 | 모든 PDF·시계열을 자동 관리하지 않음 |
| Submodel Repository | 서브모델과 요소 제공·관리 | AAS 조회만으로 전체 본문이 반환된다고 가정하지 않음 |
| AAS/Submodel Registry | descriptor와 endpoint 등 서비스 위치 정보 | 원본 모델 전체를 저장하는 원장과 다름 |
| Discovery 역할 | 자산 식별 정보에서 AAS 식별 정보 등을 찾기 | Registry와 모든 기능이 동일하지 않음 |
| AAS Web UI | 모델 탐색·표시·지원되는 업로드/수정 | UI에서 보인다고 의미·사실 검증 완료는 아님 |
| SDK | 모델 생성·읽기·서비스 연계 개발 | AI 승인이나 RDF 도메인 매핑을 자동 완성하지 않음 |

선택한 배포판은 일부 역할을 한 프로세스나 AAS Environment 구성 요소로 제공할 수 있습니다. 위 표는 논리적 역할이며 무조건 컨테이너 수를 뜻하지 않습니다.

## 3. Registry를 주소록으로 이해하기

실제 데이터가 다른 서버로 이동했다고 가정하십시오.

- 자산 ID는 그대로.
- AAS ID도 정책에 따라 그대로.
- 서비스를 호출할 endpoint는 변경.
- Registry descriptor의 endpoint는 갱신 필요.

Registry가 오래된 주소를 제공하면 모델은 저장소에 존재해도 사용자가 못 찾을 수 있습니다.

“등록됐다”는 말도 구분해야 합니다.

1. 모델이 저장소에 생성됐는가?
2. descriptor가 Registry에 등록됐는가?
3. 참조하는 서브모델을 실제 조회할 수 있는가?
4. 사용자에게 그 정보의 조회 권한이 있는가?

## 4. 설치 전 버전 확인표

| 확인할 항목 | 확인 방법 |
|---|---|
| 구성 요소 세대 | BaSyx V1/V2 등 문서와 이미지 일치 |
| 지원 AAS 메타모델 | 구성 요소의 release/지원 문서 |
| API와 기능 범위 | 해당 서버 OpenAPI·공식 문서 |
| UI/SDK 호환성 | 동일 예제·release 조합으로 시작 |
| 저장 backend | 메모리인지 영속 저장인지 확인 |
| 인증 방식 | 개발 설정과 운영 설정 구분 |

여러 문서에서 복사한 `latest` 이미지 조합보다 공식 예제의 호환 조합을 출발점으로 사용합니다. 검증한 이미지 태그·설정·문서 판본을 고정하십시오.

## 5. 선택 실습: 공식 Starter Kit

이 실습은 로컬 개발 환경만 대상으로 합니다. 운영 PLC나 비공개 장치 정보를 연결하지 않습니다.

### 준비

- Docker와 Docker Compose를 사용할 수 있는 환경.
- 공식 [Quick Start Guide](https://wiki.basyx.org/en/latest/content/introduction/quickstart.html).
- 공식 Starter Kit에서 생성·다운로드한 설정.

공식 Quick Start는 BaSyx V2의 Docker 구성 요소와 Starter Kit를 안내합니다. 생성된 설정 파일을 풀고 **그 폴더에서** 다음 명령을 실행합니다.

```powershell
docker compose up -d
docker compose ps
```

공식 기본 안내의 UI 주소는 `http://localhost:3000`입니다. 설정에서 포트를 바꿨다면 생성된 설정을 따르십시오.

컨테이너 상태만 확인하지 말고 UI가 열리는지와 모델이 조회되는지를 확인합니다.

### 첫 번째 실행에서는 공식 예제를 사용

1. Quick Start에 연결된 공식 example shell을 먼저 사용합니다.
2. UI의 지원 업로드 기능으로 모델을 가져옵니다.
3. AAS ID·자산 ID·Submodel·Property를 차례로 확인합니다.
4. 공식 예제의 구조를 이해한 뒤 [메타모델 장](02-aas-metamodel.md)의 교육용 JSON으로 확장합니다.
5. UI가 JSON/AASX 중 어떤 형식을 지원하는지 확인합니다. 교육용 JSON을 모든 UI가 그대로 받는다고 가정하지 않습니다.

오류가 나면 JSON 전체를 임의로 바꾸기보다 첫 오류 경로와 서버 지원 버전을 먼저 확인하십시오.

### 수명주기 확인

- 저장한 모델을 다시 조회합니다.
- 로컬 구성 요소를 재시작한 후 데이터가 남는지 확인합니다.
- 메모리 저장 설정이라면 재시작 후 소실될 수 있습니다. 영속성을 확인하기 전에는 원장으로 사용하지 않습니다.
- 생성한 로컬 실습 서비스를 중지하려면 같은 설정 폴더에서 `docker compose stop`을 사용할 수 있습니다.

이 자료는 임의의 Docker Compose 설정을 제공하지 않습니다. 공식 Starter Kit의 현재 호환 설정을 쓰고, 동작 확인 후 필요한 부분만 변경하는 것이 목적입니다.

## 6. API 탐색 실습

선택한 서버의 OpenAPI 또는 공식 API 문서를 열고 다음을 찾습니다.

| 찾을 기능 | 기대 확인 |
|---|---|
| AAS 목록 조회 | 반환 구조·페이지 처리·권한 |
| 특정 AAS 조회 | 식별자 경로 인코딩 |
| Submodel 조회 | 참조된 모델의 실제 본문 |
| Property 값 조회 | 값 표현과 자료형 |
| Registry 조회 | endpoint descriptor |

일부 AAS API의 경로 식별자는 Base64URL 같은 인코딩을 사용합니다. ID 원문을 URL에 그대로 붙이면 안 될 수 있습니다. 경로·인코딩·기본 URL은 선택한 API 판본과 서버 문서에서 확인합니다.

### 관찰 기록

```text
조회한 AAS ID:
연결된 Submodel ID:
호출한 서비스 역할:
HTTP 결과:
실제 받은 값:
Registry endpoint와 실제 주소가 일치하는가:
지원 API/메타모델 판본:
```

## 7. 장애를 구분하는 연습

| 현상 | 우선 확인 | 하면 안 되는 처리 |
|---|---|---|
| UI가 안 열림 | 포트·컨테이너·UI 설정 | 데이터가 없다고 단정 |
| AAS가 조회되지 않음 | ID 인코딩·권한·대상 저장소 | 빈 AAS를 생성하여 성공으로 대체 |
| 서브모델만 실패 | 참조·Registry·endpoint·권한 | 속성을 미확인 없이 0으로 표시 |
| 재시작 후 소실 | 저장 backend·영속 설정 | 저장 완료라고 운영에 보고 |
| 업로드 실패 | 형식·모델 판본·제약 오류 | 모든 검증을 꺼서 통과 |

## 8. 기존 아키텍처에 배치하기

```text
HW 자료 → AI 추출 후보 → 검증·승인
                           │
                           ▼
                         BaSyx
                           │ 승인 버전
                           ├─ RDF Projection → Triple Store
                           └─ 등록/변경 처리 → 매뉴얼 수집
```

BaSyx를 설치했다고 AI 추출·출처 관리·승인·RDF 매핑이 자동으로 생기지 않습니다. 각각 애플리케이션 계층에서 설계합니다.

시계열은 별도 historian/DB에 보관하고 AAS/그래프의 signal ID·asset ID와 연결합니다. OPC UA나 MC 수집기의 성능 문제를 AAS 서버 설치만으로 해결할 수는 없습니다.

## 확인 문제와 해답

**Q1. Registry를 백업하면 자산 데이터도 모두 복구됩니까?**  
아닙니다. Registry 정보와 실제 저장소 원문·설정·영속 데이터는 별도로 확인해야 합니다.

**Q2. API가 200을 반환했으면 정격값은 정확합니까?**  
아닙니다. HTTP 처리 성공과 사양의 사실성은 다릅니다.

**Q3. UI에 없는 기능은 표준에도 없습니까?**  
아닙니다. 표준 범위와 해당 UI/서버 구현 범위는 다릅니다.

## 공식 문서 읽기

- [BaSyx 문서 홈](https://wiki.basyx.org/en/latest/): 문서 지도를 확인.
- [Quick Start](https://wiki.basyx.org/en/latest/content/introduction/quickstart.html): 공식 구성으로 시작.
- [Components](https://wiki.basyx.org/en/latest/content/user_documentation/basyx_components/index.html): 사용하는 세대의 구성 요소를 선택.

다음: [OWL 온톨로지](04-owl.md).
