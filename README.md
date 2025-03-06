# 📚 UrLink

<div align="center">
  
  ![Group 1 (3)](https://github.com/user-attachments/assets/19540f6e-1863-4ad2-a809-05097272441c)

  URLink는 북마크 제목만으로는 파악하기 어려운 내부에 키워드를 검색하여 필요한 정보를 찾아주는 서비스입니다.
</div>


## 목차
- [1. 개발 동기](#1-개발-동기)
- [2. 기술 스택](#2-기술-스택)
- [3. 구현 세부사항](#3-구현-세부사항)
  - [3-1. 어떻게 북마크를 가져올까?](#3-1-어떻게-북마크를-가져올까)
  - [3-2. SPA, Iframe 해결](#3-2-spa-iframe-해결)
  - [3-3. 어떻게 키워드가 포함된 문장을 가져올까?](#3-3-어떻게-키워드가-포함된-문장을-가져올까)
  - [3-4. 너무 느린 크롤링 속도 어떻게 해결할까?](#3-4-너무-느린-크롤링-속도-어떻게-해결할까)
    - [3-4-1. 사용자 UI/UX를 통해 해결해보자.](#3-4-1-사용자-uiux를-통해-해결해보자)
    - [3-4-2. 로직 변경을 통해 크롤링 서버 OOM을 피해보자.](#3-4-2-로직-변경을-통해-크롤링-서버-oom을-피해보자)
  - [3-5. 크롤링으로 가져온 검색 결과를 한 번 더 검색할 순 없을까?](#3-5-크롤링으로-가져온-검색-결과를-한-번-더-검색할-순-없을까)
  - [3-5-1. 가져온 HTML을 어디에 저장할지의 문제](#3-5-1-가져온-html을-어디에-저장할지의-문제)
- [4. 의사결정 방식](#4-의사결정-방식)
- [5. 일정](#5-일정)
- [6. 팀원](#6-팀원)


## 1. 개발 동기
일상적으로 인터넷을 사용하다 보면, 저장해둔 북마크가 어느새 쌓여가는 경험을 하게 됩니다. 그러다 보니 북마크의 제목만으로는 실제 페이지의 내용을 파악하기 어려워, 필요한 정보를 찾기 위해 다시 방문해야 하는 불편함이 생겼습니다. 더불어 기존의 서비스들은 주로 북마크 제목을 기반으로 검색 기능을 제공했기에, 세세한 내용 검색에는 한계가 있었습니다.

이러한 문제점을 자연스럽게 해결하고자, 사용자가 별도의 로그인 없이도 크롬 브라우저에 저장된 북마크를 chromeAPI.bookmarks로 불러와, 각 북마크 내부의 키워드까지 부드럽게 검색할 수 있는 익스텐션을 개발하게 되었습니다.

## 2. 기술 스택
<div align="center">
  
| 프론트엔드 | 백엔드 | 빌드 | 테스트 |
| ---------------------------------------------- | ---------------------------------------------- | ---------------------------------------------- | ---------------------------------------------- | 
| <img src="https://img.shields.io/badge/React-3B4250?style=flat-square&logo=React&logoColor=#61DAFB"/> <br /> <img src="https://img.shields.io/badge/Zustand-3B4250?style=flat-square&logo=React&logoColor=#3B4250"/> <br /> <img src="https://img.shields.io/badge/Tailwind-3B4250?style=flat-square&logo=tailwindcss&logoColor=#06B6D4"/> <br /> <img src="https://img.shields.io/badge/Axios-3B4250?style=flat-square&logo=Axios&logoColor=#5A29E4"/> | <img src="https://img.shields.io/badge/Node-3B4250?style=flat-square&logo=Node.js&logoColor=#5FA04E"/> <br /> <img src="https://img.shields.io/badge/Puppeteer-3B4250?style=flat-square&logo=Puppeteer&logoColor=#40B5A4"/> <br /> <img src="https://img.shields.io/badge/Express-3B4250?style=flat-square&logo=Express&logoColor=#646CFF"/>| <img src="https://img.shields.io/badge/Vite-3B4250?style=flat-square&logo=vite&logoColor=#646CFF"/> | <img src="https://img.shields.io/badge/Vitest-3B4250?style=flat-square&logo=vitest&logoColor=#6E9F18"/> |

</div>


## 3. 구현 세부사항
### 3-1. 어떻게 북마크를 가져올까?
프로젝트를 시작하기 위해선 프로젝트 키워드인 사용자의 크롬 북마크를 수집해야 했습니다.
북마크를 가져오기 위해 chromeAPI를 사용하기 위해 익스텐션을 선택한 것이기 떄문에, chromeAPI를 사용해 북마크를 가져오고자 했습니다.
공식문서에서 chromeAPI.getBookmark()를 사용한 예제를 참고하여 가져오고자 했으나 dom을 직접적으로 건드리는 형태임을 확인했습니다.
공식문서의 예문처럼 Dom으로 컨트롤하기엔 react에서는 지양해야할 방향성이므로 다른방법을 모색하던 와중 순수하게 
chromeAPI.getBookmark()가 어떤 data를 가져오는지 의문이 들어 자료를 꺼내 확인 해보았습니다.

``` js
  useEffect(() => {
    chrome.bookmarks.getTree((treeList) => {
      console.log(treeList);
      getNewTree(treeList);
    });
  }, []);
```
  
```chrome.bookmarks.get``` 로 가져온 북마크는 사용자 북마크 폴더 개수에 따라 얼마든지 중첩이 가능한 구조였습니다.

![트리구조](https://github.com/user-attachments/assets/8694307c-720f-4d15-9c86-9edfc674d02b)

이러한 트리 구조는 React상태 관리시 불변성을 지키기 힘든 이슈가 있고, 불변성을 지키기 위해서 저희가 사용할 목적에 맞는 자료구조로의 개선이 필요했습니다. 
저희는 사용자 폴더가 얼마나 트리 구조인지 알수 없기 때문에, 재귀를 사용해 얼마나 폴더가 중첩되어 있든 필요한 정보만 추출하여 리스트 저장해야 했기 때문에 재귀 함수를 작성하고자 했습니다.<br />
재귀는 함수를 지속적으로 호출하면서, 콜 스택에 중첩되며 메모리를 많이 차지하게 되고 이는 속도 저하를 야기할 수 있었습니다.

재귀 함수의 초기 버전은 forEach와 반복 조건 등 가독성 및 성능에서 떨어진다고 판단했고, 아래와 같이 조건을 중첩하고 불필요한 조건을 줄여 가독성과 성능을 향상 시켰으며, ```forEach```를 ```for...of```로 변경해 더욱 간결하고 중간에 반복을 종료할 수 있도록 리팩토링 했습니다.

```js
// 리팩토링 버전
const getNewTree = (nodeItems) => {
    const getNewTreeResult = [];

    function recursive(nodes) {
        for (const node of nodes) {
            if (node && typeof node === "object") {
                if (node.title && node.url) {
                    getNewTreeResult.push(node);
                }
                if (node.children) {
                    recursive(node.children);
                }
            }
        }
    }

    recursive(nodeItems);
    setUrlNewList(getNewTreeResult);
};
```

### 3-2. SPA, Iframe 해결
저희가 맨 처음 생각한 방법은 북마크 URL을 받으면 해당 URL로 fetch를 요청하고 요청한 URL에 대한 HTML 문자열을 받아온 후 그 HTML에서 일치하는 내용을 찾는 것이 목표였습니다.<br />
MPA[Multi Page Application]일 경우 저희 의도대로 HTML을 추출하고 일치하는 키워드를 찾을 수 있었지만, SPA[Single Page Application}의 경우 가져오는 HTML은 렌더링 되기 전의 비어있는 HTML로 일치하는 키워드를 찾을 수 없었습니다.<br />
iframe의 경우도 마찬가지로 HTML을 fetch로 요청하여 응답받을 시 iframe 내부 html에 접근이 불가능 했습니다.<br /><br />

<div align="center">
  
  | | MPA | SPA |
  | ----- | ------ | ------ | 
  | 렌더링 시점 | 완성된 페이지 전체를 렌더링 | HTML에 페이지의 일부분을 받아 점진적 렌더링 |
  
</div>

MPA와 더불어 SPA와 iframe까지 대응하기 위해 Puppeteer의 헤드리스 브라우저 모드를 사용하여 SPA의 경우 인간적인 브라우징 패턴을 모방해서 SPA페이지 로딩을 기다리고 iframe 내부 내용또한 읽어와 해당 컨텐츠 내에 접근할 수 있었습니다.<br />
iframe경우 iframe내부에 독립적인 DOM을 제공하기 때문에 innerText를 통해 접근할 수 없던 문제가 있었지만 iframe이 발견 되면 iframe src속성을 통해 iframe내부 크롤링을 시작하고, 내부에서도 일치하는 키워드가 있는지 찾아낼 수 있었습니다.<br />

### 3-3. 어떻게 키워드가 포함된 문장을 가져올까?
![image (3)](https://github.com/user-attachments/assets/bf9e36aa-db03-42fd-bb66-f0000709f0e3)
가져온 문자열은 다음과 같이 HTML 문서 구조 내에서 요소들 사이의 공백과 개행을 유지하기 위해 개행문자가 포함돼 있는 문자열이었습니다.
이를 해결하기 위해 정규 표현식을 사용하여 데이터를 가공했습니다.
```js
const getAllSentence = (innerText) => {
  return innerText
    .replace(/\n|\r|\t/g, " ")
    .split(/(?<=다\. |요\. |니다\. |\. |! |\? )/)
    .reduce((array, sentence) => {
      const trimmedSentence = sentence.trim();
      if (trimmedSentence) {
        array.push(trimedSentence);
      }
      return array;
    }, []);
};
```
위 코드는 개행 문자를 공백으로 대체한 후 각 문장을 구분하여 배열에 담는 로직으로 개행문자 제거 로직을 구현했습니다.

또한 find함수를 통해 키워드를 포함하는 문장과 일치하는 첫 번째 문장을 가져오는 작업을 진행했지만, find함수는로 가져온 하나의 문장이 너무 긴 문자열을 포함해서 클라이언트 측에서 이 문자열을 표시하기에 어려움이 따랐기 때문에 문장을 단어 단위로 자르고, 키워드인 단어를 찾아서 그 단어 기준으로 "특정 단어 수" 만큼 데이터를 가져오도록 하여 해결했습니다.<br />

<div align="center">
  
  ![스크린샷 2025-03-04 오후 5 22 21](https://github.com/user-attachments/assets/15e7525f-44ff-4719-8974-69ed3085235d)
  
</div>

그러나 사용자에게 보여주어야 할 공간을 채우기 위해 단어의 길이와 개수를 계산해 특정 단어 수를 도출한 "특정 단어 수"를 사용해 조합한 문자열이 사용자가 느끼기에 부자연스러운 느낌을 주어 사용자 경험을 개선을 위해 한 문장을 모두 가져온 뒤 ```text-overflow: ellipsis;```를 적용해 문장을 공간에 맞게 처리할 수 있었습니다.

### 3-4. 너무 느린 크롤링 속도 어떻게 해결할까?
북마크 내부의 DOM을 탐색하고 사용자가 입력한 키워드를 검색하기 위해선 크롤링이 필수적이었습니다.<br />
다만, 크롤링을 하는데에 걸리는 시간은 절대적으로 필요한 시간이고 이 시간을 줄일 방법은 많지 않았습니다. 크롤링이 끝나기를 기대하는 사용자의 이탈을 막기 위한 대책을 세워야 했고, 한 번에 여러 개의 크롤링 요청을 보낼 시 Docker에 감싸 AWS의 EC2에 올려 놓은 크롤링 서버가 받은 인스턴스의 램 메모리 1GB보다 많은 메모리를 사용해서 OOM을 야기하며 서버가 죽는 이슈도 있었기 때문에, 서버에 대한 대책도 세워야 했습니다.

#### 3-4-1. 사용자 UI/UX를 통해 해결해보자.
각 사용자마다 가지고 있는 북마크의 개수가 달라서 사용자가 얼마만큼을 기다리는지 예측하기 힘들었습니다. 그렇기에 다양한 사용자가 한 눈에 로딩 중이라는 것일 확인 할 수 있도록 스켈레톤 UI를 사용했습니다.

#### 3-4-2. 로직 변경을 통해 크롤링 서버 OOM을 피해보자.
기존 로직은 ```Promise.allSetteld```를 사용했었습니다. allSetteld는 all과 다르게 에러가 발생해도 fullfield, rejected를 동시에 반환할 수 있기 떄문에 allSetteld를 사용하여 사용자에게 한 번에 모든 북마크를 요청해서 한 번에 검색 결과를 제공하려고 했습니다.
하지만 allSetteld 같은 경우 사용자가 50개의 북마크를 가지고 있다면 50개를 한 번에 보내기 때문에 서버에 과부하와 OOM을 야기하기도 하며, 사용자는 50개가 전부 끝나기 전까지는 결과를 보지 못한다는 단점이 있었습니다.<br />
때문에 allSetteld를 5개 단위로 묶은 프로미스 배열을 Promise.race 메서드를 사용하여 병렬로 크롤링을 처리하면서 서버의 과부화도 줄이고, 사용자가 즉시 크롤링이 끝난 결과를 순차적으로 볼수 있게 변경했습니다.

크롤링 서버를 AWS의 EC2를 사용하여 배포하기로 했고, EC2의 프리티어 계정을 사용하여 배포했습니다.
프리티어의 경우 이 있었고 Promise.allSetteld를 통해 크롤링을 여러번 요청하면 해당 많은 메모리를 차지해 OOM을 이기하며 서버가 다운되는 현상이 발견되었습니다.

### 3-5. 크롤링으로 가져온 검색 결과를 한 번 더 검색할 순 없을까?
생각외로 북마크는 사용자 취향에 맞는 페이지들을 북마크 해놓은 것이어서 키워드 선정을 잘 못하게 되면 사용자가 원했던 결과보다 더욱 많은 결과가 검색되었습니다. 그래서 1차로 나온 검색 결과를 가지고 추가적으로 검색을 하면 사용자 편의성이 증가할 것이라고 생각했습니다.
하지만 문제는 크롤링의 속도였습니다. Promise.race() 를 통해 느린 크롤링의 속도를 UX적으로 개선했는데, 다시 추가 검색을 통해 느린 속도를 경험한다면 사용자 이탈률이 높아질 것이라고 생각했습니다.
따라서 추가 검색은 1차 검색보다 빠른 속도를 사용자에게 제공해야 했고, 1차 검색 시 단순히 키워드가 포함된 문장만 가져오는 것이 아닌 페이지 HTML 본문에서 ```innerText```를 통한 텍스트를 전부 가져와 사용자가 추가 검색 시 크롤링이 아닌 가져온 HTML 본문에서 추가 키워드를 찾아 제공하기로 했습니다.

#### 3-5-1. 가져온 HTML을 어디에 저장할지의 문제
firebase, mongoDB 등 다양한 스토리지가 있지만 저희는 익스텐션 LocalStorage에 저장하기로 했습니다.
익스텐션 LocalStorage는 크롬 익스텐션에서 제공해주는 스토리지로 최대 10mb까지 사용할 수 있고 사용하기에 간편하다는 장점이 시간이 부족했던 저희 프로젝트에 적합한 선택이었다고 생각합니다.
LocalStorage의 부족한 용량을 넘기면 생기는 에러를 피하기 위해 ```StorageManager API``` 를 사용하여 현재 용량을 확인하고, 만약 용량으 넘길 시 기존에 있던 HTML 본문 중 가장 오래된 요소를 지우고 최신 요소를 넣는 방법을 선택했습니다.

## 4. 의사결정 방식


## 5. 일정

### 1차 개발 : 2025년 1월 13일 ~ 2025년 2월 2일
> 진행 사항
> - 협업 규칙 및 Git 플로우 규칙 설정
> - 프로젝트 esLint, pretteir, husky 설정
> - 서버 Express, Puppeteer 설정
> - 크롤링[Puppeteer] 서버 개발
> - 익스텐션 검색 로직 구현
> - 익스텐션 옵션페이지 추가

### 2차 개발 : 진행 중
> 진행 사항
> - 옵션 페이지 기획 변경
> - 리팩토링 진행
> - 전역상태 설정
> - fetch Aixos로 변경

## 6. 팀원
- 심소은 : euns127@gmail.com
- 박성훈 : seonghoon.dev@gmail.com
- 이수보 : nullzzoa@gmail.com
