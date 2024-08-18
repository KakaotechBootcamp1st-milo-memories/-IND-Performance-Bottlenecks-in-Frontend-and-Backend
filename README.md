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

### 분석
현재 권장사항 항목을 제외하면 전체적으로 높은 점수를 보이고 있습니다.

이러한 현상은 외부 API를 호출하는 시간이 점수에 포함되지 않아서 그렇다고 추측하고 있습니다.

권장사항 항목은 아직 HTTPS와 위치 정보 권한을 요청하는 것 때문에 점수가 떨어진 것 같습니다.


성능에서 주의를 준 부분은 다음과 같습니다.
- 리소스가 페이지의 첫 페인트를 차단하고 있습니다. 중요한 JS/CSS를 인라인으로 전달하고 중요하지 않은 모든 JS/Style을 지연하는 것이 좋습니다. -> 절감 가능치: 210 밀리초
  <img width="666" alt="image" src="https://github.com/user-attachments/assets/301164aa-6126-430e-8993-97d6dd94a0a8">

- 자바스크립트 파일을 축소하면 페이로드 크기와 스크립트 파싱 시간을 줄일 수 있습니다. -> 절감 가능치: 205KiB
  <img width="666" alt="image" src="https://github.com/user-attachments/assets/5be38d2d-46e0-4c07-a6b3-67f5c68cc436">

- 사용되지 않는 자바스크립트를 줄이고 스크립트가 필요할 때까지 로딩을 지연시켜 네트워크 활동에 소비되는 용량을 줄이세요 -> 절감 가능치: 245KiB
  <img width="666" alt="image" src="https://github.com/user-attachments/assets/ac99c410-3f42-4b77-801b-a4788a1f6302">

- 대부분의 탐색은 이전 페이지로 이동하거나 다시 원래 페이지로 돌아오는 방식으로 이루어집니다. 뒤로-앞으로 캐시(bfcache)로 이러한 돌아가기 탐색의 속도를 높일 수 있습니다 -> 실패이유: WebSocket이 있는 페이지에서는 뒤로-앞으로 캐시를 시작할 수 없습니다.
  <img width="666" alt="image" src="https://github.com/user-attachments/assets/f89da353-c50c-4719-8dc9-b10b70ec2c6a">

## 프론트엔드 성능 문제 식별 및 우선순위 설정
- DevTools에서 분석한 문제 해결 방법은 다음과 같습니다.
  - API 응답 최적화: 백엔드 서버의 성능 병목을 확인하고, 필요한 경우 서버 성능을 개선하거나 요청 데이터를 캐시하여야 할 것 같습니다.
  - 캐싱 사용: API의 응답이 자주 변하지 않기 때문에 Redis와 같은 캐시 서버를 사용을 고려해봐야할 것 같습니다.
  - 로딩 애니메이션: 데이터가 로드되는 동안 사용자에게 로딩 애니메이션을 제공을 고려해봐야 할 것 같습니다.
  - 비동기 데이터 로딩: 페이지가 초기 로드될 때 모든 데이터를 불러오지 않고, 비동기로 데이터를 받아오는 방법도 생각해봐야할 것 같습니다.
- LightHouse에서 분석한 문제 해결 방법은 다음과 같습니다.
  - 랜더링 차단 리소스 제거
  - 자바스크립트 줄이기
  - 사용하지 않는 자바스크립트 줄이기
 
### 우선순위 설정
- DevTools에서 분석한 문제 해결 방법의 우선 순위를 설정해보면 다음과 같습니다.
1. 비동기 데이터 로딩 -> 프론트엔드에서 시도해볼 수 있는 방법이라서 최우선순위로 두었습니다.
2. API 응답 최적화 -> 백엔드 서버에서 할 수 있는 방법이라서 다음 우선순위로 두었습니다.
3. 캐싱 사용 -> 캐싱 서버를 따로 두어야해서 비교적 복잡한 방법이라 생각하여 3위로 우선순위로 두었습니다,
4. 로딩 애니메이션 -> 근본적인 문제 해결 방법은 아니라 생각하여 4위로 우선순위로 두었습니다.

- LightHouse에서 분석한 문제 해결 방법의 우선 순위를 설정해보면 다음과 같습니다.
1. 랜더링 차단 리소스 제거 -> 절감 가능치: 210 밀리초
2. 사용하지 않는 자바스크립트 줄이기 -> 절감 가능치: 245KiB
3. 자바스크립트 줄이기 -> 절감 가능치: 205KiB
- 단순히 절감 가능치 순서대로 우선순위를 두었습니다.

## 프론트엔드 코드 최적화 (예: Lazy Loading, Code Splitting)
### Weather 원본 코드
```
import React, { useEffect, useState } from 'react';
import axios from 'axios';
import "../styles/weather.css"

const Weather = () => {
    const [weather, setWeather] = useState(null);

    useEffect(() => {
        const fetchWeather = async (latitude, longitude) => {
            try {
                const response = await axios.get('/api/weather', {
                    params: {
                        nx: latitude,
                        ny: longitude
                    }
                });
                setWeather(response.data);
            } catch (error) {
                console.error('Error fetching weather data', error);
            }
        };

        const getLocation = () => {
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition((position) => {
                    const { latitude, longitude } = position.coords;
                    fetchWeather(latitude, longitude);
                }, (error) => {
                    console.error('Error getting location', error);
                });
            } else {
                console.error('Geolocation is not supported by this browser.');
            }
        };

        getLocation();
    }, []);

    if (!weather) {
        return <div>Loading...</div>;
    }

    return (
        <div className="weather-container">
            <div className="weather-card">
                <div className="weather-header">나의 위치</div>
                <div className="weather-content">
                    <div className="weather-details">
                        <div className="weather-city">{weather.city}</div>
                        <div className="weather-condition">대체로 맑음</div>
                    </div>
                    <div>
                        <div className="weather-temp">{weather.cur_temperature}°C</div>
                        <div className="weather-high-low">최고 {weather.high_temperature}° 최저 {weather.low_temperature}°</div>
                    </div>
                </div>
            </div>
        </div>
    );
};

export default Weather;
```

### Code Splitting 적용 - Weather Card 분리

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
