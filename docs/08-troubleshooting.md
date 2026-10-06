# 8. 문제 해결

| 증상 | 원인 | 조치 |
| --- | --- | --- |
| 사인인 페이지가 오류 없이 멈춤 | VPN 미연결, 또는 full-tunnel | [5.1](05-vpn-and-redirect.md#51-vpn-연결) split-tunnel 프로필로 연결 |
| 로그인 끝에 `AADSTS50011` | 리다이렉트 URI 가 임시값 | [5.2](05-vpn-and-redirect.md#52-entra-리다이렉트-uri-교체) 실행 |
| 사인인은 되나 `/not-authorized` | groups 가 콘솔까지 오지 않음 | [아래](#사인인은-되는데-not-authorized) |
| `Missing required context values` | `-c` 누락 (bootstrap·destroy 포함) | 네 개 모두 전달하거나 `cdk.context.json` 사용 |
| `Unable to resolve AWS account to use` | CDK 가 CLI 의 MFA 세션을 못 읽음 | [2.3](02-aws-preparation.md#mfa-assume-프로필을-쓰는-경우) `export-credentials` 줄 실행 |
| `cdk bootstrap` 에서 `CDKToolkit` 이 `ROLLBACK_COMPLETE` | `iam:CreateRole` 권한 없음 | 권한 있는 주체로 스택 삭제 후 재시도 |
| `Cannot find version 16.6 for aurora-postgresql` | 리전에서 해당 마이너 버전이 내려감 | [아래](#aurora-버전-없음) |
| 게이트웨이 컨테이너 크래시루프 후 스택 롤백 | `gateway.yaml` 에 스키마에 없는 키 | [3단계](03-configure-source.md) 외의 `gateway.yaml` 변경을 되돌림 |
| VPN 스택 `Request content has changed ... client token` | 이전 시도의 멱등성 토큰 충돌 | VPN 스택 삭제 후 재배포 |
| 스택이 `ROLLBACK_COMPLETE` 로 남음 | CREATE 실패 스택은 업데이트 불가 | [아래](#rollback_complete-스택-정리) |
| 배포는 됐는데 추론 실패 | us-east-1 외 리전이라 `us.anthropic.*` 프로파일 없음 | [2.4](02-aws-preparation.md#24-bedrock-추론-프로파일-확인) |

## 사인인은 되는데 not-authorized

1. [3.4](03-configure-source.md#34-확인)의 `grep` 으로 `userinfo_fallback: false` 와 `ADMIN_GROUP_NAME` 이 실제로
   반영됐는지 확인합니다. fallback 이 켜져 있으면 id_token 의 groups 가 userinfo 응답에 덮여 사라질 수 있습니다.
2. `ADMIN_GROUP_NAME` 의 GUID 가 `$GRP` 와 같은지 확인합니다.
3. 로그인한 계정이 그룹에 있는지 확인합니다.

```bash
az ad group member check --group "$GRP" --member-id "$(az ad signed-in-user show --query id -o tsv)"
```

4. 토큰에 무엇이 들어오는지는 JWT payload 를 디코드해 확인합니다. `groups` 배열에 `$GRP` 의 GUID 가 있어야 합니다.

수정한 뒤에는 `cdk deploy --all` 로 다시 배포해야 컨테이너에 반영됩니다.

## Aurora 버전 없음

업스트림 `cdk/lib/database-stack.ts` 의 `AuroraPostgresEngineVersion.VER_16_6` 을 리전과 CDK 양쪽에 있는 버전으로
올립니다. 두 목록을 각각 조회합니다.

```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --region us-east-1 --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion" --output json
```

`cdk/` 에서 실행합니다.

```bash
grep -oE "VER_16_[0-9]+" node_modules/aws-cdk-lib/aws-rds/lib/cluster-engine.d.ts | sort -u
```

2026-09-18 us-east-1 기준으로는 `VER_16_13` 이 양쪽에 모두 있었습니다. 실패한 database 스택은
`ROLLBACK_COMPLETE` 로 남으므로 아래처럼 지운 뒤 재배포합니다.

## ROLLBACK_COMPLETE 스택 정리

CREATE 에 실패한 스택은 업데이트되지 않으므로 지운 뒤 재배포합니다. 롤백된 스택은 리소스가 없어 금방 지워집니다.

```bash
aws cloudformation delete-stack --stack-name <스택명> --region us-east-1 && aws cloudformation wait stack-delete-complete --stack-name <스택명> --region us-east-1
```

Secrets Manager 시크릿이 삭제 대기로 남아 있으면 같은 이름으로 다시 만들 수 없습니다. 재시도 전에 확인합니다.
빈 결과면 그대로 진행합니다.

```bash
aws secretsmanager list-secrets --include-planned-deletion --region us-east-1 --query "SecretList[?DeletedDate!=null].Name" --output json
```
