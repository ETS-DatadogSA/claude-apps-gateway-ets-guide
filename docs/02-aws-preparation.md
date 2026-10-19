# 2. AWS 배포 준비

배포용 소스를 받고, Region·자격증명을 고정하고, 배포를 막는 두 가지(Bedrock 추론 프로파일, Aurora 버전)를 미리 확인합니다.

## 2.1 Repository clone

배포에 쓸 소스를 새 폴더로 받습니다. `gateway/`·`admin-console/` 는 로컬 트리에서 그대로 패키징되므로, 다른 수정이 섞이지 않은 새 트리를 씁니다.

### Repository clone 명령어
```bash
git clone https://github.com/ETS-DatadogSA/claude-apps-gateway-ets-guide.git && cd claude-apps-gateway-ets-guide
```

### Repository clone 결과 예시
```text
Cloning into 'claude-apps-gateway-ets-guide'...
```

> [!TIP]
> 같은 이름의 폴더가 이미 있으면 `fatal: destination path ... already exists` 로 실패합니다. `git clone <URL> <새 폴더 이름> && cd <새 폴더 이름>` 처럼 폴더 이름을 지정합니다.

## 2.2 의존성과 번들러

CDK 가 Lambda 4개를 번들링할 때 로컬 `esbuild` 를 쓰고, 없으면 Docker 를 씁니다. Docker 를 쓰지 않는다면 `esbuild` 를 설치합니다.

### 의존성 설치 명령어
```bash
cd cdk && npm install && npm install --save-dev esbuild@^0.21 && npx esbuild --version && cd ..
```

### 의존성 설치 결과 예시
```text
added 2 packages, and audited 352 packages in 1s
...
38 vulnerabilities (4 moderate, 33 high, 1 critical)
...
npm warn install-scripts   esbuild@0.21.5 (postinstall: node install.js)
...
0.21.5
```

마지막 줄에 `0.21.x` 가 나오면 성공입니다. 중간의 `vulnerabilities`·`npm warn` 은 배포에 영향이 없으므로 그대로 진행합니다.

<details>
<summary>경고 문구와 esbuild 가 필요한 이유</summary>

- `vulnerabilities`: 원본 리포 의존성에 대한 `npm audit` 요약입니다. `npm audit fix` 는 실행하지 않습니다.
- `install-scripts … not yet covered by allowScripts`: npm 이 설치 스크립트를 기본으로 막아서 나오는 경고입니다. esbuild 실행 파일은 `@esbuild/<플랫폼>` 패키지로 따로 설치되므로 스크립트가 막혀도 동작합니다.
- 이 명령은 `cdk/package.json`·`package-lock.json` 에 esbuild 를 추가합니다. 배포용 트리에서만 쓰고 커밋하지 않습니다.
- 컨테이너 이미지는 AWS 안의 임시 EC2 가 빌드하므로 로컬 Docker 가 필요 없습니다. 하지만 Lambda 번들링(`NodejsFunction`)은 로컬에서 하고, `cdk/package.json` 에는 `esbuild` 가 없습니다. 버전 `^0.21` 은 CDK 의 Docker 폴백과 맞춘 값입니다.
</details>

## 2.3 AWS Region과 배포 자격증명 고정

Region 을 us-east-1 로 고정하고, 배포에 쓸 자격증명을 확인합니다. 기본 Region 이 다른 환경(예: `ap-northeast-2`)이면 반드시 덮어씁니다. `AWS_PAGER=""` 는 출력이 페이저(`(END)` 화면)로 멈추지 않게 합니다.

### Region·자격증명 확인 명령어
```bash
export AWS_REGION=us-east-1 AWS_DEFAULT_REGION=us-east-1 CDK_DEFAULT_REGION=us-east-1 AWS_PAGER="" && aws sts get-caller-identity
```

### Region·자격증명 확인 결과 예시
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/<user-name>"
}
```

`Account` 가 배포할 계정이면 됩니다. `Arn` 은 IAM 사용자면 `:user/…`, 롤이면 `:assumed-role/…` 로 나옵니다.

> [!WARNING]
> 배포에 쓰는 자격증명(IAM 사용자 또는 롤)에 IAM 롤 생성 권한이 있어야 합니다. CDK 가 배포하면서 IAM 롤을 직접 만들기 때문입니다. `IAMReadOnlyAccess` 처럼 IAM 이 읽기 전용이면 4단계 bootstrap 에서 실패하므로 권한이 있는 프로필로 바꿉니다.

<details>
<summary>MFA assume 프로필을 쓰는 경우</summary>

CDK 는 `aws` CLI 의 세션 캐시를 공유하지 않습니다. CLI 세션을 환경변수로 넘깁니다.

```bash
aws sts get-caller-identity --profile <admin-profile>
```

```bash
unset AWS_PROFILE && eval "$(aws configure export-credentials --profile <admin-profile> --format env)" && export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text) && aws sts get-caller-identity
```

- 마지막 출력이 `assumed-role/...` 이어야 합니다. IAM 사용자 ARN 이 나오면 `eval` 줄을 다시 실행합니다.
- TOTP 코드는 공백 없이 여섯 자리로 넣습니다.
- 세션은 기본 1시간 뒤 만료되고 자동 갱신되지 않습니다. 배포가 25~35분이므로 **배포 직전에** 다시 인증합니다.
- 아직 유효한 세션이 캐시에 있으면 MFA 를 묻지 않고 그 세션을 그대로 돌려줍니다. 새 1시간 세션이 필요하면 캐시를 비우고 받습니다.

```bash
rm -rf ~/.aws/cli/cache && unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_CREDENTIAL_EXPIRATION && eval "$(aws configure export-credentials --profile <admin-profile> --format env)" && export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text) && aws sts get-caller-identity
```
</details>

## 2.4 Bedrock 추론 프로파일 확인

게이트웨이는 기본으로 `us.anthropic.*` 추론 프로파일로 Bedrock 을 호출합니다. 배포 Region 에 이 프로파일이 있는지 확인합니다.

### 추론 프로파일 확인 명령어
```bash
aws bedrock list-inference-profiles --query "inferenceProfileSummaries[?starts_with(inferenceProfileId,'us.anthropic')].inferenceProfileId" --output table
```

### 추론 프로파일 확인 결과 예시
```text
|              ListInferenceProfiles             |
+------------------------------------------------+
|  us.anthropic.claude-3-sonnet-20240229-v1:0    |
...
|  us.anthropic.claude-opus-5-5                  |
|  us.anthropic.claude-sonnet-5-5                |
|  us.anthropic.claude-haiku-5-5                 |
...
```

`us.anthropic.` 항목이 하나 이상 나오면 됩니다. 목록은 계정의 모델 액세스에 따라 다르고, 비어 있으면 Bedrock 콘솔의 Model access 에서 Anthropic 모델 액세스를 먼저 받습니다.

> [!WARNING]
> `us.anthropic.*` 프로파일이 없는 Region(예: `ap-northeast-2`)에서는 **배포는 성공하고 추론이 실패**합니다. application inference profile 을 만들어 쓰면 IAM 수정 없이 해결됩니다([8장](08-custom-inference-profile.md)).

<details>
<summary>IAM 이 허용하는 프로파일 범위</summary>

게이트웨이 IAM 정책은 배포 Region 의 `inference-profile/us.anthropic.*` 시스템 프로파일과, 같은 계정의 application inference profile(Region 무관)만 허용합니다. `global.anthropic.*` 같은 다른 시스템 프로파일을 쓰려면 `cdk/lib/gateway-stack.ts` 의 IAM 정책도 고쳐야 하며, 이 가이드의 범위를 벗어납니다.
</details>

## 2.5 Aurora 버전 확인

`cdk/lib/database-stack.ts` 는 Aurora PostgreSQL `VER_16_6` 으로 고정돼 있습니다. Region 에 `16.6` 이 없으면 database 스택 배포가 실패하므로 미리 확인합니다.

### Aurora 버전 확인 명령어
```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --region us-east-1 --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion" --output json
```

### Aurora 버전 확인 결과 예시
```json
[
    "16.4-limitless",
    "16.6-limitless",
    "16.8",
    ...
    "16.13",
    "16.13-limitless",
    "16.14",
    "16.15"
]
```

결과에 `"16.6"` 이 있으면 다음 장으로 넘어갑니다.

> [!WARNING]
> `"16.6"` 이 없고 `16.6-limitless` 만 있으면 아래 **버전 바꾸기**를 따릅니다. 2026-09-18 과 2026-10-19 us-east-1 에서 모두 이 경우였습니다.

<details>
<summary>버전 바꾸기 (<code>16.6</code> 이 없을 때)</summary>

`database-stack.ts` 는 CDK 상수(`VER_16_x`)로 버전을 지정하므로, Region 과 CDK 양쪽에 있는 버전이어야 합니다.

**1. CDK 가 아는 버전 확인** (리포 루트에서)

```bash
cd cdk && grep -oE "VER_16_[0-9]+" node_modules/aws-cdk-lib/aws-rds/lib/cluster-engine.d.ts | sort -u && cd ..
```

```text
VER_16_0
...
VER_16_11
VER_16_13
...
VER_16_9
```

**2. 버전 고르기**: `-limitless` 가 아닌 Region 버전 가운데 CDK 상수가 있는 가장 높은 버전을 고릅니다. 위 예시(`aws-cdk-lib 2.261.0`)에서는 `16.14`·`16.15` 의 상수가 없으므로 `16.13`(`VER_16_13`)입니다.

**3. 버전 변경** (`VER_16_13` 은 고른 버전으로)

macOS:

```bash
sed -i '' 's/AuroraPostgresEngineVersion.VER_16_6,/AuroraPostgresEngineVersion.VER_16_13,/' cdk/lib/database-stack.ts && grep -n "VER_16" cdk/lib/database-stack.ts
```

Windows Git Bash / Linux:

```bash
sed -i 's/AuroraPostgresEngineVersion.VER_16_6,/AuroraPostgresEngineVersion.VER_16_13,/' cdk/lib/database-stack.ts && grep -n "VER_16" cdk/lib/database-stack.ts
```

```text
49:        version: rds.AuroraPostgresEngineVersion.VER_16_13,
```

이 수정은 배포용 트리에만 하고 커밋하지 않습니다. 이미 배포 중에 database 스택이 실패했다면 [10장](10-troubleshooting.md#aurora-버전-없음)대로 실패한 스택을 지운 뒤 다시 배포합니다.
</details>

다음: [3. Entra 용 설정값 반영](03-configure-source.md)
