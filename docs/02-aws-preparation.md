# 2. AWS 배포 준비

이 리포를 새로 받고, 배포 계정·리전을 고정하고, 배포를 막는 두 가지(Bedrock 추론 프로파일, Aurora 버전)를 미리
확인합니다. [1단계](01-entra-id.md)와 같은 셸에서 진행합니다.

## 2.1 리포 clone

이 리포의 `admin-console/`, `cdk/`, `gateway/` 는 업스트림 `main`(3bb468f)에 Entra 용 설정 두 곳을 반영한
소스입니다([3단계](03-configure-source.md)). 배포용 트리는 새로 받습니다. `gateway/` 와 `admin-console/` 는 로컬
트리에서 그대로 패키징되므로, 다른 실험용 수정이 섞인 트리로 배포하면 그 수정까지 올라갑니다.

```bash
git clone https://github.com/ETS-DatadogSA/claude-apps-gateway-ets-guide.git && cd claude-apps-gateway-ets-guide
```

## 2.2 의존성과 번들러

두 컨테이너 이미지는 AWS 안의 임시 EC2 가 빌드하므로 그쪽에는 로컬 Docker 가 필요 없습니다. 다만 CDK 가
Lambda 4개를 `NodejsFunction` 으로 번들링하는데, 이는 로컬 `esbuild` 가 있으면 그것을 쓰고 없으면 Docker 로
폴백합니다. `cdk/package.json` 에는 esbuild 가 없으므로 둘 중 하나를 준비합니다.

Docker 를 쓰지 않는다면 esbuild 를 넣습니다. 버전은 CDK 의 Docker 폴백과 맞춘 값입니다.

```bash
cd cdk && npm install && npm install --save-dev esbuild@^0.21 && cd ..
```

## 2.3 리전과 자격증명 고정

기본 리전이 다른 환경(예: `ap-northeast-2`)이면 반드시 덮어씁니다.

```bash
export AWS_REGION=us-east-1 AWS_DEFAULT_REGION=us-east-1 CDK_DEFAULT_REGION=us-east-1
```

배포 주체는 IAM 롤을 만들 수 있어야 합니다. 스택이 롤 10개를, `cdk bootstrap` 이 롤 5개와 SSM 파라미터를
만듭니다.

```bash
aws sts get-caller-identity
```

### MFA assume 프로필을 쓰는 경우

CDK(Node SDK)는 `aws` CLI 의 세션 캐시(`~/.aws/cli/cache`)를 공유하지 않습니다. CLI 로 인증했더라도 CDK 는 따로
assume 을 시도하고, 실패하면 `Unable to resolve AWS account to use` 로 멈춥니다. CLI 세션을 환경변수로 넘깁니다.

```bash
aws sts get-caller-identity --profile <admin-profile>
```

```bash
unset AWS_PROFILE && eval "$(aws configure export-credentials --profile <admin-profile> --format env)" && export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text) && aws sts get-caller-identity
```

마지막 출력이 `assumed-role/...` 이어야 합니다. IAM 사용자 ARN 이 나오면 환경변수가 비어 프로필로 폴백한
것이므로 `eval` 줄을 다시 실행합니다.

- TOTP 코드는 공백 없이 여섯 자리로 넣습니다.
- 세션은 role 의 `MaxSessionDuration`(기본 1시간) 뒤 만료되고 자동 갱신되지 않습니다. 배포가 25~35분이므로
  **배포 직전에** 인증합니다.

## 2.4 Bedrock 추론 프로파일 확인

게이트웨이 설정(`auto_include_builtin_models: true`)은 요청을 `us.anthropic.*` 크로스리전 추론 프로파일로
보냅니다. 목록이 비어 있으면 Bedrock 콘솔의 Model access 에서 Anthropic 모델 액세스를 먼저 받습니다.

```bash
aws bedrock list-inference-profiles --query "inferenceProfileSummaries[?starts_with(inferenceProfileId,'us.anthropic')].inferenceProfileId" --output table
```

> us-east-1 이 아닌 리전에 배포하면 이 프로파일이 없어 **배포는 성공하고 추론에서 실패**합니다. 그 경우
> `gateway/gateway.yaml` 의 `models:` 블록([upstream/06-custom-inference-profile.md](upstream/06-custom-inference-profile.md))과
> `cdk/lib/gateway-stack.ts` 의 IAM 정책(`inference-profile/us.anthropic.*` 만 허용)을 함께 고쳐야 하며,
> 이 가이드의 범위를 벗어납니다.

## 2.5 Aurora 버전 확인

`cdk/lib/database-stack.ts` 는 업스트림 그대로 Aurora PostgreSQL 을 `VER_16_6` 으로 고정합니다. AWS 가 그 마이너
버전을 리전에서 내리면 database 스택이 `Cannot find version 16.6 for aurora-postgresql` 로 실패합니다.
2026-09-18 us-east-1 에서 실제로 발생했습니다(일반 `16.6` 이 사라지고 `16.6-limitless` 만 남음).

```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --region us-east-1 --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion" --output json
```

결과에 `16.6` 이 있으면 그대로 진행합니다. 없으면 [8. 문제 해결](08-troubleshooting.md#aurora-버전-없음)대로
리전과 CDK 양쪽에 있는 버전으로 올린 뒤 진행합니다.

다음: [3. Entra 용 설정값 반영](03-configure-source.md)
