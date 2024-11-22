- [프론트엔드 배포 파이프라인](#프론트엔드-배포-파이프라인)
  - [1.주요 링크](#1주요-링크)
  - [2. 개요](#2-개요)
    - [2.1 정적 웹사이트 배포 파이프라인](#21-정적-웹사이트-배포-파이프라인)
    - [2.2 GitHub Actions 워크플로우](#22-github-actions-워크플로우)
  - [3. 주요 개념](#3-주요-개념)
    - [3.1 GitHub Actions과 CI/CD 도구](#31-github-actions과-cicd-도구)
      - [3.1.1 CI/CD란?](#311-cicd란)
      - [3.1.2 CI/CD 도구](#312-cicd-도구)
      - [3.1.3 GitHub Actions](#313-github-actions)
    - [3.2 S3와 스토리지](#32-s3와-스토리지)
      - [3.2.1 Amazon S3의 주요 특징](#321-amazon-s3의-주요-특징)
      - [3.2.2 Amazon S3의 주요 활용 사례](#322-amazon-s3의-주요-활용-사례)
    - [3.3 CloudFront와 CDN](#33-cloudfront와-cdn)
      - [3.3.1 CDN](#331-cdn)
        - [1) CDN이란 무엇인가요?](#1-cdn이란-무엇인가요)
        - [2) CDN이 중요한 이유](#2-cdn이-중요한-이유)
        - [3) CDN의 주요 이점](#3-cdn의-주요-이점)
        - [4) CDN을 통해 전송할 수 있는 콘텐츠](#4-cdn을-통해-전송할-수-있는-콘텐츠)
        - [4) CDN 작동 방식](#4-cdn-작동-방식)
        - [5) CDN의 주요 사용 사례](#5-cdn의-주요-사용-사례)
        - [5) CDN의 활용](#5-cdn의-활용)
      - [3.3.2 Amazon CloudFront](#332-amazon-cloudfront)
    - [3.4 캐시 무효화(Cache Invalidation)](#34-캐시-무효화cache-invalidation)
    - [3.5 Repository secret과 환경변수](#35-repository-secret과-환경변수)
- [\[심화과제\] CDN과 성능최적화](#심화과제-cdn과-성능최적화)
  - [1. 개요](#1-개요)
  - [2. 테스트 환경](#2-테스트-환경)
  - [3. 성능 측정 결과](#3-성능-측정-결과)
    - [3.1 네트워크 요청 분석](#31-네트워크-요청-분석)
    - [3.2 주요 성능 지표 비교](#32-주요-성능-지표-비교)
    - [3.3 응답 헤더 분석](#33-응답-헤더-분석)
    - [3.4 상세 응답 시간 분석](#34-상세-응답-시간-분석)
      - [시크릿 창에서 4회 반복 측정 결과:](#시크릿-창에서-4회-반복-측정-결과)
  - [4. 결론](#4-결론)
- [참고 자료](#참고-자료)
  - [공식문서](#공식문서)
  - [블로그 및 기타소스](#블로그-및-기타소스)

# 프론트엔드 배포 파이프라인

> 과제 팁1: 다이어그램 작성엔 Draw.io, [Lucidchart](https://lucid.app/documents#/documents?folder_id=home) 등을 이용합니다.

> 과제 팁2: 새로운 프로젝트 진행시, 프론트엔드팀 리더는 예시에 있는 다이어그램을 준비한 후, 전사 회의에 들어가 발표하게 됩니다. 미리 팀장이 되었다 생각하고 아키텍쳐를 도식화 하는걸 연습해봅시다.

> 과제 팁3: 캐시 무효화는 배포와 장애 대응에 중요한 개념입니다. .github/workflows/deployment.yml 에서 캐시 무효화가 어떤 시점에 동작하는지 보고, 추가 리서치를 통해 반드시 개념을 이해하고 넘어갑시다.

> 과제 팁4: 상용 프로젝트에선 DNS 서비스가 필요합니다. 도메인 구입이 필요해 본 과제에선 ‘Route53’을 붙이는걸 하지 않았지만, 실무에선 다음과 같은 인프라 구성이 필요하다는걸 알아둡시다

## 1.주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghae99.s3-website-ap-southeast-2.amazonaws.com/
- CloudFront 배포 도메인 이름: https://d2xvtvxlifkv9.cloudfront.net

## 2. 개요

### 2.1 정적 웹사이트 배포 파이프라인

<img src="/public/diagram.png" alt="프론트엔드 배포 파이프라인 다이어그램" />

0. **Git Repository**

   - 개발자가 GitHub 저장소에 코드 푸시

1. **CI/CD 파이프라인**

   - `🐱GitHub Actions`에서 자동 빌드 및 테스트 실행
   - `🔑 AWS IAM`을 통한 리소스 접근 권한 관리

2. **빌드 결과물 배포**

   - `🪣 S3 버킷` 에 빌드 산출물 업로드
   - `🔑 AWS IAM` 정책을 통한 접근 권한 제어
     - S3 버킷에 대한 쓰기 권한 제공
     - CloudFront 배포 설정에 대한 접근 권한 관리

3. **CDN 배포**
   - `🌎 CloudFront`를 통한 전역 배포
   - 캐시 전략을 통한 성능 최적화

### 2.2 GitHub Actions 워크플로우

[GitHub Actions 워크플로우](https://docs.github.com/ko/actions/writing-workflows/quickstart)를 작성해 배포가 진행되도록 했습니다.

1. 저장소를 체크아웃합니다.

   ```yaml
   - name: Checkout repository
           uses: actions/checkout@v2
   ```

2. Node.js 18.x 버전을 설정합니다.

   ```yaml
   - name: Setup Node.js and install pnpm
     uses: actions/setup-node@v3
     with:
       node-version: 18 # Node.js 버전을 18 이상으로 설정
   - run: corepack enable
   - run: corepack prepare pnpm@latest --activate
   ```

3. 프로젝트 의존성을 설치합니다.

   ```yaml
   - name: Install dependencies
     run: pnpm install --frozen-lockfile # or npm ci
   ```

4. Next.js 프로젝트를 빌드합니다.

   ```yaml
   - name: Build
     run: pnpm build # or npm run build
   ```

5. AWS 자격 증명을 구성합니다.

   ```yaml
   - name: Configure AWS credentials
     uses: aws-actions/configure-aws-credentials@v1
     with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }} # IAM 계정 생성시 발급받은 액세스 키
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # IAM 계정 생성시 발급받은 비밀 액세스 키
       aws-region: ${{ secrets.AWS_REGION }} # S3를 세팅한 리전(지역)의 코드
   ```

6. 빌드된 파일을 S3 버킷에 동기화합니다.

   ```yaml
   - name: Deploy to S3
     run: |
       aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
   ```

7. CloudFront 캐시를 무효화합니다.

   ```yaml
   - name: Invalidate CloudFront cache
     run: |
       aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
   ```

## 3. 주요 개념

### 3.1 GitHub Actions과 CI/CD 도구

#### 3.1.1 CI/CD란?

> [redhat 문서](https://www.redhat.com/ko/topics/devops/what-is-ci-cd)를 참고하여 작성하였습니다.

CI/CD는 소프트웨어 개발 라이프사이클을 간소화하고 가속화하는 방법으로, 지속적 통합(Continuous Integration)과 지속적 제공/배포(Continuous Delivery/Deployment)를 포함합니다.

1. **CI (지속적 통합)**

   - 코드 변경 사항을 **공유 리포지토리에 자동으로 자주 통합**합니다.
   - 변경된 코드가 기존 시스템에 문제가 없는지 자동으로 빌드하고 테스트하여 검증합니다.
   - 테스트는 단위 테스트와 통합 테스트를 포함하며, 충돌을 빠르게 발견하고 수정할 수 있습니다.
   - CI는 새로운 코드가 기존 코드와 충돌하지 않도록 빠르게 검증할 수 있어, 품질 높은 소프트웨어를 유지할 수 있습니다.
   - 💡 CI에서의 **"자동으로 자주 통합한다"** 의 개념이 헷갈려서 추가 리서치 해보았습니다.
   - > - **CI(Continuous Integration)** 는 단순히 "작은 단위로 자주 머지한다"는 개념을 넘어서, **코드의 품질과 안정성을 자동화된 과정으로 확인하고 통합하는 전체 프로세스**를 말합니다.
     >
     > - **작은 단위로 자주 머지** 👷
     >   - 개발자들이 작업 중인 코드를 짧은 간격으로 공유 저장소에 병합합니다.
     >   - 이렇게 하면 병합 충돌을 빠르게 발견하고, 문제를 조기에 해결할 수 있습니다.
     > - **자동화된 검증** ✅
     >   - "자동으로 통합"의 의미는 단순히 코드를 머지하는 것 뿐만 아니라, **자동화된 검증 프로세스를 통해 병합된 코드의 품질과 안정성을 확인**하는 것입니다.
     >   - 예시: 코드가 병합될 때 아래 과정을 자동으로 실행
     >     - **테스트:** 코드 변경이 기존 기능을 깨지 않는지 확인.
     >     - **Lint/Code Style Check:** 코드 스타일을 자동 검증.
     >     - **빌드:** 애플리케이션이 정상적으로 빌드되는지 확인.

1. **CD (지속적 제공 및/또는 배포):**

   - **지속적 제공:** CI에서 빌드된 코드를 리포지토리로 릴리스하고, **운영 팀이 이를 빠르게 프로덕션 환경으로 배포**할 수 있도록 자동화합니다.
   - **지속적 배포:** 지속적 제공을 확장하여, **자동으로 변경 사항을 프로덕션 환경에 릴리스**합니다. 이 과정은 사용자 피드백을 빠르게 반영할 수 있도록 돕습니다.

   - **주요 차이점**

     | 항목               | 지속적 제공 (Continuous Delivery)                   | 지속적 배포 (Continuous Deployment)                                  |
     | ------------------ | --------------------------------------------------- | -------------------------------------------------------------------- |
     | **배포 주기**      | 자동화된 빌드 및 테스트 후, 수동으로 배포           | 자동화된 빌드, 테스트 후 자동으로 프로덕션 배포                      |
     | **배포 준비 상태** | 배포 준비 완료, 수동 배포 가능                      | 자동 배포가 이루어짐                                                 |
     | **운영 팀의 개입** | 배포 전에 운영 팀의 수동 승인이 필요                | 수동 개입 없이 자동 배포                                             |
     | **장점**           | 배포가 자동화되며, 배포 준비 상태를 항상 유지       | 빠른 피드백과 빠른 업데이트 가능, 배포 주기 짧음                     |
     | **단점**           | 운영 팀의 승인에 의존, 수동 개입으로 인한 지연 가능 | 자동화 의존도가 높아지며, 테스트가 부족할 경우 문제가 발생할 수 있음 |

   - 따라서, **지속적 제공**은 자동화된 빌드와 테스트 후, 운영 팀이 배포를 승인하는 방식인 반면, **지속적 배포**는 배포까지 자동화되어 있어 코드가 변경될 때마다 즉시 프로덕션 환경에 반영되는 방식입니다.

#### 3.1.2 CI/CD 도구

2024년 인기 있는 CI/CD 도구들은 [20+ Best CI/CD Tools for DevOps in 2024](https://spacelift.io/blog/ci-cd-tools)에 따라 다음과 같습니다:

<img src="https://spacelift.io/_next/image?url=https%3A%2F%2Fspaceliftio.wpcomstaging.com%2Fwp-content%2Fuploads%2F2022%2F09%2Fci-cd-tools-comparison.png&w=1920&q=75"/>

- Spacelift
- GitHub Actions
- Jenkins
- GitLab CI
- CircleCI

#### 3.1.3 GitHub Actions

- [GitHub Actions](https://docs.github.com/ko/actions/about-github-actions/understanding-github-actions)는 GitHub에서 제공하는 CI/CD 플랫폼으로, 빌드, 테스트, 배포를 자동화할 수 있는 도구입니다.
- pull request을 빌드하고 테스트하거나, 병합된 요청을 자동으로 프로덕션에 배포할 수 있는 **워크플로우를 구성**할 수 있습니다.
- 이를 통해 소프트웨어 개발 주기를 단축시키고, 품질을 높이며, 배포 주기를 빠르게 만들 수 있습니다.

### 3.2 S3와 스토리지

> **Amazon S3란 무엇인가요?**
>
> Amazon Simple Storage Service(Amazon S3)는 업계 최고의 확장성, 데이터 가용성, 보안 및 성능을 제공하는 객체 스토리지 서비스입니다.
> 모든 규모와 업종의 고객은 Amazon S3를 사용하여 데이터 레이크, 웹 사이트, 모바일 애플리케이션, 백업 및 복원, 아카이브, 엔터프라이즈 애플리케이션, IoT 디바이스, 빅 데이터 분석 등 다양한 사용 사례에서 원하는 양의 데이터를 저장하고 보호할 수 있습니다.
> Amazon S3는 특정 비즈니스, 조직 및 규정 준수 요구 사항에 맞게 데이터에 대한 액세스를 최적화, 구조화 및 구성할 수 있는 관리 기능을 제공합니다.

아래는 위 [아마존 공식문서](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/Welcome.html)를 기반으로 정리한 내용입니다.

Amazon S3는 데이터를 인터넷을 통해 저장, 검색, 관리할 수 있는 **확장 가능한 객체 스토리지 서비스**입니다.

#### 3.2.1 Amazon S3의 주요 특징

1.  **객체 스토리지**:

    - 데이터 저장 방식
    - 데이터는 **객체(Object)** 라는 단위로 저장됩니다.
    - 각 객체는 파일 데이터, 메타데이터, 고유 키(이름)로 구성됩니다.

2.  **버킷(Bucket)**:

    - S3에서 이 객체를 담는 컨테이너
    - 데이터는 **버킷(Bucket)** 에 저장됩니다.
    - 버킷은 데이터를 저장하는 논리적인 컨테이너 역할을 합니다.
    - 구조

      ```
          객체 스토리지 (Amazon S3)
             └─ 버킷 (Bucket)
                  ├─ 객체 1 (파일 + 메타데이터 + 고유 키)
                  ├─ 객체 2 (파일 + 메타데이터 + 고유 키)
                  └─ 객체 3 (파일 + 메타데이터 + 고유 키)
      ```

3.  **다양한 스토리지 클래스**:

    - 데이터를 어떻게 저장하고 비용을 최적화할지 선택하는 옵션
    - 사용 사례와 비용에 맞게 다양한 스토리지 클래스를 제공합니다.
      - **Standard**: 자주 액세스하는 데이터에 적합.
      - **Intelligent-Tiering**: 액세스 패턴에 따라 비용을 최적화.
      - **Glacier**: 장기 보관용(아카이브) 데이터에 적합.

4.  **유연한 액세스 제어**:

    - IAM 정책, 버킷 정책, ACL(Access Control List)을 통해 데이터 접근 권한을 세밀하게 관리할 수 있습니다.

#### 3.2.2 Amazon S3의 주요 활용 사례

- **정적 웹사이트 호스팅**: HTML, CSS, JavaScript 파일을 S3에 저장하여 정적 웹사이트를 호스팅.

- **백업 및 복구**: 중요한 데이터의 백업 및 복구 작업.

- **데이터 분석**: 로그, 이미지, 동영상 등 대량 데이터를 저장하고 분석.

- **미디어 저장소**: 이미지, 동영상, 문서 등 미디어 파일 저장 및 스트리밍.

- **빅데이터 및 머신러닝**: 대량의 데이터를 저장하고 Amazon EMR, SageMaker와 통합하여 분석.

### 3.3 CloudFront와 CDN

#### 3.3.1 CDN

> **CDN이란 무엇인가요?**
>
> 콘텐츠 전송 네트워크(CDN)는 데이터 사용량이 많은 애플리케이션의 웹 페이지 로드 속도를 높이는 상호 연결된 서버 네트워크입니다. CDN은 콘텐츠 전송 네트워크 또는 콘텐츠 배포 네트워크를 의미할 수 있습니다.

아래는 위 [아마존 공식문서](https://aws.amazon.com/ko/what-is/cdn/)를 기반으로 정리한 내용입니다.

##### 1) CDN이란 무엇인가요?

콘텐츠 전송 네트워크(CDN)는 데이터를 전 세계적으로 분산된 서버 네트워크에 저장하여 사용자가 콘텐츠에 더 빠르게 접근할 수 있도록 도와주는 기술입니다.

- 사용자가 웹사이트에 접속하면 CDN 서버는 **지리적으로 가장 가까운 서버**에서 데이터를 제공합니다.
- 이를 통해 대기 시간을 줄이고, 대용량 파일(예: 동영상, 이미지)의 로딩 속도를 크게 향상시킵니다.

![CDN 이미지](http://nexnetsolutions.com/wp-content/uploads/2021/08/Content-Delivery-.png)

출처: [Nexnet Solutions](https://nexnetsolutions.com/solutions/telecom-service-provider/content-delivery-network-cdn/)

##### 2) CDN이 중요한 이유

- **대기 시간 감소**: 네트워크 지연을 줄여 사용자 경험 개선.
- **서버 효율성 향상**: 중간 서버가 웹 트래픽을 분산 처리하여 원본 서버의 부담 감소.
- **비용 절감**: 대역폭 사용량을 줄이고 호스팅 비용 감소.
- **사용자 환경 개선**: 빠른 로딩과 안정적인 서비스 제공.

##### 3) CDN의 주요 이점

1. **페이지 로드 시간 단축**:
   - 느린 로드 시간은 사용자 이탈을 유발합니다. CDN은 이를 방지하고 사이트 체류 시간을 늘립니다.
2. **대역폭 비용 절감**:
   - 캐싱 및 최적화를 통해 원본 서버의 데이터 제공량을 줄이고 비용을 절감합니다.
3. **콘텐츠 가용성 강화**:
   - 트래픽 폭증이나 하드웨어 오류에도 서버 다운을 방지.
   - 여러 CDN 서버 간의 백업을 통해 중단 없는 서비스 제공.
4. **웹 사이트 보안 강화**:
   - **DDoS 공격**에 대한 방어, 보안 연결(SSL/TLS)을 통한 데이터 보호.

##### 4) CDN을 통해 전송할 수 있는 콘텐츠

1. **정적 콘텐츠**:
   - 변하지 않는 데이터(예: 로고, 이미지, CSS 파일).
   - 캐싱에 적합.
2. **동적 콘텐츠**:
   - 사용자별로 변경되는 데이터(예: 소셜 미디어 피드, 사용자 맞춤형 콘텐츠).
   - 동적 가속을 통해 빠르게 제공.

##### 4) CDN 작동 방식

1. **캐싱**:
   - 정적 콘텐츠를 캐싱 서버에 저장하여 빠르게 제공.
2. **동적 가속**:
   - 실시간 콘텐츠 요청 속도 향상을 위해 서버 연결 최적화.
3. **엣지 로직 계산**:
   - 사용자의 요청을 분석하고, 콘텐츠를 최적화하거나 잘못된 요청을 처리.

##### 5) CDN의 주요 사용 사례

1. **고속 콘텐츠 전송**:
   - 글로벌 사용자를 대상으로 빠르고 안정적인 서비스 제공.
   - 예: Reuters는 Amazon CloudFront를 사용하여 전 세계에 뉴스를 신속히 전송.
2. **실시간 스트리밍**:
   - 비디오 및 오디오 스트리밍 기업이 대역폭 비용 절감 및 확장성 강화.
   - 예: Hulu는 CloudFront를 통해 초당 20GB 이상의 데이터를 스트리밍.
3. **다중 사용자 확장**:
   - 동시 접속자가 많은 서비스의 안정적인 확장 지원.
   - 예: King(소셜 게임 회사)은 매일 수백 테라바이트의 콘텐츠를 제공.

##### 5) CDN의 활용

CDN은 전 세계적으로 콘텐츠를 효율적으로 제공하여 **속도, 안정성, 보안, 비용 절감**의 이점을 제공합니다. 특히 글로벌 사용자 대상의 대규모 웹사이트와 애플리케이션에서 필수적인 기술로 자리 잡고 있습니다.

#### 3.3.2 Amazon CloudFront

> **Amazon CloudFront란 무엇입니까?**
>
> Amazon CloudFront는 .html, .css, .js 및 이미지 파일과 같은 정적 및 동적 웹 콘텐츠를 사용자에게 더 빨리 배포하도록 지원하는 웹 서비스입니다.
> CloudFront는 엣지 로케이션이라고 하는 데이터 센터의 전 세계 네트워크를 통해 콘텐츠를 제공합니다.
> CloudFront를 통해 서비스하는 콘텐츠를 사용자가 요청하면 지연 시간이 가장 낮은 엣지 로케이션으로 요청이 라우팅되므로 가능한 최고의 성능으로 콘텐츠가 제공됩니다.

CloudFront는 AWS의 콘텐츠 전송 네트워크(CDN) 서비스로, 전 세계에 분산된 서버를 통해 파일을 빠르게 전달합니다.
CloudFront는 파일 캐시를 관리하고, 사용자가 파일을 더 빠르게 접근할 수 있도록 최적화합니다.

### 3.4 캐시 무효화(Cache Invalidation)

CloudFront의 캐시를 무효화하는 과정은 배포 후 반드시 수행되어야 합니다.
새로 배포된 파일이 캐시로 인해 사용자에게 전달되지 않는 문제를 방지하고, 최신 파일을 제공할 수 있도록 합니다.
이 과정은 자동화된 배포 프로세스의 중요한 부분입니다.

### 3.5 Repository secret과 환경변수

GitHub에서 제공하는 `repository secret`과 `환경 변수`를 사용하여 민감한 정보를 안전하게 관리할 수 있습니다.
이를 통해 AWS 자격 증명, API 키 등 중요한 정보를 코드에 포함시키지 않고 안전하게 저장하고 사용할 수 있습니다.

---

# [심화과제] CDN과 성능최적화

> 과제 팁1 : CDN 도입후 성능 개선 보고서 영역은 [프론트엔드 개발자를 위한 CloudFront](https://sprout-log-68d.notion.site/CloudFront-2c0653cb130f42b2b21078389511cca2) 에서 네트워크 탭을 비교한 영역을 참고해주세요. 이미지와 수치등을 표로 정리해서 보여주면 가독성이 높은 보고서가 됩니다.

> 과제 팁2 : 저장소 → 스토리지 → CDN을 통해 정적파일을 배포하는 방식을 이해하지 못하면 다양한 기술 문서를 이해하지 못합니다. 링크로 첨부한 문서를 보고 실무에서 이런 네트워크 지식이 어떻게 쓰이는지 맛보기 해보세요.

## 1. 개요

본 보고서는 정적 웹 사이트 배포 시 Amazon CloudFront 도입 전후의 성능을 비교 분석한 결과입니다.

## 2. 테스트 환경

- **테스트 도구:** Chrome DevTools Network 탭
- **측정 방식:** 시크릿 모드에서 4회 반복 측정
- **테스트 대상:**
  - CDN 도입 전: S3 버킷 직접 호스팅
  - CDN 도입 후: CloudFront 배포

## 3. 성능 측정 결과

### 3.1 네트워크 요청 분석

네트워크 탭을 통해 컨텐츠의 전반적인 파일 사이즈와 응답속도를 비교했습니다.

<img src="/public/네트워크 탭 비교.png" />

| 이름                         | 도입전 크기 | 도입 후 크기 | 크기 개선율 | 도입 전 시간 | 도입 후 시간 | 시간 개선율 |
| ---------------------------- | ----------- | ------------ | ----------- | ------------ | ------------ | ----------- |
| text/html                    | 12.4 kB     | 3.2 kB       | **74.19%**  | 330 ms       | 44 ms        | **86.67%**  |
| 4473ecc91f70f139-s.p.woff    | 66.8 kB     | 66.7 kB      | **0.15%**   | 383 ms       | 107 ms       | **72.06%**  |
| 463dafcda517f24f-s.p.woff    | 68.4 kB     | 68.3 kB      | **0.15%**   | 700 ms       | 150 ms       | **78.57%**  |
| 80e6897492cf7ff8.css         | 9.1 kB      | 2.9 kB       | **68.13%**  | 203 ms       | 49 ms        | **75.86%**  |
| Imagenext.svg                | 1.8 kB      | 1.1 kB       | **38.89%**  | 380 ms       | 102 ms       | **73.16%**  |
| webpack-2afed4d10bc16fb2.js  | 3.8 kB      | 2.0 kB       | **47.37%**  | 375 ms       | 101 ms       | **73.07%**  |
| 0759e794-dd46e11c5c9e6178.js | 167 kB      | 50.4 kB      | **69.82%**  | 973 ms       | 70 ms        | **92.81%**  |
| 743-5eb21586de6d8583.js      | 182 kB      | 43.5 kB      | **76.10%**  | 861 ms       | 55 ms        | **93.61%**  |
| main-app-3f41541ca5513008.js | 887 B       | 815 B        | **8.11%**   | 284 ms       | 55 ms        | **80.63%**  |
| 75-5881d6c7f834e27b.js       | 14.4 kB     | 5.5 kB       | **61.81%**  | 288 ms       | 50 ms        | **82.64%**  |
| page-e789aedd5130169a.js     | 619 B       | 548 B        | **11.48%**  | 184 ms       | 50 ms        | **72.83%**  |
| Imagevercel.svg              | 550 B       | 481 B        | **12.55%**  | 498 ms       | 49 ms        | **90.16%**  |
| Imagefile.svg                | 813 B       | 744 B        | **8.49%**   | 498 ms       | 50 ms        | **89.96%**  |
| Imagewindow.svg              | 807 B       | 737 B        | **8.68%**   | 281 ms       | 51 ms        | **81.85%**  |
| Imageglobe.svg               | 1.5 kB      | 900 B        | **40.00%**  | 498 ms       | 44 ms        | **91.16%**  |
| Otherfavicon.ico             | 26.4 kB     | 26.3 kB      | **0.38%**   | 208 ms       | 48 ms        | **76.92%**  |
| js.js                        | 1.3 kB      | 1.3 kB       | **0.00%**   | 4 ms         | 2 ms         | **100.00%** |

→ CDN 도입 후(오른쪽)의 파일 사이즈와 응답속도가 전반적으로 개선된 것을 확인할 수 있습니다.

### 3.2 주요 성능 지표 비교

[네트워크 패널 하단의 상대표시줄](https://developer.chrome.com/docs/devtools/network/reference?utm_source=devtools&hl=ko#total-number)을 통해 주요 성능 지표를 비교했습니다.

<img src="/public/네트워크 탭 하단 상태표시줄 비교.png" />

이를 표로 나타내면 다음과 같습니다.

| 지표                  | 도입 전 | 도입 후 | 개선율     |
| --------------------- | ------- | ------- | ---------- |
| 전송된 리소스 총 크기 | 564 kB  | 282 kB  | 50% 감소   |
| 완료 시간             | 7.02초  | 6.49초  | 7.5% 감소  |
| DOMContentLoaded      | 540ms   | 99ms    | 81.7% 감소 |
| 로드 시간             | 1.21초  | 210ms   | 82.6% 감소 |

→ 모든 지표에서 성능이 개선되었으며, 특히 로드 시간의 경우 82.6% 감소하였습니다.

### 3.3 응답 헤더 분석

<img src="/public/응답 헤더 비교.png" />

**주요 개선사항:**

1. **컨텐츠 압축 적용**: Content-Encoding 헤더를 통해 [Brotli(Content-Encoding: br)](https://yozm.wishket.com/magazine/detail/1739/) 압축 확인
2. **캐시 적용**: X-Cache 헤더 존재로 캐시 동작 확인

### 3.4 상세 응답 시간 분석

<img src="/public/요청의 타이밍 분석 비교.png" />

- 도입 전: 331.41ms
- 도입 후: 45.16ms

#### 시크릿 창에서 4회 반복 측정 결과:

**도입 전 측정 결과**
<img src="/public/요청의 타이밍 분석 - 도입 전.png" />

- 1차: 339.79ms
- 2차: 179.09ms
- 3차: 168.56ms
- 4차: 323.13ms
- **평균: 227.14ms**

**도입 후 측정 결과**
<img src="/public/요청의 타이밍 분석 - 도입 후.png" />

- 1차: 53.92ms
- 2차: 13.25ms
- 3차: 20.05ms
- 4차: 13.90ms
- **평균: 25.55ms**

→ 응답 시간 88.8% 개선

## 4. 결론

CloudFront CDN 도입으로 다음과 같은 주요 성능 개선을 달성했습니다:

1. 페이지 로드 시간 82.6% 감소
2. 전송 데이터 크기 50% 감소
3. 평균 응답 시간 88.8% 개선

---

# 참고 자료

## 공식문서

- [Amazon S3 사용 설명서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Amazon CloudFront 개발자 안내서](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [IAM이란 무엇입니까?](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/introduction.html)
- [GitHub Actions에 대한 워크플로 구문](https://docs.github.com/ko/actions/writing-workflows/workflow-syntax-for-github-actions#about-yaml-syntax-for-workflows)

## 블로그 및 기타소스

- [CI/CD란?](https://www.redhat.com/ko/topics/devops/what-is-ci-cd)
- [20+ Best CI/CD Tools for DevOps in 2024](https://spacelift.io/blog/ci-cd-tools)
- [프론트엔드 CI/CD 파이프라인 구축](https://velog.io/@taegon1998/%ED%94%84%EB%A1%A0%ED%8A%B8%EC%97%94%EB%93%9C-CICD-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EA%B5%AC%EC%B6%95)
- [팀에 맞는 프론트엔드 CI/CD 파이프라인 설계 및 배포 자동화 구축하기](https://medium.com/@dlxotjde_87064/%ED%94%84%EB%A1%A0%ED%8A%B8%EC%97%94%EB%93%9C-ci-cd-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EC%84%A4%EA%B3%84-%EB%B0%8F-%EB%B0%B0%ED%8F%AC-%EC%9E%90%EB%8F%99%ED%99%94-c84c973ce45d)
