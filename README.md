# 프론트엔드 배포 파이프라인

## 개요

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/83c75a39-3aba-4ba4-a792-7aefe4b07895/6912169d-ce70-41bf-b624-946d4ee984eb/Untitled.png)

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://wonjung-bucket.s3-website-ap-southeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름:

아래 내용은 블로그 회고글의 일부이며, '~입니다.'가 아닌 '~다.'로 끝납니다.
회고글과 과제 내용이 겹쳐서 회고글에 작성하여 가져왔습니다.
양해 부탁드립니다.

## 💰 GitHub Actions과 CI/CD 도구
---

GitHub Actions는 CI/CD 도구다.
GitHub 레파지토리 내에서 코드의 빌드, 테스트, 배포 등 여러 작업을 자동화하기 위해 사용한다.

GitHub Actions는 코드 푸쉬, 풀 리퀘스트, 이슈 생성 등 다양한 GitHub 이벤트에 반응하여 워크 플로우가 실행된다.

레파지토리 내 `.github/workflows` 폴더에서 YAML 파일로 워크 플로우를 정의한다.

### 💵 Jenkins와의 차이점

예전에 CI/CD라고 하면 Jenkins만 떠올렸는데, 요즘에는 GitHub Actions를 많이 사용하는 듯하다.

Jenkins와의 차이점은 Jenkins는 오픈소스 CI/CD 도구로, 사용자가 직접 서버에 설치하고 관리해야 한다.
오랜 시간 동안 사용되어 온 만큼 다양한 커뮤니티 지원이 되고, 많은 플러그인을 통해 다양한 환경에 맞춰 확장할 수 있지만 그만큼 러닝커브가 높다.

하지만 GitHub Actions는 GitHub에서 제공하여 별도의 설치가 필요없다.
또한 YAML 파일 하나로 워크플로우를 정의하고 GitHub 이벤트에 기반하여 트리거된다.

### 💵 GitHub Actions에서 워크 플로우 파일을 찾는 방법

GitHub 이벤트가 발생하면 레파지토리 내 `.github/workflows` 폴더에 있는 모든 YAML 파일을 스캔한다.
각 YAML 파일에 정의된 `on` 키워드를 확인하여 이벤트에 대한 워크 플로우가 트리거되어 실행된다.

### 💵 Repository secret과 환경변수

워크플로우 파일에서 빌드된 파일을 AWS에 저장하기 위해 IAM 계정 정보를 입력하여 접근 권한을 얻게 된다.

하지만 이런 정보는 보안상 공개되면 안되는 정보기 때문에 Repository secrets에 저장한다.
Repository secrets에 저장하면 Github 내부에서 암호화되어 저장된다.

워크플로우 파일에서 `${{ secrets.AWS_ACCESS_KEY_ID }}` 형식으로 참조할 수 있으며, AWS_ACCESS_KEY_ID는 저장 시에 설정할 수 있는 이름이다.

환경 변수는 워크플로우 실행에 필요한 설정값을 정의할 때 사용한다.
```yaml
env:
  NODE_ENV: production
  API_URL: https://api.example.com
```
워크플로우 파일 안에서 `env` 키를 사용하여 정의할 수 있고 `$NODE_ENV`, `${{ env.API_URL }}` 형태로 참조할 수 있다.

이번 과제에서 사용하진 않았지만 민감한 정보가 아니고 워크플로우 파일에서 전역적으로 쓴다면 활용할 수 있다.

## 💰 S3와 스토리지
---

S3는 서버라기 보단 스토리지다.
여지껏 jsp + Spring으로 만들어진 프로젝트를 Docker에 배포한 적만 있어서 이 개념이 헷갈렸다.

서버라 함은 OS 위에 Java면 JVM, JavaScript라면 Node.js를 띄운 다음에 코드를 실행시키는 이미지다.
빌드된 파일을 S3에 저장하고 제공된 URL로 접근 시에 페이지를 띄워줘서 서버인 줄 알았다.

> S3는 빌드가 완료된 HTML, CSS, JavaScript 파일을 포함한 정적 파일을 갖고 있다가 전달하는 것뿐인데 어떻게 동작할 수 있을까?

빌드된 파일을 S3에 저장하고 제공된 URL로 접근 시에 페이지를 띄워주긴 하지만 그 파일들은 브라우저에서 실행된다.
따라서 S3가 정적 파일을 저장하고 있다가 제공만 해도 브라우저에서 화면을 확인할 수 있다.

### 💵 Next.js의 Image 컴포넌트

과제에서는 `create-next-app`으로 만든 프로젝트를 빌드하고 배포해서 괜찮았지만 Next.js 프로젝트를 S3에 배포하는 것은 사실 위험하다.

만약에 `png` 파일을 `Image` 컴포넌트를 사용하여 보여준다면 오류가 발생한다.
Next.js의 Image 컴포넌트는 서버 측에서 이미지 최적화를 하기 때문이다.

Image 컴포넌트는 요청이 들어오면 해당 이미지를 변환하고 최적화 하는 과정을 거치는데, 브라우저가 아닌 Node.js나 Vercel과 같은 서버 환경에서 이뤄진다.
즉 동적 서버 환경에서 이미지 최적화 API를 활용하기 때문에 정적 스토리지인 S3에 빌드 결과물을 올리면 오류가 발생한다.

## 💰 CloudFront와 CDN
---

CloudFront는 AWS에서 제공하는 CDN 서비스다.
그럼 CDN 서비스란 무엇일까?

CDN은 Content Delivery Network의 약자로, 쉽게 말해 사용자와 가까운 곳에 파일을 저장하는 저장소이다.

S3 저장소가 사용자와 물리적인 거리가 멀어 파일을 받을 때까지 시간이 오래 걸리는 문제를 사용자와 가까운 엣지 로케이션이라는 곳에 캐싱하여 시간을 단축한다.

사용자가 요청을 하면 CloudFront는 가장 가까운 엣지 로케이션에서 캐싱된 데이터를 제공한다.

또한 HTTP 압축, 불필요한 헤더 제거 등 캐싱 정책에 따라 최적화된 컨텐츠를 전달한다.

### 💵 성능 비교

CDN을 사용하면 얼마나 빨라질까?
S3의 엔드포인트와 CloudFront의 배포 도메인 주소를 비교해보자.
비교를 위해 https://www.webpagetest.org/ 사이트를 이용했다.
Chapt GPT한테 성능 비교 시각화하고 싶은데 알려달라니 알려줬다. ㅋ

![](https://velog.velcdn.com/images/jang_expedition/post/25caf287-b027-447f-8678-46dbf4c3886a/image.png)

먼저 S3에 직접 접근했을 때의 지표이다.
첫 번째로 나오는 TTFB는 위에서 언급한 핵심 지표다.
그 다음으로 차례대로 설명하면
- **Start Render**: 브라우저가 화면에 처음으로 비어있지 않은 콘텐츠가 그려지는 시점까지 걸리는 시간이다.
- **First Contentful Paint**: 브라우저가 첫 번째 텍스트나 이미지, 캔버스 등 DOM의 실제 콘텐츠를 그려내는 데 걸린 시간이다.
- **Speed Index**: 페이지가 얼마나 빠르게 보여지는지를 수치화한 것으로, 낮을 수록 좋다.
- **Largest Contentful Paint**: 화면에 나타나는 요소들 가운데 가장 큰 텍스트 혹은 이미지 요소가 렌더링된 시점까지 걸린 시간이다.
- **Cumulative Layout Shift**: 페이지가 로드되는 동안 발생하는 레이아웃 변경을 정량화한 지표로 0에 가까울수록 사용자가 화면 이동없이 안정적인 레이아웃을 경험한다.
- **Total Blocking Time**: 메인 스레드가 사용자가 입력에 응답할 수 없을 정도로 막혀 있는 시간을 합산한 지표로 오래 걸리는 JavaScript 작업이 있을 경우 증가할 수 있고 사용자의 반응성에 영향을 준다.
- **Page Weight**: 페이지가 로딩하는 전체 리소스 크기를 합산한 수치다.

Cloud Front를 사용하면 얼마나 좋아질까?

![](https://velog.velcdn.com/images/jang_expedition/post/2f2c220d-b6e6-49a3-b785-704f690f26ad/image.png)

모든 수치가 1초를 넘지 않고 확연히 좋아진 걸 알 수 있다.
Page Weight는 약 200kb 줄어들었다.
두 수치를 비교하면 아래와 같다.


| 성능 지표 | CloudFront (CDN 적용) |	S3 (직접 접근) |
| --- | --- | --- |
| TTFB | 0.175 s | 3.37 s |
| Start Render | 약 0.5 s	| 약 3.70 s
| FCP	| 약 0.48 s	| 약 3.71 s
| Speed Index	| 약 0.51 s	| 약 3.81 s
| LCP	| 약 0.48 s	| 약 3.95 s
| Page Weight	| 194 KB	| 483 KB

### 💵 캐시 무효화(Cache Invalidation)

캐시 무효화는 CDN에 저장된 캐시된 컨텐츠가 최신 상태가 아닐 때 이를 삭제하거나 갱신하는 과정을 의미한다.

캐시 무효화가 필요한 이유는 웹 애플리케이션의 콘텐츠가 변경되었을 때 사용자에게 최신 데이터를 제공하여 모든 사용자가 동일한 최신 컨텐츠를 받을 수 있게 하기 위해서다.
또한 잘못된 파일이 발견된 경우에 해당 캐시를 제거함으로써 문제를 해결할 수 있다.
