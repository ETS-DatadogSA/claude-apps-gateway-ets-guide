# 2. AWS 배포 준비

리포를 받고, 리전·자격증명을 고정하고, 배포를 막는 두 가지(Bedrock 추론 프로파일, Aurora 버전)를 미리 확인합니다.

## 2.1 Repository clone

배포에 쓸 소스를 새 폴더로 받습니다. 명령을 실행한 위치 아래에 `claude-apps-gateway-ets-guide` 폴더가 생깁니다.

### Repository clone 명령어
```bash
git clone https://github.com/ETS-DatadogSA/claude-apps-gateway-ets-guide.git && cd claude-apps-gateway-ets-guide
```

### Repository clone 결과 예시
```text
Cloning into 'claude-apps-gateway-ets-guide'...
```

> [!TIP]
> 같은 이름의 폴더가 이미 있으면 `fatal: destination path 'claude-apps-gateway-ets-guide' already exists and is not an empty directory.` 로 실패합니다. \
> 이때는 `git clone <URL> <새 폴더 이름> && cd <새 폴더 이름>` 처럼 폴더 이름을 지정합니다.

> [!WARNING]
> `gateway/`·`admin-console/` 는 로컬 트리에서 그대로 패키징됩니다. 다른 수정이 섞인 트리로 배포하면 그 수정까지 올라가므로 배포용 트리는 새로 받습니다.

## 2.2 의존성과 번들러

CDK 가 Lambda 4개를 번들링할 때 로컬 `esbuild` 를 쓰고, 없으면 Docker 를 씁니다. Docker 를 쓰지 않는다면 `esbuild` 를 설치합니다.

### 의존성 설치 명령어
```bash
cd cdk && npm install && npm install --save-dev esbuild@^0.21 && npx esbuild --version && cd ..
```

### 의존성 설치 결과 예시
마지막 `npm install` 출력과 esbuild 버전입니다.

```text
added 2 packages, and audited 352 packages in 1s

35 packages are looking for funding
  run `npm fund` for details

38 vulnerabilities (4 moderate, 33 high, 1 critical)
...
npm warn install-scripts 2 packages have install scripts not yet covered by allowScripts:
npm warn install-scripts   fsevents@2.3.3 (install: (install scripts present))
npm warn install-scripts   esbuild@0.21.5 (postinstall: node install.js)
...
0.21.5
```

마지막 줄에 `0.21.x` 가 나오면 성공입니다. npm warn 관련 내용의 경우 배포해 영향이 없으므로 진행합니다.

- `vulnerabilities`: 원본 리포 의존성에 대한 `npm audit` 요약입니다. `npm audit fix` 는 실행하지 않습니다.
- `install-scripts … not yet covered by allowScripts`: npm 이 설치 스크립트를 기본으로 막아서 나오는 경고입니다. esbuild 실행 파일은 `@esbuild/<플랫폼>` 패키지로 따로 설치되므로 스크립트가 막혀도 동작합니다.

> [!TIP]
> 이 명령은 `cdk/package.json`·`package-lock.json` 에 esbuild 를 추가합니다. 배포용 트리에서만 쓰고 커밋하지 않습니다.

<details>
<summary>둘 중 한가지 작업이 필요한 이유</summary>

컨테이너 이미지는 AWS 안의 임시 EC2 가 빌드하므로 로컬 Docker 가 필요 없습니다. 하지만 Lambda 번들링(`NodejsFunction`)은 로컬에서 하고, `cdk/package.json` 에는 `esbuild` 가 없습니다. 버전 `^0.21` 은 CDK 의 Docker 폴백과 맞춘 값입니다.
</details>

## 2.3 리전과 자격증명 고정

리전을 us-east-1 로 고정하고, 배포에 쓸 자격증명이 무엇인지 확인합니다. 기본 리전이 다른 환경(예: `ap-northeast-2`)이면 반드시 덮어씁니다. \
`AWS_PAGER=""` 는 AWS CLI 가 출력을 페이저(`(END)` 화면)로 보여 주지 않게 합니다.

### 리전·자격증명 확인 명령어
```bash
export AWS_REGION=us-east-1 AWS_DEFAULT_REGION=us-east-1 CDK_DEFAULT_REGION=us-east-1 AWS_PAGER="" && aws sts get-caller-identity
```

### 리전·자격증명 확인 결과 예시
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/<user-name>"
}
```

`Account` 가 배포할 계정이면 됩니다. `Arn` 은 IAM 사용자면 `:user/…`, 롤이면 `:assumed-role/…` 로 나옵니다.

> [!WARNING]
> 배포에 쓰는 자격증명(IAM 사용자 또는 롤)에 IAM 롤 생성 권한이 있어야 합니다. CDK 가 배포하면서 IAM 롤을 직접 만들기 때문입니다(스택 10개, `cdk bootstrap` 5개 + SSM 파라미터). `IAMReadOnlyAccess` 처럼 IAM 이 읽기 전용인 자격증명이면 4단계 `cdk bootstrap` 에서 실패하므로, 권한이 있는 프로필로 바꿉니다.

### MFA assume 프로필을 쓰는 경우

CDK 는 `aws` CLI 의 세션 캐시를 공유하지 않습니다. CLI 세션을 환경변수로 넘깁니다.

```bash
aws sts get-caller-identity --profile <admin-profile>
unset AWS_PROFILE && eval "$(aws configure export-credentials --profile <admin-profile> --format env)" && export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text) && aws sts get-caller-identity
```

- 마지막 출력이 `assumed-role/...` 이어야 합니다. IAM 사용자 ARN 이 나오면 `eval` 줄을 다시 실행합니다.
- TOTP 코드는 공백 없이 여섯 자리로 넣습니다.
- 세션은 기본 1시간 뒤 만료되고 자동 갱신되지 않습니다. 배포가 25~35분이므로 **배포 직전에** 인증합니다.

## 2.4 Bedrock 추론 프로파일 확인

게이트웨이는 기본으로 `us.anthropic.*` 프로파일을 씁니다. 목록이 비어 있으면 Bedrock 콘솔의 Model access 에서 Anthropic 모델 액세스를 먼저 받습니다.

### 추론 프로파일 확인 명령어
```bash
aws bedrock list-inference-profiles --query "inferenceProfileSummaries[?starts_with(inferenceProfileId,'us.anthropic')].inferenceProfileId" --output table
```

### 추론 프로파일 확인 결과 예시
2026-10-19 us-east-1 기준입니다. 목록은 계정의 모델 액세스에 따라 다릅니다.

```text
--------------------------------------------------
|              ListInferenceProfiles             |
+------------------------------------------------+
|  us.anthropic.claude-3-sonnet-20240229-v1:0    |
|  us.anthropic.claude-3-haiku-20240307-v1:0     |
|  us.anthropic.claude-opus-4-1-20250805-v1:0    |
...
|  us.anthropic.claude-fable-5-1                 |
|  us.anthropic.claude-opus-5-5                  |
|  us.anthropic.claude-sonnet-5-5                |
|  us.anthropic.claude-haiku-5-5                 |
...
+------------------------------------------------+
```

`us.anthropic.` 으로 시작하는 항목이 하나 이상 나오면 됩니다.

> [!WARNING]
> `us.anthropic.*` 프로파일이 없는 리전(예: `ap-northeast-2`)에서는 **배포는 성공하고 추론이 실패**합니다. 이 경우 application inference profile 을 만들어 쓰면 IAM 수정 없이 해결됩니다([8장](08-custom-inference-profile.md)).

<details>
<summary>IAM 이 허용하는 프로파일 범위</summary>

게이트웨이 IAM 정책은 배포 리전의 `inference-profile/us.anthropic.*` 시스템 프로파일과, 같은 계정의 application inference profile(리전 무관)만 허용합니다. `global.anthropic.*` 같은 다른 시스템 프로파일을 쓰려면 `cdk/lib/gateway-stack.ts` 의 IAM 정책도 고쳐야 하며, 이 가이드의 범위를 벗어납니다.
</details>

## 2.5 Aurora 버전 확인

`cdk/lib/database-stack.ts` 는 Aurora PostgreSQL `VER_16_6` 으로 고정돼 있습니다. 리전에 `16.6` 이 없으면 database 스택이 실패합니다.

### Aurora 버전 확인 명령어
```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --region us-east-1 --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion" --output json
```

### Aurora 버전 확인 결과 예시
2026-10-19 us-east-1 기준입니다.

```json
[
    "16.4-limitless",
    "16.6-limitless",
    "16.8",
    "16.8-limitless",
    "16.9",
    "16.9-limitless",
    "16.10",
    "16.10-limitless",
    "16.11",
    "16.11-limitless",
    "16.13",
    "16.13-limitless",
    "16.14",
    "16.15"
]
```

> [!WARNING]
> 결과에 `"16.6"` 이 없으면(`16.6-limitless` 만 있으면) database 스택이 실패합니다. 아래 순서로 버전을 바꿉니다. \
> 2026-09-18 과 2026-10-19 us-east-1 에서 모두 `16.6` 이 없었습니다.

`"16.6"` 이 있으면 이 절의 나머지는 건너뜁니다.

### CDK 가 아는 버전 확인 명령어
`cdk/` 에서 실행합니다. `cdk/lib/database-stack.ts` 는 CDK 상수(`VER_16_x`)로 버전을 지정하므로, 리전에 있어도 CDK 에 상수가 없는 버전은 쓸 수 없습니다.

```bash
cd cdk && grep -oE "VER_16_[0-9]+" node_modules/aws-cdk-lib/aws-rds/lib/cluster-engine.d.ts | sort -u && cd ..
```

### CDK 가 아는 버전 확인 결과 예시
`aws-cdk-lib 2.261.0` 기준입니다.

```text
VER_16_0
VER_16_1
VER_16_10
VER_16_11
VER_16_13
VER_16_2
VER_16_3
VER_16_4
VER_16_5
VER_16_6
VER_16_8
VER_16_9
```

두 결과를 대조해, `-limitless` 가 아닌 리전 버전 가운데 CDK 상수가 있는 **가장 높은 버전**을 고릅니다. 위 예시에서는 `16.14`·`16.15` 는 CDK 상수가 없으므로 `16.13`(`VER_16_13`)입니다.

### Aurora 버전 변경 명령어
리포 루트에서 실행합니다. `VER_16_13` 은 위에서 고른 버전으로 바꿉니다.

macOS:

```bash
sed -i '' 's/AuroraPostgresEngineVersion.VER_16_6,/AuroraPostgresEngineVersion.VER_16_13,/' cdk/lib/database-stack.ts && grep -n "VER_16" cdk/lib/database-stack.ts
```

Windows Git Bash / Linux:

```bash
sed -i 's/AuroraPostgresEngineVersion.VER_16_6,/AuroraPostgresEngineVersion.VER_16_13,/' cdk/lib/database-stack.ts && grep -n "VER_16" cdk/lib/database-stack.ts
```

### Aurora 버전 변경 결과 예시
```text
49:        version: rds.AuroraPostgresEngineVersion.VER_16_13,
```

> [!TIP]
> 이 수정은 배포용 트리에만 하고 커밋하지 않습니다. 이미 배포 중에 database 스택이 실패했다면 [10장](10-troubleshooting.md#aurora-버전-없음)대로 실패한 스택을 지운 뒤 다시 배포합니다.

다음: [3. Entra 용 설정값 반영](03-configure-source.md)
