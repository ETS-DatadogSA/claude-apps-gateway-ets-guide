# 10. 문제 해결

| 증상 | 원인 | 조치 |
| --- | --- | --- |
| 사인인 페이지가 오류 없이 멈춤 | VPN 미연결, 또는 full-tunnel | [5.1](05-vpn-and-redirect.md#51-vpn-연결) |
| 로그인 끝에 `AADSTS50011` | 리다이렉트 URI 가 임시값 | [5.2](05-vpn-and-redirect.md#52-entra-리다이렉트-uri-교체) |
| 사인인은 되나 `/not-authorized` | groups 가 콘솔까지 오지 않음 | [아래](#사인인은-되는데-not-authorized) |
| `Missing required context values` | `-c` 누락 (bootstrap·destroy 포함) | 네 개 모두 전달, 또는 [`cdk.context.json`](04-deploy.md#43-컨텍스트-파일로-저장-선택) |
| `Unable to resolve AWS account to use` | CDK 가 CLI 의 MFA 세션을 못 읽음 | [2.3](02-aws-preparation.md#mfa-assume-프로필을-쓰는-경우) |
| `CDKToolkit` 이 `ROLLBACK_COMPLETE` | `iam:CreateRole` 권한 없음 | 권한 있는 주체로 [스택 삭제](#rollback_complete-스택-정리) 후 재시도 |
| `Cannot find version 16.6 for aurora-postgresql` | 리전에서 버전이 내려감 | [아래](#aurora-버전-없음) |
| 게이트웨이 크래시루프 후 스택 롤백 | `gateway.yaml` 에 스키마에 없는 키 | [3단계](03-configure-source.md) 외의 변경을 되돌림 |
| 게이트웨이 부팅 실패, 설정 오류 | `gateway.yaml` 들여쓰기·키 오류 | 로그에서 부팅 순서(설정 로드 → DB 마이그레이션 → `claude gateway listening`) 확인 |
| VPN 스택 `Request content has changed ... client token` | 이전 시도의 멱등성 토큰 충돌 | VPN 스택 삭제 후 재배포 |
| 스택이 `ROLLBACK_COMPLETE` 로 남음 | CREATE 실패 스택은 업데이트 불가 | [아래](#rollback_complete-스택-정리) |
| 배포는 됐는데 추론 실패 | 리전에 `us.anthropic.*` 프로파일이 없음 | us-east-1 배포 또는 [8장](08-custom-inference-profile.md) ([2.4](02-aws-preparation.md#24-bedrock-추론-프로파일-확인)) |
| `/v1/messages` 에서 `400` | 모델이 허용 목록에 없음 | 콘솔 Models 에서 켬 ([7.2](07-admin-console.md#72-모델-접근)) |

## 사인인은 되는데 not-authorized

1. [3.2](03-configure-source.md#32-확인)로 `userinfo_fallback: false` 와 `ADMIN_GROUP_NAME` 을 확인합니다.
2. `ADMIN_GROUP_NAME` 의 GUID 가 `$GRP` 와 같은지 봅니다.
3. 로그인한 계정이 그룹에 있는지 봅니다.
   ```bash
   az ad group member check --group "$GRP" --member-id "$(az ad signed-in-user show --query id -o tsv)"
   ```
4. JWT payload 를 디코드해 `groups` 에 `$GRP` 가 있는지 봅니다.

고친 뒤에는 `cdk deploy --all` 로 다시 배포해야 반영됩니다.

<details>
<summary>userinfo_fallback 이 문제가 되는 이유</summary>

fallback 이 켜져 있으면 id_token 에 실려 온 groups 가 userinfo 응답에 덮여 사라질 수 있습니다. Entra 의 userinfo 는 groups 를 돌려주지 않습니다.
</details>

## Aurora 버전 없음

`cdk/lib/database-stack.ts` 의 `AuroraPostgresEngineVersion.VER_16_6` 을 리전과 CDK 양쪽에 있는 버전으로 바꿉니다.

리전에서 쓸 수 있는 버전:

```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --region us-east-1 --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion" --output json
```

CDK 가 아는 버전(`cdk/` 에서):

```bash
grep -oE "VER_16_[0-9]+" node_modules/aws-cdk-lib/aws-rds/lib/cluster-engine.d.ts | sort -u
```

2026-09-18 us-east-1 기준으로는 `VER_16_13` 이 양쪽에 있었습니다. 실패한 database 스택은 아래처럼 지운 뒤 재배포합니다.

## ROLLBACK_COMPLETE 스택 정리

CREATE 에 실패한 스택은 지우고 다시 배포합니다.

```bash
aws cloudformation delete-stack --stack-name <스택명> --region us-east-1 && aws cloudformation wait stack-delete-complete --stack-name <스택명> --region us-east-1
```

삭제 대기 중인 시크릿이 있으면 같은 이름으로 다시 만들 수 없습니다. 재시도 전에 확인합니다(빈 결과면 진행).

```bash
aws secretsmanager list-secrets --include-planned-deletion --region us-east-1 --query "SecretList[?DeletedDate!=null].Name" --output json
```
