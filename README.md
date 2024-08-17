# 프론트엔드와 백엔드의 성능 병목점 파악 및 이에 대한 최적화 수행

## 문제 정의: 애플리케이션의 성능 병목점 파악 및 목표 설정
현재 팀프로젝트에서 공공 데이터 포탈의 API와 카카오맵 API, 구글맵 API 등 외부 API를 호출하여 홈화면에 날씨 정보와 주변 맛집 정보를 표시하고 있습니다.

해당 서버를 실행하고 홈페이지를 처음 켰을 때 날씨와 주변 맛집을 나타내주는 화면이 15초 정도 지나고 나서 표시가 되는 것을 확인했습니다.

현재로써는 외부 API 호출로 인해 발생하는 지연이 문제의 원인일 가능성이라는 생각이 들어 네트워크 지연, API 응답 시간, 데이터 처리 등의 요인을 확인해봐야할 것 같습니다.

이에 대한 성능을 개선하는 것을 목표로 합니다.
## 솔루션 도출: 최적화 전략 수립 및 도구 선택
- 비동기 로딩 및 병렬 처리
  - 비동기 호출 (Async/Await): 날씨 정보와 주변 맛집 정보를 동시에 비동기로 호출
  - 병렬 처리: 날씨와 주변 맛집 API 호출을 병렬로 처리
- API 응답 캐싱
  - 서버 캐싱: 외부 API에서 받은 데이터를 일정 시간 동안 서버 쪽에서 캐싱
  - 브라우저 캐싱: 캐시 가능한 데이터를 클라이언트 쪽에서 캐싱
- 프리로딩 및 프리페칭: 사용자가 웹사이트에 접속하기 전에 날씨 정보와 맛집 데이터를 미리 로드하거나 프리페칭
## 설계: 최적화 작업을 위한 계획 수립 및 설계
- 비동기 API 호출 구조 설계
  - 현재 프론트엔드에서 리액트 JavaScript를 사용하기 때문에 JavaScript의 async/await을 사용하여 비동기적으로 API를 호출하고, Promise.all()을 활용하여 병렬 처리
- 캐싱 전략 설계
  - 서버에서 데이터를 캐싱할 때  날씨 정보는 자주 변경되지 않으므로 10~15분 동안 캐시하는 등의 캐싱 전략 수립
  - Redis와 같은 캐시 서버를 사용
- 로딩 상태 설계
  - 로딩 중임을 사용자에게 알려주는 UI 요소를 설계
- API 요청 간소화
  - 구글맵 API에서 반환되는 데이터 중 필요한 데이터만 요청하도록 쿼리 파라미터를 조정 등 필요 이상의 데이터를 요청하지 않도록 API 요청을 간소화
## Chrome DevTools를 사용하여 프론트엔드 성능 분석
### Network 패널 분석(API 호출에 걸리는 시간 확인)

<img width="718" alt="image" src="https://github.com/user-attachments/assets/d3d3fa12-aa0c-4de6-816e-fef27f3f8ad3">

API를 호출하는 부분의 소요 시간이며 가장 오래 걸리는 작업들입니다.

### 성능 탭 확인

<img width="613" alt="image" src="https://github.com/user-attachments/assets/d9b359c1-2393-4bd3-a593-faf35674aef0">

#### 분석 단계

##### 1. 타임라인 분석
   - 상단에 표시된 네트워크 영역을 보면, restaurant, weather과 같은 외부 API 호출의 네트워크 요청들이 발생한 시점과 시간을 확인할 수 있습니다.
   - restaurant와 weather은 병렬적으로 호출되는 것을 확인할 수 있고 첫 번째 줄의 restaurant가 호출되고 페이지에 랜더링되고 난 후 한 번 더 호출되는 것이 확인되고 weather 또한 2, 3번째 줄의 호출 후 랜더링 되고나서 각각 한 번씩 더 호출되는 것을 확인할 수 있습니다. 이 부분에서도 문제가 발생한 것으로 추측됩니다.
##### 2. 네트워크 요청 분석 (다 비슷한 외부 API GET 요청이기 때문에 첫 번쨰 줄의 Restaurant만 분석해보겠습니다)

<img width="694" alt="image" src="https://github.com/user-attachments/assets/83515a76-8bd2-4df2-aa36-573e7903178c">

1. 네트워크 요청
- URL: `restaurant` API 요청을 확인할 수 있습니다. 이 요청은 사용자가 주변 맛집 정보를 얻기 위해 호출했습니다!
- 소요 시간: 
  - 3.044초동안 네트워크 요청 및 응답이 처리되었습니다.
  - 네트워크 전송 시간이 3.044초 중 대부분을 차지하며, 리소스 로드에는 950마이크로초(µs)가 소요되었습니다. 이는 응답이 빠르지 않다는 것을 의미합니다.
- 요청 메소드: `GET` 요청을 사용하고 있습니다.
- MIME 유형: 요청된 데이터는 `application/json` 형태로, 주변 맛집 정보를 JSON 데이터를 반환하는 API 호출입니다.

2. 문제 분석
- 네트워크 요청의 전송 시간이 3.044초로 오래 걸렸습니다. 이로 인해 API 서버의 응답 시간이 느리다는 것을 확인할 수 있었습니다.
- 해결 방안
  - API 응답 최적화: 백엔드 서버의 성능 병목을 확인하고, 필요한 경우 서버 성능을 개선하거나 요청 데이터를 캐시하여야 할 것 같습니다.
  - 캐싱 사용: API의 응답이 자주 변하지 않기 때문에 Redis와 같은 캐시 서버를 사용을 고려해봐야할 것 같습니다.

3. 로드와 유휴 시간
- 페이지 로드 후 많은 유휴 시간(Idle)이 발생하고 있고 이는 페이지가 로딩되는 동안 일부 작업이 완료되지 않아서 기다려야 하는 상태로 생각됩니다.
- 해결 방안
  - 로딩 애니메이션: 데이터가 로드되는 동안 사용자에게 로딩 애니메이션을 제공을 고려해봐야 할 것 같습니다.
  - 비동기 데이터 로딩: 페이지가 초기 로드될 때 모든 데이터를 불러오지 않고, 비동기로 데이터를 받아오는 방법도 생각해봐야할 것 같습니다.

## Lighthouse를 사용하여 웹 페이지 성능 점수 확인
### SCORE

<img width="688" alt="image" src="https://github.com/user-attachments/assets/981f3610-68e2-42a8-8f68-7267c908349b">

#### 성능

<img width="688" alt="image" src="https://github.com/user-attachments/assets/e3fc5387-3bd7-41d9-8100-c4290dc926a9">

##### 진단

<img width="688" alt="image" src="https://github.com/user-attachments/assets/a0ad2f46-7380-40c1-9c83-e229c8970b06">

<img width="688" alt="image" src="https://github.com/user-attachments/assets/9b805364-febc-40a8-8abb-0d0b186f8b2f">

##### 통과한 감사

<img width="694" alt="image" src="https://github.com/user-attachments/assets/07a0a519-e248-4268-941b-1a10731522fc">

<img width="694" alt="image" src="https://github.com/user-attachments/assets/a411b18b-5f60-4cfa-a82d-4dfb7f62c1f7">

#### 접근성
<img width="688" alt="image" src="https://github.com/user-attachments/assets/c74cfcc2-37ec-4d9f-a758-f93113858e3b">

##### 통과한 감사

<img width="688" alt="image" src="https://github.com/user-attachments/assets/c74e2662-86b8-48fe-84e6-748d9f41bf65">

#### 권장 사항
<img width="688" alt="image" src="https://github.com/user-attachments/assets/749783ab-2a1e-4d41-90ff-b77de2aee87e">

##### 통과한 감사

<img width="688" alt="image" src="https://github.com/user-attachments/assets/28f3e421-0446-4fde-b1e6-ecedc44c455e">

#### 검색엔진 최적화
<img width="688" alt="image" src="https://github.com/user-attachments/assets/7c12831e-58f2-49a7-943a-31fc86bd2e9d">

##### 통과한 감사
<img width="688" alt="image" src="https://github.com/user-attachments/assets/e33d0dfe-c45a-48f6-a53e-b497f826034f">

프론트엔드 성능 문제 식별 및 우선순위 설정

프론트엔드 코드 최적화 (예: Lazy Loading, Code Splitting)

이미지 및 자원 최적화 (예: 이미지 압축, 파일 Minification)

프론트엔드 성능 개선 후 성능 재측정

Prometheus를 사용하여 백엔드 성능 분석

Grafana를 사용하여 모니터링 대시보드 구성

데이터베이스 쿼리 최적화 및 인덱싱

캐싱 전략 도입 (예: Redis 사용)

백엔드 비동기 처리 및 로드 밸런싱 구현

API 응답 시간 단축을 위한 코드 최적화

백엔드 성능 개선 후 성능 재측정

성능 개선 전후 비교 결과 작성
