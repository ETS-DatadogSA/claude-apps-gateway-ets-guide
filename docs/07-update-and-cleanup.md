# 7. 업데이트와 삭제

![업데이트와 삭제 시 바뀌는 것과 남는 것](images/07-update-and-cleanup.drawio.png)

## 7.1 업데이트

이 리포의 변경을 받아 와 재배포합니다. [3.1](03-configure-source.md#31-관리-콘솔의-어드민-그룹-지정)에서 넣은 GUID 는
커밋하지 않은 로컬 수정이므로 `--autostash` 로 잠시 치워 두고 받은 뒤 되살립니다. 같은 줄이 바뀌었다면 충돌이 나므로
직접 맞춥니다. 끝나면 [3.2](03-configure-source.md#32-확인)로 세 값을 다시 확인합니다.

```bash
git pull --autostash && git status --short
```

```bash
cd cdk && npm install && npx cdk deploy --all -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

`gateway/` 나 `admin-console/` 가 바뀌었으면 BuildMachine 스택이 이미지를 다시 빌드합니다.

## 7.2 스택 삭제

`destroy` 도 스택을 다시 합성하므로 컨텍스트가 필요합니다. 업스트림 [05-cleanup.md](upstream/05-cleanup.md) 의 예시에는
`oidcClientSecret` 이 빠져 있어 그대로 실행하면 실패합니다. 새 셸이라면
[4.5](04-deploy.md#45-새-셸에서-변수-복원)로 변수를 먼저 복원합니다.

```bash
npx cdk destroy --all -c oidcIssuer="$ISSUER" -c oidcClientId="$APP" -c oidcClientSecret="$SECRET" -c adminOktaGroupName="$GRP"
```

남은 스택이 없는지 확인합니다. 빈 결과면 7개 모두 지워진 것입니다.

```bash
aws cloudformation list-stacks --region us-east-1 --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE DELETE_FAILED --query "StackSummaries[?starts_with(StackName,'ClaudeGateway')].{Name:StackName,Status:StackStatus}"
```

BuildMachine 스택이 만든 ECR 리포 2개는 `emptyOnDelete` 로 이미지째 함께 지워집니다. 업스트림 정리 문서의 ECR 안내는
이 리포의 현재 소스와 맞지 않습니다. `cdk destroy` 가 지우지 않는 것은 다음과 같습니다.

| 항목 | 이유 | 조치 |
| --- | --- | --- |
| CDK bootstrap 자산 버킷(`cdk-hnb659fds-assets-<account>-us-east-1`)에 올라간 `gateway/`·`admin-console/` 소스 zip | 같은 계정·리전의 다른 CDK 앱과 공유 | 필요하면 해당 객체를 직접 삭제 |
| `CDKToolkit` 스택 | bootstrap 결과물로 다른 CDK 앱도 사용 | 그대로 둠 |
| Entra 앱과 어드민 그룹 | AWS 밖의 리소스 | 7.3 |

## 7.3 Entra 정리

```bash
az ad app delete --id "$APP" && az ad group delete --group "$GRP"
```

로컬의 `cdk/cdk.context.json` 과 `claude-gateway-vpn-client.ovpn` 에는 secret 과 VPN 키가 남아 있습니다. 더
쓰지 않으면 지웁니다.

다음: [8. 문제 해결](08-troubleshooting.md)
