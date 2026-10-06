# 4. CDK 배포

[1단계](01-entra-id.md)에서 만든 값으로 CDK 를 bootstrap 하고 스택 7개를 배포합니다. 여기부터는 `cdk/` 에서
실행합니다.

```bash
cd cdk
```

CDK 컨텍스트 네 개를 매 명령에 넘깁니다.

| 컨텍스트 키 | 셸 변수 | 비고 |
| --- | --- | --- |
| `oidcIssuer` | `$ISSUER` | 필수 |
| `oidcClientId` | `$APP` | 필수 |
| `oidcClientSecret` | `$SECRET` | 필수 |
| `adminOktaGroupName` | `$GRP` | 키 이름은 Okta 지만 Entra 에서는 그룹 GUID 를 넣음. 빠지면 `claude-gateway-admins` 로 대체됨 |

## 4.1 bootstrap

계정·리전당 한 번입니다. 환경을 `aws://<account>/<region>` 으로 명시해도 CDK CLI 는 `bin/app.ts` 를 합성하므로
컨텍스트가 필요합니다. 빠지면 `Missing required context values` 로 멈춥니다.

```bash
npx cdk bootstrap "aws://$(aws sts get-caller-identity --query Account --output text)/us-east-1" -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

`iam:CreateRole` 권한이 없으면 `CDKToolkit` 스택이 `ROLLBACK_COMPLETE` 로 남습니다.
[8. 문제 해결](08-troubleshooting.md)을 참고합니다.

## 4.2 배포

```bash
npx cdk deploy --all -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

- 스택 7개가 25~35분에 걸쳐 올라갑니다. 의존성이 없는 스택은 병렬로 진행됩니다.
- IAM 변경 승인 프롬프트가 한 번 뜹니다. 자리를 비울 거면 `--require-approval never` 를 붙입니다.
- 도중에 자격증명이 만료돼 CDK 가 멈춰도 CloudFormation 은 AWS 쪽에서 계속 진행됩니다. 다시 인증하고 같은
  명령을 실행하면 이어집니다.

> `oidcClientSecret` 은 합성된 템플릿(`cdk.out/ClaudeGatewaySecretsStack.template.json`)에 평문으로 남습니다.
> 업스트림이 `SecretValue.unsafePlainText` 로 넘기기 때문입니다. PoC 용 패턴이며 운영 배포에는 맞지 않습니다.

## 4.3 컨텍스트 파일로 저장 (선택)

매번 `-c` 네 개를 붙이지 않으려면 `cdk/cdk.context.json` 에 저장합니다. CDK 가 이 파일을 컨텍스트로 읽으므로
이후 `bootstrap`·`deploy`·`destroy` 가 `-c` 없이 동작합니다. 업스트림 `.gitignore` 대상이지만 secret 이
평문이므로 권한을 좁힙니다.

```bash
jq -n --arg i "$ISSUER" --arg a "$APP" --arg s "$SECRET" --arg g "$GRP" '{oidcIssuer:$i, oidcClientId:$a, oidcClientSecret:$s, adminOktaGroupName:$g}' > cdk.context.json && chmod 600 cdk.context.json
```

## 4.4 출력값 기록

배포가 끝나면 두 URL 이 출력됩니다. 호스트명은 ECS Express Mode 가 만들기 때문에 배포 전에는 알 수 없습니다.

```
ClaudeGatewayStack.GatewayEndpoint                  = https://cl-xxxx.ecs.us-east-1.on.aws
ClaudeGatewayAdminConsoleStack.AdminConsoleEndpoint = https://cl-yyyy.ecs.us-east-1.on.aws
```

나중에 다시 조회할 때는 CloudFormation 출력에서 읽습니다.

```bash
GWEP=$(aws cloudformation describe-stacks --stack-name ClaudeGatewayStack --query "Stacks[0].Outputs[?OutputKey=='GatewayEndpoint'].OutputValue" --output text) && echo "GWEP=$GWEP"
```

```bash
aws cloudformation describe-stacks --stack-name ClaudeGatewayAdminConsoleStack --query "Stacks[0].Outputs[?OutputKey=='AdminConsoleEndpoint'].OutputValue" --output text
```

## 4.5 새 셸에서 변수 복원

셸을 새로 열면 변수가 사라집니다. [4.3](#43-컨텍스트-파일로-저장-선택)에서 파일을 저장했다면 `cdk/` 에서 다시
읽습니다. secret 은 Entra 에서 다시 조회할 수 없으므로, 파일이 없다면 [1.4](01-entra-id.md#14-client-secret-생성)로
새 secret 을 만들어야 합니다.

```bash
eval "$(jq -r '"export ISSUER=\(.oidcIssuer|@sh) APP=\(.oidcClientId|@sh) SECRET=\(.oidcClientSecret|@sh) GRP=\(.adminOktaGroupName|@sh)"' cdk.context.json)" && echo "APP=$APP GRP=$GRP ISSUER=$ISSUER SECRET_LEN=${#SECRET}"
```

리전·자격증명 환경변수([2.3](02-aws-preparation.md#23-리전과-자격증명-고정))도 다시 설정합니다.

다음: [5. VPN 연결과 리다이렉트 URI](05-vpn-and-redirect.md)
