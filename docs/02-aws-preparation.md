# 2. AWS 배포 준비

리포를 받고, 리전·자격증명을 고정하고, 배포를 막는 두 가지(Bedrock 추론 프로파일, Aurora 버전)를 미리 확인합니다.

## 2.1 리포 clone

```bash
git clone https://github.com/ETS-DatadogSA/claude-apps-gateway-ets-guide.git && cd claude-apps-gateway-ets-guide
```

> [!WARNING]
> `gateway/`·`admin-console/` 는 로컬 트리에서 그대로 패키징됩니다. 다른 수정이 섞인 트리로 배포하면 그 수정까지 올라가므로 배포용 트리는 새로 받습니다.

## 2.2 의존성과 번들러

CDK 가 Lambda 4개를 번들링할 때 로컬 `esbuild` 를 쓰고, 없으면 Docker 를 씁니다. Docker 를 쓰지 않는다면 `esbuild` 를 설치합니다.

```bash
cd cdk && npm install && npm install --save-dev esbuild@^0.21 && cd ..
```

<details>
<summary>왜 둘 중 하나가 필요한가</summary>

컨테이너 이미지는 AWS 안의 임시 EC2 가 빌드하므로 로컬 Docker 가 필요 없습니다. 하지만 Lambda 번들링(`NodejsFunction`)은 로컬에서 하고, `cdk/package.json` 에는 `esbuild` 가 없습니다. 버전 `^0.21` 은 CDK 의 Docker 폴백과 맞춘 값입니다.
</details>

## 2.3 리전과 자격증명 고정

기본 리전이 다른 환경(예: `ap-northeast-2`)이면 반드시 덮어씁니다.

```bash
export AWS_REGION=us-east-1 AWS_DEFAULT_REGION=us-east-1 CDK_DEFAULT_REGION=us-east-1
aws sts get-caller-identity
```

배포 주체는 IAM 롤을 만들 수 있어야 합니다(스택 10개, `cdk bootstrap` 5개 + SSM 파라미터).

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

```bash
aws bedrock list-inference-profiles --query "inferenceProfileSummaries[?starts_with(inferenceProfileId,'us.anthropic')].inferenceProfileId" --output table
```

> [!WARNING]
> `us.anthropic.*` 프로파일이 없는 리전(예: `ap-northeast-2`)에서는 **배포는 성공하고 추론이 실패**합니다. 이 경우 application inference profile 을 만들어 쓰면 IAM 수정 없이 해결됩니다([8장](08-custom-inference-profile.md)).

<details>
<summary>IAM 이 허용하는 프로파일 범위</summary>

게이트웨이 IAM 정책은 배포 리전의 `inference-profile/us.anthropic.*` 시스템 프로파일과, 같은 계정의 application inference profile(리전 무관)만 허용합니다. `global.anthropic.*` 같은 다른 시스템 프로파일을 쓰려면 `cdk/lib/gateway-stack.ts` 의 IAM 정책도 고쳐야 하며, 이 가이드의 범위를 벗어납니다.
</details>

## 2.5 Aurora 버전 확인

`cdk/lib/database-stack.ts` 는 Aurora PostgreSQL `VER_16_6` 으로 고정돼 있습니다. 리전에 `16.6` 이 없으면 database 스택이 실패합니다.

```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --region us-east-1 --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion" --output json
```

결과에 `16.6` 이 없으면 [10장](10-troubleshooting.md#aurora-버전-없음)대로 버전을 올립니다. 2026-09-18 us-east-1 에서 실제로 발생했습니다(`16.6-limitless` 만 남음).

다음: [3. Entra 용 설정값 반영](03-configure-source.md)
