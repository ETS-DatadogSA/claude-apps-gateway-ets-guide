# 8. 커스텀 추론 프로파일 (선택)

기본으로 모든 모델은 `us.anthropic.*` 크로스리전 추론 프로파일로 갑니다. 특정 모델만 **커스텀 Bedrock 추론 프로파일**로 보내는 방법입니다. 원본은 [original/06-custom-inference-profile.md](original/06-custom-inference-profile.md) 입니다.

![기본 프로파일과 커스텀 프로파일로 갈리는 요청 경로](images/08-custom-inference-profile.drawio.png)

- 주로 비용 배분용 [application inference profile](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference-profiles-support.html), 프로비저닝된 처리량, 가드레일을 붙인 프로파일에 씁니다.
- 모델마다 다른 프로파일을 지정할 수 있고, 지정하지 않은 모델은 기본 프로파일을 그대로 씁니다.

> [!TIP]
> `us.anthropic.*` 프로파일이 없는 리전에 배포할 때도 이 방법을 쓰면 됩니다. 같은 계정의 application inference profile 은 리전과 관계없이 IAM 이 이미 허용합니다([8.4](#84-iam--추가할-것-없음)).

## 8.1 시작 전에 필요한 것

| 항목 | 예시 | 확인 방법 |
| --- | --- | --- |
| 프로파일 ARN | `arn:aws:bedrock:us-east-2:111111111111:application-inference-profile/abc123` | Bedrock 콘솔, `aws bedrock list-application-inference-profiles` |
| 게이트웨이 모델 ID | `claude-sonnet-5`, `claude-opus-4-8` | 관리 콘솔 Models, `AVAILABLE_MODELS_RAW` |
| 모델 액세스 | 프로파일이 있는 리전에서 해당 모델 | [Bedrock 모델 액세스](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) |

프로파일은 배포 리전과 다른 리전에 있어도 됩니다.

## 8.2 `gateway/gateway.yaml` 에 모델 추가

`auto_include_builtin_models: true` 아래의 주석 처리된 `models:` 예시를 풀고 값을 채웁니다.

```yaml
auto_include_builtin_models: true

models:
  - id: claude-sonnet-5
    label: Claude Sonnet 5 (custom inference profile)
    upstream_model:
      bedrock: arn:aws:bedrock:us-east-2:111111111111:application-inference-profile/abc123
```

모델을 더 보내려면 항목을 추가합니다.

```yaml
models:
  - id: claude-sonnet-5
    label: Claude Sonnet 5 (custom inference profile)
    upstream_model:
      bedrock: arn:aws:bedrock:us-east-2:111111111111:application-inference-profile/abc123
  - id: claude-opus-4-8
    label: Claude Opus 4.8 (custom inference profile)
    upstream_model:
      bedrock: arn:aws:bedrock:us-east-2:111111111111:application-inference-profile/def456
```

- `upstream_model` 의 키는 `bedrock` 으로 둡니다(`upstreams:` 항목에 `name:` 이 없어 provider 이름이 기본값).
- `id:` 는 게이트웨이 모델 ID 입니다. ARN 과 맞출 필요 없습니다.
- 환경변수나 CDK 컨텍스트와 무관한 순수 YAML 입니다.

> [!WARNING]
> 게이트웨이는 부팅 때 설정 전체를 검증합니다. 들여쓰기가 틀리거나 `id:` 가 중복되면 기동에 실패합니다. [3단계](03-configure-source.md)의 `oidc` 설정도 그대로인지 확인합니다.

## 8.3 허용 목록에 모델 켜기 (아직 꺼져 있다면)

`models:` 는 요청이 **어디로 갈지**만 정합니다. 모델이 허용 목록(`availableModels`)에 없으면 `400` 으로 거부됩니다.

- 첫 배포 전: `cdk/lib/gateway-stack.ts` 의 `AVAILABLE_MODELS_RAW` 에 추가
- 배포 후: 관리 콘솔 Models 화면에서 켬 (재빌드 불필요, [7.2](07-admin-console.md#72-모델-접근))

## 8.4 IAM — 추가할 것 없음

게이트웨이 task role 은 이미 같은 계정의 모든 application inference profile 을 리전과 관계없이 허용합니다(`cdk/lib/gateway-stack.ts`). 프로파일을 추가·변경해도 IAM 은 그대로입니다.

```typescript
taskRole.addToPolicy(new iam.PolicyStatement({
  sid: 'BedrockInvokeCustomInferenceProfiles',
  actions: ['bedrock:InvokeModel', 'bedrock:InvokeModelWithResponseStream'],
  resources: [
    `arn:${cdk.Aws.PARTITION}:bedrock:*:${cdk.Aws.ACCOUNT_ID}:application-inference-profile/*`,
    `arn:${cdk.Aws.PARTITION}:bedrock:*::foundation-model/anthropic.*`,
  ],
}));
```

> [!NOTE]
> 프로파일이 **다른 AWS 계정**에 있으면 허용 범위 밖입니다. 그 프로파일 ARN 을 명시한 구문과 상대 계정의 교차 계정 허용이 필요하며, 이 가이드의 범위를 벗어납니다.

## 8.5 재빌드와 재배포

`gateway.yaml` 은 이미지에 들어가므로 `models:` 를 바꾸면 재빌드가 필요합니다. `cdk/` 에서 실행합니다(새 셸이면 [4.5](04-deploy.md#45-새-셸에서-변수-복원) 먼저).

```bash
npx cdk deploy ClaudeGatewayBuildMachineStack ClaudeGatewayStack -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

- `--all` 로 전체를 재배포해도 됩니다.
- `BedrockInvokeCustomInferenceProfiles` 구문을 처음 배포할 때 IAM 변경 승인을 묻습니다.
- BuildMachine 이 이미지를 다시 빌드하고 서비스가 새 이미지로 교체됩니다.

## 8.6 동작 확인

필요한 만큼 골라 확인합니다. 원본에서 실제로 쓴 방법입니다.

| 확인 | 어디서 | 기대 결과 |
| --- | --- | --- |
| 1. 부팅 로그 | 게이트웨이 CloudWatch 로그 그룹 | ARN 을 언급하는 `spend meter has no exact rates ...` 경고 |
| 2. 추론 로그 | 같은 로그 그룹 | `{"evt":"inference","model":"claude-sonnet-5",...,"status":200}` |
| 3. Bedrock 호출 로그 | [모델 호출 로깅](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) (켠 경우) | `modelId` 가 커스텀 프로파일 ARN |
| 4. 사용액 | 관리 콘솔 `/spend/dashboard` | 해당 모델 사용량이 정상 집계 |

> [!NOTE]
> 1번 경고는 정상입니다. 게이트웨이가 커스텀 ARN 을 읽었다는 뜻이고, 내부 단가표에만 영향이 있습니다. 사용량은 4번처럼 정상 집계됩니다.

<details>
<summary>로그 예시</summary>

부팅 로그:

```
warn spend meter has no exact rates for claude-sonnet-5 (arn:aws:bedrock:...:application-inference-profile/abc123) — these will be metered at the unknown-model default tier
```

Bedrock 호출 로그:

```json
{
  "operation": "InvokeModelWithResponseStream",
  "modelId": "arn:aws:bedrock:us-east-2:111111111111:application-inference-profile/abc123",
  "output": { "outputBodyJson": [{ "message": { "model": "claude-sonnet-5", ... } }] }
}
```

`models:` 에 없는 모델은 같은 로그에 기본 프로파일 ARN(예: `arn:...:inference-profile/us.anthropic.claude-sonnet-4-6`)이 찍힙니다. 나열한 모델만 바뀌었는지 비교할 때 씁니다.
</details>

## 8.7 문제 해결

| 증상 | 원인 | 조치 |
| --- | --- | --- |
| `bedrock:InvokeModel` 에서 `AccessDeniedException` | 8.5 재배포 전이거나, 프로파일이 다른 계정에 있음 | 재배포 확인, 다른 계정이면 8.4 NOTE |
| `/v1/messages` 에서 `400` | 허용 목록에 모델이 없음 | 8.3 |
| 게이트웨이 기동 실패, `models` 관련 설정 오류 | YAML 들여쓰기, `id:` 중복 | 예시와 같은 들여쓰기, `id:` 중복 확인 |

다음: [9. 업데이트와 삭제](09-update-and-cleanup.md)
