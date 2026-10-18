# S3 스토리지 설정 · 설치 · 사용방법 가이드

Amazon S3(Simple Storage Service)는 객체 스토리지 서비스입니다. 이 문서는 버킷 생성 → AWS CLI 설치 → 자격 증명 설정 → 업로드/다운로드 → 보안 설정 순서로 정리한 가이드입니다.

## 1. S3 기본 개념

| 용어 | 설명 |
|---|---|
| 버킷(Bucket) | 객체를 담는 최상위 컨테이너. 이름은 전 세계에서 유일해야 하며 소문자, 숫자, 하이픈만 사용 (3~63자) |
| 객체(Object) | 저장되는 파일과 메타데이터. 단일 객체 최대 크기는 5TB |
| 키(Key) | 버킷 내 객체의 고유 경로 (예: `images/2026/logo.png`). 폴더처럼 보이지만 실제로는 접두사(prefix) |
| 리전(Region) | 버킷이 위치하는 물리적 지역. 서울 리전은 `ap-northeast-2` |

### 스토리지 클래스

| 클래스 | 용도 |
|---|---|
| S3 Standard | 자주 접근하는 데이터 (기본값) |
| S3 Intelligent-Tiering | 접근 패턴을 알 수 없거나 변할 때 자동으로 계층 이동 |
| S3 Standard-IA / One Zone-IA | 자주 접근하지 않지만 필요 시 즉시 접근해야 하는 데이터 |
| S3 Glacier 계열 | 장기 보관 및 아카이브용 (복원 시간과 비용 확인 필요) |

## 2. 사전 준비

- [ ] AWS 계정 생성
- [ ] 루트 계정에 MFA(다중 인증) 설정
- [ ] 작업용 IAM 사용자 또는 IAM Identity Center(SSO) 사용자 생성
- [ ] 사용할 리전 결정 (예: 서울 `ap-northeast-2`)

> ⚠️ 루트 계정의 액세스 키는 만들지 않는 것이 원칙입니다. 일상 작업은 반드시 IAM 사용자 또는 SSO 역할로 진행하세요.

## 3. 버킷 생성

### 3-1. 콘솔에서 생성

1. AWS 콘솔 로그인 후 서비스 검색창에서 S3 선택
2. 우측 상단에서 리전 확인 (예: 아시아 태평양 서울)
3. 버킷 만들기 클릭
4. 버킷 이름 입력 (전역 고유)
5. 객체 소유권: ACL 비활성화 (권장, 기본값)
6. 퍼블릭 액세스 차단: 모든 퍼블릭 액세스 차단 유지 (권장, 기본값)
7. 버킷 버전 관리: 필요 시 활성화
8. 기본 암호화: SSE-S3 (기본값) 또는 SSE-KMS
9. 버킷 만들기 클릭

### 3-2. CLI로 생성

```bash
aws s3 mb s3://my-example-bucket-12345 --region ap-northeast-2
```

## 4. AWS CLI 설치

### Windows

```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

### macOS

```bash
brew install awscli
```

### Linux (x86_64)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 설치 확인

```bash
aws --version
```

## 5. 자격 증명 설정

### 방법 A. IAM 사용자 액세스 키 (간단)

1. IAM 콘솔 → 사용자 → 보안 자격 증명 → 액세스 키 만들기 (CLI 용도 선택)
2. 터미널에서 설정

```bash
aws configure
# AWS Access Key ID: <ACCESS_KEY_ID>
# AWS Secret Access Key: <SECRET_ACCESS_KEY>
# Default region name: ap-northeast-2
# Default output format: json
```

설정 파일은 홈 디렉터리의 `.aws/credentials` 와 `.aws/config` 에 저장됩니다.

### 방법 B. IAM Identity Center(SSO) (권장)

```bash
aws configure sso
aws sso login --profile my-profile
```

### 연결 확인

```bash
aws sts get-caller-identity
aws s3 ls
```

> 🔒 액세스 키와 시크릿 키는 코드, 깃 저장소, 문서에 절대 그대로 적지 마세요. 노출되면 즉시 키를 비활성화하고 재발급해야 합니다.

## 6. 기본 사용법 (CLI)

### 버킷 및 객체 조회

```bash
aws s3 ls                                   # 버킷 목록
aws s3 ls s3://my-example-bucket-12345      # 버킷 내 객체
aws s3 ls s3://my-example-bucket-12345/images/ --recursive --human-readable
```

### 업로드 / 다운로드

```bash
# 업로드
aws s3 cp ./report.pdf s3://my-example-bucket-12345/docs/report.pdf

# 다운로드
aws s3 cp s3://my-example-bucket-12345/docs/report.pdf ./report.pdf

# 폴더 전체 업로드
aws s3 cp ./images s3://my-example-bucket-12345/images --recursive
```

### 동기화 (sync)

```bash
# 로컬 → S3
aws s3 sync ./site s3://my-example-bucket-12345/site

# S3 → 로컬
aws s3 sync s3://my-example-bucket-12345/site ./site

# 원본에 없는 파일은 대상에서 삭제 (주의)
aws s3 sync ./site s3://my-example-bucket-12345/site --delete
```

### 삭제 / 이동

```bash
aws s3 rm s3://my-example-bucket-12345/docs/report.pdf
aws s3 rm s3://my-example-bucket-12345/tmp/ --recursive
aws s3 mv s3://my-example-bucket-12345/a.txt s3://my-example-bucket-12345/b.txt

# 버킷 삭제 (비어 있어야 함, 내용물 포함 삭제는 --force)
aws s3 rb s3://my-example-bucket-12345 --force
```

### 스토리지 클래스 지정 업로드

```bash
aws s3 cp ./archive.zip s3://my-example-bucket-12345/archive/ --storage-class STANDARD_IA
```

### 임시 공유 링크 (사전 서명 URL)

```bash
aws s3 presign s3://my-example-bucket-12345/docs/report.pdf --expires-in 3600
```

버킷을 공개하지 않고도 지정한 시간(초) 동안만 접근 가능한 링크를 만들 수 있습니다.

## 7. 보안 및 권한 설정

### 7-1. 퍼블릭 액세스 차단

```bash
aws s3api put-public-access-block \
  --bucket my-example-bucket-12345 \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### 7-2. IAM 최소 권한 정책 예시

특정 버킷에만 읽기/쓰기/삭제를 허용하는 정책입니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::my-example-bucket-12345"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-example-bucket-12345/*"
    }
  ]
}
```

### 7-3. 버전 관리 활성화

```bash
aws s3api put-bucket-versioning \
  --bucket my-example-bucket-12345 \
  --versioning-configuration Status=Enabled
```

실수로 삭제하거나 덮어쓴 객체를 복구할 수 있습니다.

### 7-4. 암호화

- 신규 버킷은 기본적으로 SSE-S3(AES-256)로 서버 측 암호화가 적용됩니다.
- 키를 직접 관리해야 하면 SSE-KMS를 선택합니다.
- 전송 구간은 HTTPS만 허용하도록 버킷 정책에서 `aws:SecureTransport` 조건으로 강제할 수 있습니다.

### 7-5. 수명 주기 규칙 (비용 절감)

콘솔 경로: 버킷 → 관리(Management) → 수명 주기 규칙 만들기

- 일정 기간 후 Standard-IA 또는 Glacier로 자동 전환
- 오래된 객체 및 이전 버전 자동 삭제
- 미완료 멀티파트 업로드 자동 정리

## 8. SDK 사용 예시 (Python boto3)

```bash
pip install boto3
```

```python
import boto3

s3 = boto3.client("s3", region_name="ap-northeast-2")
bucket = "my-example-bucket-12345"

# 업로드
s3.upload_file("report.pdf", bucket, "docs/report.pdf")

# 다운로드
s3.download_file(bucket, "docs/report.pdf", "report_copy.pdf")

# 목록 조회
resp = s3.list_objects_v2(Bucket=bucket, Prefix="docs/")
for obj in resp.get("Contents", []):
    print(obj["Key"], obj["Size"])

# 사전 서명 URL (1시간)
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": bucket, "Key": "docs/report.pdf"},
    ExpiresIn=3600,
)
print(url)
```

자격 증명은 코드에 넣지 않고 `aws configure` 설정, 환경 변수, 또는 EC2/Lambda의 IAM 역할에서 자동으로 읽어오게 합니다.

## 9. 정적 웹사이트 호스팅 (참고)

1. 버킷의 속성 → 정적 웹 사이트 호스팅 활성화, 인덱스 문서(`index.html`) 지정
2. 직접 공개하는 방식보다 CloudFront + OAC(Origin Access Control) 조합을 권장 (버킷은 비공개 유지)
3. 파일 업로드 후 CloudFront 배포 도메인으로 접속 확인

## 10. 비용 및 운영 팁

- 요금 구성: 저장 용량, 요청 수(PUT/GET 등), 데이터 전송(인터넷 방향 아웃바운드) 비용
- 같은 리전의 AWS 서비스 간 전송은 대체로 저렴하거나 무료
- 버킷 삭제 전에 버전 관리된 이전 버전과 삭제 마커까지 모두 지워야 비어 있는 상태가 됨
- 대용량 파일은 CLI가 자동으로 멀티파트 업로드 처리
- 예산 알림(AWS Budgets)을 설정해 예상치 못한 과금을 방지

## 11. 자주 겪는 오류

| 오류 | 원인 / 해결 |
|---|---|
| AccessDenied | IAM 정책 또는 버킷 정책 권한 부족. `ListBucket`은 버킷 ARN, 객체 작업은 버킷 ARN 뒤에 `/*`를 붙인 ARN에 부여해야 함 |
| BucketAlreadyExists | 버킷 이름이 이미 사용 중. 고유한 이름으로 변경 |
| NoSuchBucket | 버킷 이름 오타 또는 리전 불일치. `--region` 옵션 확인 |
| InvalidAccessKeyId / SignatureDoesNotMatch | 키 오타, 비활성화된 키, 또는 시스템 시간 오차. `aws configure` 재설정 |
| ExpiredToken | SSO 또는 임시 자격 증명 만료. `aws sso login` 으로 재로그인 |

## 12. 참고 링크

- [Amazon S3 사용 설명서](https://docs.aws.amazon.com/s3/)
- [AWS CLI S3 명령어 레퍼런스](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [AWS CLI 설치 가이드](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Amazon S3 요금](https://aws.amazon.com/s3/pricing/)
