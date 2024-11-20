# 프론트엔드 배포 파이프라인

## 목차

- [프론트엔드 배포 파이프라인](#프론트엔드-배포-파이프라인)
  - [목차](#목차)
  - [주요 링크](#주요-링크)
  - [개요](#개요)
    - [GitHub Actions 워크플로우](#github-actions-워크플로우)
  - [주요 개념](#주요-개념)
    - [GitHub Actions과 CI/CD 도구](#github-actions과-cicd-도구)
    - [S3와 스토리지](#s3와-스토리지)
    - [CloudFront와 CDN](#cloudfront와-cdn)
    - [캐시 무효화(Cache Invalidation)](#캐시-무효화cache-invalidation)
    - [Repository secret과 환경변수](#repository-secret과-환경변수)
  - [CDN과 성능최적화](#cdn과-성능최적화)

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghae99.s3-website-ap-southeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름: https://d2xvtvxlifkv9.cloudfront.net

## 개요

<img src="/public/diagram.png" />

1. 개발자가 코드 푸시
2. GitHub Actions에서 자동 빌드 및 테스트
   - AWS IAM
     - GitHub Actions에서 AWS 리소스에 접근할 권한 부여
3. S3에 빌드 결과물 업로드
   - AWS IAM
     - S3 버킷에 대한 쓰기 권한 제공
     - CloudFront 배포 설정에 대한 접근 권한 관리
4. CloudFront를 통해 사용자에게 배포

### GitHub Actions 워크플로우

[GitHub Actions 워크플로우](https://docs.github.com/ko/actions/writing-workflows/quickstart)를 작성해 다음과 같이 배포가 진행되도록 합니다.

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
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
       aws-region: ${{ secrets.AWS_REGION }}
   ```

   - `AWS_ACCESS_KEY_ID`: IAM 계정 생성시 발급받은 액세스 키
   - `AWS_SECRET_ACCESS_KEY`: IAM 계정 생성시 발급받은 비밀 액세스 키
   - `AWS_REGION`: S3를 세팅한 리전(지역)의 코드

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

## 주요 개념

### GitHub Actions과 CI/CD 도구

GitHub Actions는 자동화된 워크플로우를 생성하여 CI/CD(Continuous Integration/Continuous Deployment)를 관리할 수 있는 도구입니다.
이를 통해 코드 변경 시마다 자동으로 빌드, 테스트, 배포를 수행합니다.
GitHub Actions는 배포 파이프라인을 관리하는 데 중요한 역할을 합니다.

### S3와 스토리지

AWS S3는 정적 파일을 저장하고 제공하는 스토리지 서비스입니다.
이 서비스를 사용하여 빌드된 파일들을 안전하게 저장하고, 클라이언트에 제공할 수 있습니다.

### CloudFront와 CDN

CloudFront는 AWS의 콘텐츠 전송 네트워크(CDN) 서비스로, 전 세계에 분산된 서버를 통해 파일을 빠르게 전달합니다.
CloudFront는 파일 캐시를 관리하고, 사용자가 파일을 더 빠르게 접근할 수 있도록 최적화합니다.

### 캐시 무효화(Cache Invalidation)

CloudFront의 캐시를 무효화하는 과정은 배포 후 반드시 수행되어야 합니다.
새로 배포된 파일이 캐시로 인해 사용자에게 전달되지 않는 문제를 방지하고, 최신 파일을 제공할 수 있도록 합니다.
이 과정은 자동화된 배포 프로세스의 중요한 부분입니다.

### Repository secret과 환경변수

GitHub에서 제공하는 `repository secret`과 `환경 변수`를 사용하여 민감한 정보를 안전하게 관리할 수 있습니다.
이를 통해 AWS 자격 증명, API 키 등 중요한 정보를 코드에 포함시키지 않고 안전하게 저장하고 사용할 수 있습니다.

> 과제 팁1: 다이어그램 작성엔 Draw.io, [Lucidchart](https://lucid.app/documents#/documents?folder_id=home) 등을 이용합니다.

> 과제 팁2: 새로운 프로젝트 진행시, 프론트엔드팀 리더는 예시에 있는 다이어그램을 준비한 후, 전사 회의에 들어가 발표하게 됩니다. 미리 팀장이 되었다 생각하고 아키텍쳐를 도식화 하는걸 연습해봅시다.

> 과제 팁3: 캐시 무효화는 배포와 장애 대응에 중요한 개념입니다. .github/workflows/deployment.yml 에서 캐시 무효화가 어떤 시점에 동작하는지 보고, 추가 리서치를 통해 반드시 개념을 이해하고 넘어갑시다.

> 과제 팁4: 상용 프로젝트에선 DNS 서비스가 필요합니다. 도메인 구입이 필요해 본 과제에선 ‘Route53’을 붙이는걸 하지 않았지만, 실무에선 다음과 같은 인프라 구성이 필요하다는걸 알아둡시다

## CDN과 성능최적화

(CDN 도입 전과 도입 후의 성능 개선 보고서 작성)

> 과제 팁1 : CDN 도입후 성능 개선 보고서 영역은 [프론트엔드 개발자를 위한 CloudFront](https://sprout-log-68d.notion.site/CloudFront-2c0653cb130f42b2b21078389511cca2) 에서 네트워크 탭을 비교한 영역을 참고해주세요. 이미지와 수치등을 표로 정리해서 보여주면 가독성이 높은 보고서가 됩니다.

> 과제 팁2 : 저장소 → 스토리지 → CDN을 통해 정적파일을 배포하는 방식을 이해하지 못하면 다양한 기술 문서를 이해하지 못합니다. 링크로 첨부한 문서를 보고 실무에서 이런 네트워크 지식이 어떻게 쓰이는지 맛보기 해보세요.
