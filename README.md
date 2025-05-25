# 프론트엔드 배포 파이프라인

## 개요

![Diagram](https://github.com/user-attachments/assets/f042282d-285e-4c00-837d-0591bc4b9c65)

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다. (.github/workflows/deployment.yml)

### Workflow

#### 트리거 조건

```yml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

- `main` 브랜치에 코드가 푸시될 때 자동 실행
- `workflow_dispatch로` GitHub 웹 인터페이스에서 수동 실행도 가능

#### 실행 환경

```yml
runs-on: ubuntu-latest
```

- Ubuntu 최신 버전 환경에서 실행

#### 주요 단계별 설명

1. 코드 체크아웃

```yml
yaml- name: Checkout repository
  uses: actions/checkout@v4
```

- 저장소의 소스 코드를 작업 환경으로 다운로드

2. 의존성 설치

```yml
yaml- name: Install dependencies
  run: npm ci
```

- `npm ci`로 `package-lock.json`을 기준으로 정확한 버전의 패키지들을 설치
- `npm install`보다 빠르고 안정적

3. 프로젝트 빌드

```yml
yaml- name: Build
  run: npm run build
```

- Next.js 애플리케이션을 정적 파일로 빌드
- 빌드 결과물은 `out/` 디렉토리에 생성됨 (정적 익스포트 설정 필요)

4. AWS 인증 설정

```yml
yaml- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```

- GitHub Secrets에 저장된 AWS 자격 증명으로 AWS CLI 설정

5. S3 배포

```yml
yaml- name: Deploy to S3
  run: |
    aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
```

- `out/` 디렉토리의 정적 파일들을 S3 버킷에 동기화
- `--delete` 옵션으로 S3에서 제거된 파일들도 삭제

6. CloudFront 캐시 무효화

```yml
yaml- name: Invalidate CloudFront cache
  run: |
    aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

- CloudFront 캐시를 무효화하여 새로운 콘텐츠가 즉시 반영되도록 함
- `"/*"`로 모든 경로의 캐시를 무효화

### 필요한 GitHub Secrets

- 저장소 설정에서 다음 시크릿들을 추가해야 합니다:

- `AWS_ACCESS_KEY_ID`: AWS 액세스 키
- `AWS_SECRET_ACCESS_KEY`: AWS 시크릿 키
- `AWS_REGION`: AWS 리전 (예: ap-northeast-2)
- `S3_BUCKET_NAME`: S3 버킷 이름
- `CLOUDFRONT_DISTRIBUTION_ID`: CloudFront 배포 ID

### 주의사항

- Next.js 설정: 정적 익스포트를 위해 `next.config.js`에 `output: 'export'` 설정이 필요합니다.
- S3 권한: IAM 사용자에게 S3와 CloudFront 권한이 필요합니다.
- 비용: CloudFront 무효화는 월 1,000개까지 무료이므로 자주 배포하면 비용이 발생할 수 있습니다.

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: [S3](http://mphnwsbucket01.s3-website-ap-southeast-2.amazonaws.com/)
- CloudFront 배포 도메인 이름: [CloudFront](https://d16u1u2oc0oup8.cloudfront.net/)

## 주요 개념

> GitHub Actions, AWS S3, CloudFront, 캐시 무효화, Repository Secrets의 핵심 개념과 실무 활용법

## 1. GitHub Actions와 CI/CD 도구

### CI/CD란?

**CI (Continuous Integration)**는 개발자들이 코드 변경사항을 자주 메인 브랜치에 통합하는 개발 방법론입니다. **CD (Continuous Deployment/Delivery)**는 코드 변경사항을 자동으로 프로덕션 환경에 배포하는 과정입니다.

### GitHub Actions의 핵심 개념

**워크플로우(Workflow)**

- `.github/workflows/` 디렉토리에 YAML 파일로 정의
- 특정 이벤트(push, pull request 등)에 의해 트리거
- 하나 이상의 Job으로 구성

**Job과 Step**

- Job: 동일한 러너(runner)에서 실행되는 단계들의 집합
- Step: 개별 작업 단위 (명령어 실행 또는 액션 사용)
- 여러 Job은 병렬 실행 가능

**러너(Runner)**

- 워크플로우를 실행하는 서버 환경
- GitHub 호스팅 러너: `ubuntu-latest`, `windows-latest`, `macos-latest`
- 셀프 호스팅 러너: 자체 서버에서 실행

**액션(Actions)**

- 재사용 가능한 코드 블록
- GitHub Marketplace에서 공개 액션 사용 가능
- 예: `actions/checkout@v4`, `aws-actions/configure-aws-credentials@v1`

### CI/CD의 장점

- 버그 조기 발견 및 수정
- 배포 과정 자동화로 인한 인적 오류 감소
- 빠른 피드백 루프
- 안정적이고 일관된 배포 환경

## 2. S3와 스토리지

### Amazon S3란?

Amazon Simple Storage Service(S3)는 AWS의 객체 스토리지 서비스입니다. 웹 애플리케이션, 백업, 데이터 아카이브 등 다양한 용도로 사용됩니다.

### S3의 핵심 개념

**버킷(Bucket)**

- S3의 최상위 컨테이너
- 전역적으로 고유한 이름 필요
- 특정 AWS 리전에 생성
- 정적 웹사이트 호스팅 기능 제공

**객체(Object)**

- S3에 저장되는 개별 파일
- 키(Key): 객체의 고유 식별자 (파일 경로)
- 메타데이터: 객체에 대한 추가 정보

**스토리지 클래스**

- Standard: 자주 액세스하는 데이터용
- IA (Infrequent Access): 가끔 액세스하는 데이터용
- Glacier: 아카이브용 (검색 시간 길음)
- One Zone-IA: 단일 가용 영역에 저장

### 정적 웹사이트 호스팅

- HTML, CSS, JavaScript 파일을 직접 제공
- 서버 없이 정적 콘텐츠 호스팅 가능
- 도메인 연결 및 SSL 인증서 적용 가능

### S3 동기화

```bash
aws s3 sync local-folder/ s3://bucket-name/ --delete
```

- 로컬 폴더와 S3 버킷 간 파일 동기화
- `--delete`: 대상에 없는 파일 삭제
- 변경된 파일만 업로드하여 효율적

## 3. CloudFront와 CDN

### CDN(Content Delivery Network)이란?

CDN은 전 세계에 분산된 서버 네트워크를 통해 사용자에게 콘텐츠를 빠르게 전달하는 서비스입니다. 사용자와 가장 가까운 서버에서 콘텐츠를 제공하여 로딩 속도를 향상시킵니다.

### CloudFront의 핵심 개념

**배포(Distribution)**

- CloudFront의 설정 단위
- 원본 서버(Origin)와 CDN 설정을 정의
- 고유한 도메인 이름 제공 (예: `d123456abcdef8.cloudfront.net`)

**엣지 로케이션(Edge Location)**

- 전 세계에 분산된 캐시 서버
- 사용자와 가까운 곳에서 콘텐츠 제공
- 400개 이상의 글로벌 엣지 로케이션

**원본 서버(Origin)**

- 실제 콘텐츠가 저장된 서버
- S3 버킷, EC2 인스턴스, 온프레미스 서버 등
- 여러 원본 서버 설정 가능

**TTL(Time To Live)**

- 캐시 만료 시간
- 기본 TTL, 최대 TTL, 최소 TTL 설정 가능
- HTTP 헤더로 개별 파일의 캐시 시간 제어

### CloudFront의 장점

- 빠른 콘텐츠 전달 속도
- 원본 서버 부하 감소
- DDoS 공격 방어
- SSL/TLS 암호화 지원
- 실시간 로그 및 모니터링

## 4. 캐시 무효화(Cache Invalidation)

### 캐시 무효화란?

CDN에 저장된 캐시를 강제로 삭제하여 새로운 콘텐츠를 즉시 반영하는 과정입니다.

### 캐시 무효화가 필요한 경우

- 웹사이트 업데이트 후 즉시 반영 필요
- 잘못된 콘텐츠 수정
- 긴급 보안 패치 적용
- 마케팅 캠페인 등 시점이 중요한 콘텐츠

### CloudFront 무효화 방법

**AWS CLI 사용**

```bash
aws cloudfront create-invalidation \
  --distribution-id E1PA6795UKMFR9 \
  --paths "/*"
```

**특정 경로 무효화**

```bash
aws cloudfront create-invalidation \
  --distribution-id E1PA6795UKMFR9 \
  --paths "/index.html" "/css/*" "/js/app.js"
```

**와일드카드 사용**

- `/*`: 모든 파일 무효화
- `/images/*`: images 폴더의 모든 파일
- `*.css`: 모든 CSS 파일

### 무효화 비용 및 제한사항

- 월 1,000개 경로까지 무료
- 1,000개 초과 시 경로당 $0.005 요금
- 와일드카드 패턴도 1개 경로로 계산
- 무효화 완료까지 10-15분 소요

### 대안 방법

**파일명 버전 관리**

```html
<link rel="stylesheet" href="/css/style.css?v=1.2.3" />
<script src="/js/app.js?v=2024.01.15"></script>
```

**해시 기반 파일명**

```
app.a1b2c3d4.js
style.e5f6g7h8.css
```

## 5. Repository Secret과 환경변수

### GitHub Secrets란?

GitHub 저장소에 안전하게 저장되는 암호화된 환경변수입니다. API 키, 비밀번호, 토큰 등 민감한 정보를 저장하는 데 사용합니다.

### Secrets의 종류

**Repository Secrets**

- 특정 저장소에서만 사용 가능
- 저장소 Settings > Secrets and variables > Actions에서 관리
- 워크플로우에서 `${{ secrets.SECRET_NAME }}`으로 접근

**Organization Secrets**

- 조직 전체 저장소에서 사용 가능
- 조직 단위로 관리
- 저장소별 접근 권한 설정 가능

**Environment Secrets**

- 특정 환경(production, staging 등)에서만 사용
- 환경별 배포 승인 프로세스 적용 가능

### 환경변수 사용법

**Secrets 설정**

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```

**일반 환경변수**

```yaml
env:
  NODE_ENV: production
  API_URL: https://api.example.com

steps:
  - name: Build application
    run: npm run build
    env:
      REACT_APP_API_KEY: ${{ secrets.REACT_APP_API_KEY }}
```

**조건부 환경변수**

```yaml
- name: Set environment variables
  run: |
    if [ "${{ github.ref }}" == "refs/heads/main" ]; then
      echo "ENVIRONMENT=production" >> $GITHUB_ENV
    else
      echo "ENVIRONMENT=staging" >> $GITHUB_ENV
    fi
```

### 보안 모범 사례

**최소 권한 원칙**

- 필요한 최소한의 권한만 부여
- AWS IAM 사용자는 특정 리소스에만 접근 가능하도록 설정

**정기적인 키 순환**

- API 키와 액세스 키를 정기적으로 교체
- 만료 기간이 있는 토큰 사용 권장

**환경별 분리**

- 개발, 스테이징, 프로덕션 환경별로 다른 키 사용
- 환경별 Secret 분리 관리

**로그 보안**

- Secrets는 GitHub Actions 로그에서 자동으로 마스킹됨
- `echo ${{ secrets.SECRET }}`와 같은 직접 출력 금지

### 환경변수 디버깅

```yaml
- name: Debug environment
  run: |
    echo "Current branch: ${{ github.ref }}"
    echo "Environment: $ENVIRONMENT"
    echo "Node version: $(node --version)"
    # Secret 값은 출력하지 않음!
```

---

### 전체 플로우

- 개발자가 코드 변경 → GitHub에 push
- GitHub Actions 트리거 → CI/CD 파이프라인 시작
- 코드 빌드 및 테스트 → 정적 파일 생성
- S3에 배포 → 정적 호스팅 서버에 파일 업로드
- CloudFront 캐시 무효화 → 전 세계 사용자에게 즉시 반영
- Repository Secrets 활용 → 안전한 AWS 인증

### 실무에서 고려할 점

#### 성능 최적화

- ₩CloudFront TTL 설정으로 캐시 효율성 향상
- 파일 압축 및 이미지 최적화
- 불필요한 캐시 무효화 최소화

#### 보안 강화

- IAM 역할 기반 접근 제어
- 환경별 Secret 분리
- 정기적인 액세스 키 교체

#### 모니터링

- CloudWatch를 통한 성능 모니터링
- GitHub Actions 로그 분석
- 배포 실패 시 알림 설정

#### 비용 관리

- S3 스토리지 클래스 최적화
- CloudFront 무효화 횟수 제한
- 불필요한 워크플로우 실행 방지
