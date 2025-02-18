## 카카오 지도 API

장소를 저장하고, 저장한 장소를 지도로 보여주는 ODIT 프로젝트를 진행하는 과정에서 지도 API를 사용해야 했다.

![](https://blog.kakaocdn.net/dn/vSr5I/btsMljMZbuM/wTAJXUxiQgMKdFFz5ZQjT0/img.png)

지도 API는 카카오 지도 API를 사용했는데, 공식 문서에서는 특정 라이브러리를 사용하지 않고 순수 JavaScript 코드를 사용한 예시 코드만 존재했기 때문에 리액트에 맞게 어느 정도 코드 변환이 필요했다.

이번 글에서는 리액트에서 카카오 지도 API를 사용하는 방법과, 지도 API 관련 로직들을 hook으로 묶어 따로 처리한 방버에 대해 작성해 보겠다.

## 리액트에서 카카오 지도 API 사용하기

### 지도 API 등록하기

카카오 지도 API를 사용하기 위해서는 카카오 개발자 사이트에서 애플리케이션을 등록하고, 앱 키를 발급받아야 한다.

[https://developers.kakao.com](https://developers.kakao.com/)

해당 페이지에서 애플리케이션을 등록하고, 도메인을 추가하면 앱 키를 발급받을 수 있다.

앱 키에서 JavaScript 키를 가져와 프로젝트의 환경변수로 등록해준다.

```jsx
REACT_APP_KAKAO_APP_KEY = '발급받은 앱 키';
```

그리고 index.html 파일에 API를 불러올 script 태그를 넣어준다.

```jsx
<body>
  <script src="//dapi.kakao.com/v2/maps/sdk.js?appkey=%REACT_APP_KAKAO_APP_KEY%&libraries=services"></script>
  <div id="root"></div>
</body>
```

script 태그 위치는 body에 넣었지만, 문서에서는 head나 body 어디에 넣어도 상관없지만, **반드시 실행 코드보다 먼저 선언되어야 한다**고 나와있다.

해당 프로젝트는 지도 검색 기능이 있기 때문에 파라미터에 "libraries=services"도 추가해주어야 한다.

### 지도 생성하기

https://apis.map.kakao.com/web/sample/basicMap/

공식 사이트에서 제공하는 예시와 같이 지도를 생성할 수 있다.

```jsx
<div id="map" style="width:100%;height:350px;"></div><script>
var mapContainer = document.getElementById('map'),// 지도를 표시할 div
    mapOption = {
        center: new kakao.maps.LatLng(33.450701, 126.570667),// 지도의 중심좌표level: 3// 지도의 확대 레벨
    };

// 지도를 표시할 div와  지도 옵션으로  지도를 생성합니다var map = new kakao.maps.Map(mapContainer, mapOption);
</script>
```

div 태그의 id를 "map"으로 선언해 map 태그를 생성한다. 그리고 자바스크립트 코드에서 map 태그의 요소를 가져와 kakao.maps.Map 생성자를 통해 지도 객체를 생성한다.

여기까지 진행하면 자바스크립트를 통해 지도를 생성할 수 있다.

위 코드를 리액트에서 사용 가능하게 변환하려면 useRef를 사용해서 지도를 생성할 수 있다.

```jsx
import React from 'react';
import styled from '@emotion/styled/macro';

const Map = () => {
  const mapContainerRef = React.useRef(null);
  const mapRef = React.useRef(null);

  const { isLoding: isCurrentLocationLoading, currentLocation } = useCurrentLocation();

  React.useEffect(() => {
    if (isCurrentLocationLoading || !currentLocation || !mapContainerRef.current) {
      return;
    }

    const options = {
      center: new window.kakao.maps.LatLng(currentLocation.latitude, currentLocation.longitude),
      level: 3,
    };

    mapRef.current = new window.kakao.maps.Map(mapContainerRef.current, options);
  }, [isCurrentLocationLoading, currentLocation]);

  return (
    <>
      <StyledMap ref={mapContainerRef} />
      {/* some components */}
    </>
  );
};

export default Map;

const StyledMap = styled.div`
  position: fixed;
  inset: 0;
  width: 100%;
  height: 100%;
`;
```

map 태그의 요소를 document.getElementById 함수를 통해 가져오는 대신에 useRef를 사용해서 mapContainerRef 이름으로 저장한다. 그리고 useEffect를 사용해 kakao.maps.Map 생성자에서 첫 번째 인자에 mapContainerRef 요소를 전달해 준다.

script 태그를 사용해 API를 로드하면 window 객체에 등록된다. 보통 자바스크립트에서 window는 생략 가능하지만, 리액트서는 ES6 모듈 시스템과 동적 로딩 때문에 kakao가 자동으로 전역 변수로 등록되지 않을 수 있어 window를 붙여 사용해 준다.

useCurrentLocation은 navigator.gelocation.getCerrentPostion 함수를 통해 현재 위경도 값을 가져오고, 지도를 생성할 때 처음에 보이는 위치로 사용된다.

생성자가 반환한 요소도 useRef를 통해 mapRef 이름으로 저장한다.

### 키워드 검색

https://apis.map.kakao.com/web/sample/keywordBasic/

키워드 기반으로 검색 결과를 받아오는 코드는 공식 문서에서 다음과 같이 나와있다.

```jsx
// 장소 검색 객체를 생성합니다var ps = new kakao.maps.services.Places();

// 키워드로 장소를 검색합니다
ps.keywordSearch('이태원 맛집', placesSearchCB);

// 키워드 검색 완료 시 호출되는 콜백함수 입니다function placesSearchCB (data, status, pagination) {
    if (status === kakao.maps.services.Status.OK) {
      console.log(data);
    }
}
```

키워드로 장소를 검색하는 기능을 사용하기 위해서는 kakao.maps.services.Places 생성자를 통해 장소 검색 객체를 생성하고, 해당 객체의 keywordSearch 함수를 통해 검색 결과를 받아올 수 있다.

첫 번째 인자로 검색할 문자열 keyword를 전달해 주고, 검색 결과를 받을 콜백 함수를 전달해서 카카오 지도의 키워드 검색 기능을 사용할 수 있다.

키워드 검색 기능도 리액트에서 사용하기 위해서는 다음과 같이 코드를 작성하면 된다.

```jsx
import React from 'react';

const KakaoPlacesSearch = () => {
  const [keyword, setKeyword] = React.useState();
  const [result, setResult] = React.useState();

  const keywordSearch = React.useRef();

  React.useEffect(() => {
    const ps = new window.kakao.maps.services.Places();

    keywordSearch.current = ps.keywordSearch;
  }, []);

  React.useEffect(() => {
    if (!keywordSearch.current) {
      return;
    }

    keywordSearch.current(keyword, (data, status) => {
      if (status === kakao.maps.services.Status.OK) {
        setResult(data);
      }
    });
  }, [keyword]);

  return {
    /* some components */
  };
};

export default KakaoPlacesSearch;
```

useRef를 사용해서 kaka.maps.services.Places의 keywordSearch 함수를 ref로 저장한다. 그리고 keyword가 변경되면 useEffect를 통해 검색 결과를 받아올 콜백 함수를 전달해 준다.

## 카카오 지도 API 로직 분리하기

카카오 지도 API 코드를 위와 같이 작성하면 리액트에서 문제없이 사용이 가능하다. 그러나 프로젝트에서 지도를 사용하는 코드가 단순히 지도만 보여주는 것이 아니라 커스텀 마커를 추거 하거나, 다른 로직들도 추가되어야 하는 경우가 많다. 이런 경우 **Map 파일의 코드가 길어지고, 로직이 분리되지 않아 복잡한 코드**가 만들어질 수 있다.

그래서 프로젝트에서 필요한 로직과 카카오 지도 API 로직을 분리하기 위해 **카카오 지도 API 코드를 hook으로 분리**하기로 했다.

### 지도 생성하기 hook

```jsx
import React from 'react';
import useCurrentLocation from './useCurrentLocation';
import usePlaces from './usePlaces';

const useKakaomap = ({ mapContainerRef }) => {
  const mapRef = React.useRef(null);

  const { isLoding: isCurrentLocationLoading, currentLocation } = useCurrentLocation();

  React.useEffect(() => {
    if (isCurrentLocationLoading || !currentLocation || !mapContainerRef.current) {
      return;
    }

    const options = {
      center: new window.kakao.maps.LatLng(currentLocation.latitude, currentLocation.longitude),
      level: 3,
    };

    mapRef.current = new window.kakao.maps.Map(mapContainerRef.current, options);
  }, [isCurrentLocationLoading, currentLocation, mapContainerRef.current]);

  return { mapRef };
};

export default useKakaomap;
```

기존에는 Map 컴포넌트에서 뷰를  나타내는 코드와 카카오 지도 API를 사용하는 코드가 같이 존재했지만, 카카오 지도 API 코드를 useKakaoMap hook으로 분리해서 카카오 지도 API 로직을 분리할 수 있다.

### 지도 생성 hook

```jsx
import React from 'react';
import styled from '@emotion/styled/macro';
import useKakaoMap from './useKakaoMap';

const Map = () => {
  const mapContainerRef = React.useRef(null);

  const { mapRef } = useKakaoMap(mapContainerRef);

  return (
    <>
      <StyledMap ref={mapContainerRef} />
      {/* some components */}
    </>
  );
};

export default Map;

const StyledMap = styled.div`
  position: fixed;
  inset: 0;
  width: 100%;
  height: 100%;
`;
```

map 태그의 ref은 mapContainerRef만 hook의 props로 전달해서 지도를 생성하기 때문에 기존의 코드보다 훨씬 간결해진 거를 볼 수 있다!!

이렇게 코드를 분리해서 지도와 관련된 로직은 useKakaoMap hook에서, 뷰와 관련된 코드는 Map 컴포넌트에서 관리해 주어 로직에 따라 파일을 분리할 수 있다.

useKakaomap hook에서 카카오 지도 API에 접근할 수 있는 요소인 mapRef를 반환하게 하여 부득이하게 hook 외부에서 지도에 대한 요소를 조작하게도 가능하게 해 주었다.

### 키워드 검색 hook

```jsx
import React from 'react';

const useKakaoKeywordSearch = async (keyword) => {
  const [result, setResult] = React.useState();

  const keywordSearchRef = useRef();

  React.useEffect(() => {
    const ps = new window.kakao.maps.services.Places();

    keywordSearchRef.current = ps.keywordSearch;
  }, []);

  React.useEffect(() => {
    keywordSearchRef.current(keyword, (status, data) => {
      if (status === window.kakao.maps.services.Status.OK) {
        setResult(data);
      }
    });
  }, [keyword]);

  return result;
};

export default searchKakaoPlaces;
```

기존 KakaoSearchPlaces 컴포넌트에서 키워드 검색 로직을 hook으로 분리했다. kakao.maps.services.Places 생성자를 통해 장소 검색 객체를 생성하고, useEffect를 사용해 keyword가 변경될 때마다 keywordSearch 함수를 실행시켜 결과를 state로 저장한 이후 반환한다.

```jsx
import React from 'react';
import useKakaoKeywordSearch from './useKakaoKeywordSearch';

const KakaoPlacesSearch = () => {
  const [keyword, setKeyword] = React.useState();

  const result = useKakaoKeywordSearch(keyword);

  return {
    /* some components */
  };
};

export default KakaoPlacesSearch;
```

키워드 검색 로직을 hook으로 분리한 결과 기존 KakaoPlacesSearch 컴포넌트의 코드를 Map 컴포넌트와 같이 간결하게 변경할 수 있게 되면서 뷰에 집중할 수 있는 컴포넌트를 만들 수 있다.

## 결론 : 커스텀 hook을 애용하자

요새 개발하면서 가장 크게 하고 있는 고민은 가독성 높은 코드를 작성하는 것이다. 가독성 높은 코드를 만드는 방법은 여러 가지가 있지만, 그중 가장 중요한 거는 **로직별로 코드를 모으는 것이라** 생각한다.

기존에는 커스텀 hook을 사용하지 않고 항상 모든 로직을 컴포넌트에 다 작성했지만, 최근에 데이터의 처리가 복잡한 코드들은 커스텀 hook을 적용해 보기 위해 노력하고 있다.

그중 카카오 지도 API를 사용하면서 컴포넌트에 지도를 생성하는 코드와 뷰를 처리하는 코드가 같이 혼합돼있는 걸 발견하고, 지도 API 로직을 커스텀 hook으로 분리해서 적용해 보았다.

커스텀 hook을 적용한 결과가 컴포넌트에서 로직을 분리하여 지도 API와 관련된 기능을 추가해도 기존 컴포넌트에는 코드가 추가되지 않아 훨씬 가독성 높은 코드가 될 수 있다는 거를 느낄 수 있었다.
