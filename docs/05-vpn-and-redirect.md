# 5. VPN 연결과 리다이렉트 URI

게이트웨이는 프라이빗 서브넷에 있어 VPN 없이는 닿지 않고, Entra 앱의 리다이렉트 URI 는 아직 임시값입니다.
사인인 전에 두 가지를 마칩니다. `cdk/` 에서 실행합니다.

![VPN 경로와 Entra 리다이렉트](images/05-vpn-and-redirect.drawio.png)

## 5.1 VPN 연결

VPN 프로필은 배포가 만들어 Secrets Manager 에 넣어 둡니다. 내려받습니다.

```bash
aws secretsmanager get-secret-value --secret-id "$(aws cloudformation describe-stacks --stack-name ClaudeGatewayVpnStack --query "Stacks[0].Outputs[?OutputKey=='VpnClientProfileSecretArn'].OutputValue" --output text)" --query SecretString --output text | jq -r .ovpnProfile > claude-gateway-vpn-client.ovpn
```

1. [OpenVPN Connect](https://openvpn.net/client/) 를 설치합니다. 프로필은 인증서와 키가 파일 안에 들어 있는 표준 OpenVPN
   형식이라 따로 챙길 인증서 파일이 없습니다.
2. OpenVPN Connect 에서 프로필 가져오기(Import Profile)로 `claude-gateway-vpn-client.ovpn` 파일을 불러옵니다.
3. 가져온 프로필을 켜서 연결합니다.

> [!NOTE]
> 같은 `.ovpn` 파일을 AWS 가 배포하는 [AWS VPN Client](https://docs.aws.amazon.com/vpn/latest/clientvpn-user/user-getting-started.html)
> 나 다른 표준 OpenVPN 클라이언트로 불러와도 됩니다.

이 엔드포인트는 **split-tunnel** 입니다. VPC 로 가는 트래픽만 터널을 타고, 브라우저의 Entra 리다이렉트는 터널
밖으로 나갑니다. full-tunnel 로 바꾸면 사인인이 오류 없이 멈춥니다.

관리 콘솔은 퍼블릭이지만 사인인 과정에서 브라우저가 게이트웨이의 `/device` 페이지로 이동하므로, 관리자도 VPN 이
필요합니다. VPN 없이 사인인하면 페이지가 오류 없이 무한 대기합니다.

`.ovpn` 파일에는 클라이언트 인증서와 키가 들어 있습니다. 리포에 커밋하지 않습니다.

## 5.2 Entra 리다이렉트 URI 교체

[1.2](01-entra-id.md#12-앱-등록)에서 넣은 임시값을 게이트웨이의 실제 주소로 바꿉니다.

```bash
GWEP=$(aws cloudformation describe-stacks --stack-name ClaudeGatewayStack --query "Stacks[0].Outputs[?OutputKey=='GatewayEndpoint'].OutputValue" --output text) && az ad app update --id "$APP" --web-redirect-uris "$GWEP/oauth/callback" && echo "$GWEP/oauth/callback"
```

건너뛰면 device 플로우 끝에서 Entra 가 `AADSTS50011` 로 사인인을 거부합니다.

## 5.3 헬스체크

VPN 이 연결된 상태에서 게이트웨이가 응답하는지 확인합니다.

```bash
curl -s "$GWEP/healthz"
```

다음: [6. 배포 확인](06-verify.md)
