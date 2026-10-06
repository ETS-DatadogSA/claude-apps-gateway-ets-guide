# 9. 업데이트와 삭제

![업데이트와 삭제 시 바뀌는 것과 남는 것](images/09-update-and-cleanup.drawio.png)

## 9.1 업데이트

변경을 받아 와 재배포합니다. 로컬의 GUID 수정([3.1](03-configure-source.md#31-관리-콘솔의-어드민-그룹-지정))은 `--autostash` 가 잠시 치웠다가 되살립니다.

```bash
git pull --autostash && git status --short
```

```bash
cd cdk && npm install && npx cdk deploy --all -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

- 같은 줄이 바뀌어 충돌하면 직접 맞추고, [3.2](03-configure-source.md#32-확인)로 값을 다시 확인합니다.
- `gateway/`·`admin-console/` 가 바뀌었으면 이미지가 다시 빌드됩니다.

## 9.2 스택 삭제

`destroy` 에도 컨텍스트 네 개가 필요합니다(새 셸이면 [4.5](04-deploy.md#45-새-셸에서-변수-복원) 먼저).

```bash
npx cdk destroy --all -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

> [!WARNING]
> CloudWatch 로그 그룹도 함께 지워집니다. 나중에 볼 로그는 먼저 내보냅니다.

남은 스택 확인(빈 결과면 완료):

```bash
aws cloudformation list-stacks --region us-east-1 --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE DELETE_FAILED --query "StackSummaries[?starts_with(StackName,'ClaudeGateway')].{Name:StackName,Status:StackStatus}"
```

`DELETE_FAILED` 면 CloudFormation 콘솔의 스택 이벤트에서 원인 리소스를 찾아 정리하고 다시 `destroy` 합니다. 흔한 원인은 아직 참조 중인 보안 그룹, 비어 있지 않은 S3 버킷입니다.

`destroy` 가 지우지 않는 것:

| 항목 | 이유 | 조치 |
| --- | --- | --- |
| bootstrap 자산 버킷(`cdk-hnb659fds-assets-<account>-us-east-1`)의 소스 zip | 다른 CDK 앱과 공유 | 필요하면 직접 삭제 |
| `CDKToolkit` 스택 | 다른 CDK 앱과 공유 | 그대로 둠 |
| Entra 앱과 어드민 그룹 | AWS 밖 | 9.3 |

<details>
<summary>지워지는 대상과 순서</summary>

의존 관계의 역순으로 지워집니다. ECS Express Mode 서비스 2개(와 Express Mode 가 만든 로드밸런서·대상 그룹·보안 그룹), Aurora 클러스터, Secrets Manager 시크릿, VPC(NAT Gateway, 서브넷, 엔드포인트), Client VPN, CloudWatch 로그 그룹(`RemovalPolicy.DESTROY`)입니다.

ECR 리포 2개도 `emptyOnDelete` 로 이미지째 지워집니다. 원본 [05-cleanup.md](original/05-cleanup.md)의 ECR 안내와 destroy 예시(`oidcClientSecret` 누락)는 현재 소스와 맞지 않습니다.
</details>

## 9.3 Entra 정리

```bash
az ad app delete --id "$APP" && az ad group delete --group "$GRP"
```

> [!CAUTION]
> 로컬의 `cdk/cdk.context.json`(secret)과 `claude-gateway-vpn-client.ovpn`(VPN 키)은 더 쓰지 않으면 지웁니다.

다음: [10. 문제 해결](10-troubleshooting.md)
