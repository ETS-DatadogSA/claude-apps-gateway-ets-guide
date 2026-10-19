# 6. 배포 확인

다섯 항목이 모두 통과하면 인증, 비용 한도, 모델 관리, Bedrock 추론까지 확인된 것입니다. 모두 VPN 을 연결한 상태에서 진행합니다.

| 확인 | 방법 | 기대 결과 |
| --- | --- | --- |
| 게이트웨이 헬스 | `curl -s "$GWEP/healthz"` | `ok` |
| 관리 콘솔 사인인 | 브라우저로 `<AdminConsoleEndpoint>/signin` | Entra 로그인 후 `/spend/dashboard` 도착 |
| 비용 한도 | Limits 에서 테스트 한도 생성, Audit 확인 | actor 가 `oidc:<로그인한 관리자>` |
| 모델 접근 | Models 화면 | Bedrock Anthropic 모델 목록, 기본 3종 활성 |
| 실제 추론 | Claude Code 로 게이트웨이 로그인 후 프롬프트 1회 | 응답 완료, Audit 에 추론 기록 |

원본의 화면 설명은 [original/03-verify.md](original/03-verify.md) 에 있습니다.

## 6.1 게이트웨이 헬스

[5.3](05-vpn-and-redirect.md#53-헬스체크)에서 `ok` 와 `200` 을 확인했다면 통과입니다. 실패하면 CloudWatch 의 게이트웨이 로그 그룹에서 부팅 순서(설정 로드 → DB 마이그레이션 → `claude gateway listening`)를 확인합니다.

## 6.2 관리 콘솔 사인인

관리 콘솔은 게이트웨이를 거쳐 Entra 로 로그인하고, 토큰의 `groups` 에 Admin Group 이 있으면 관리자로 들어갑니다. 1·3장의 설정이 맞는지 여기서 확인됩니다. VPN 을 연결한 상태에서 진행합니다.

### 관리 콘솔 주소 확인 명령어
```bash
echo "$(aws cloudformation describe-stacks --stack-name ClaudeGatewayAdminConsoleStack --query "Stacks[0].Outputs[?OutputKey=='AdminConsoleEndpoint'].OutputValue" --output text)/signin"
```

### 관리 콘솔 주소 확인 결과 예시
```text
https://cl-yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy.ecs.us-east-1.on.aws/signin
```

### 사인인 순서
1. 브라우저로 위 주소를 열고 **Sign in through the gateway** 를 누릅니다.

   <img src="images/06-signin.png" alt="관리 콘솔 사인인" width="420">

2. 확인 코드와 **Open verification page** 가 나옵니다. 버튼을 누르면 새 탭에 게이트웨이의 확인 페이지가 열리고, 이 화면은 `Waiting for confirmation...` 으로 기다립니다.

   <img src="images/06-signin-code.png" alt="확인 코드" width="420">

3. 새 탭의 **Approve sign-in?** 에서 코드가 앞 화면과 같은지 확인하고 **This matches my device — continue** 를 누릅니다. 이 페이지가 게이트웨이의 `/device` 이므로 VPN 이 필요합니다.

   <img src="images/06-device-approve.png" alt="Approve sign-in" width="520">

4. Entra 로그인으로 넘어갑니다. Admin Group 에 속한 계정으로 로그인합니다. 처음이면 **요청된 권한** 동의 화면이 나오며 **수락**을 누릅니다.

   <img src="images/06-entra-consent.png" alt="Entra 권한 동의" width="360">

5. 첫 탭으로 돌아와 **Effective spend**(`/spend/dashboard`) 화면이 열리면 성공입니다. 왼쪽 메뉴에 Spend limits·Audit log·Available models 가 보이면 관리자로 인식된 것입니다.

   ![관리 콘솔 Effective spend](images/06-console-dashboard.png)

아직 추론 기록이 없어 `No spend recorded yet for this period.` 가 나오는 것이 정상입니다. 6.5 에서 추론을 한 번 하면 사용자가 나타납니다.

> [!TIP]
> 동의 화면의 **조직 대신 동의**는 테넌트 관리자만 체크할 수 있습니다. 체크하면 다른 사용자는 동의 화면 없이 바로 로그인합니다. 체크하지 않으면 사용자마다 처음 한 번 동의합니다.

> [!WARNING]
> `/not-authorized` 로 가면 Admin Group 판별이 실패한 것입니다. [10장](10-troubleshooting.md#사인인은-되는데-not-authorized)을 봅니다. 2단계에서 멈추면 VPN 연결을 확인합니다(5.1).

## 6.3 비용 한도와 감사 로그

1. Limits(`/spend/limits`)에서 테스트 한도(예: 조직, 월 $10)를 만듭니다.
2. Audit(`/spend/audit`)에서 actor 가 로그인한 관리자 본인(`oidc:<subject>`)인지 봅니다.
3. 테스트 한도를 지웁니다.

게이트웨이에 직접 조회할 수도 있습니다(관리자의 게이트웨이 토큰 필요).

```bash
curl -s -H "Authorization: Bearer <관리자의 게이트웨이 토큰>" "$GWEP/v1/organizations/audit_log?limit=5"
```

## 6.4 모델 접근

Models(`/models`)에 리전의 Bedrock Anthropic 모델 목록이 보이고, 기본 3종(Opus, Sonnet, Haiku)이 켜져 있는지 봅니다.

## 6.5 실제 추론

macOS 는 `/Library/Application Support/ClaudeCode/managed-settings.json` 에 다음을 씁니다(`sudo` 필요).

```json
{
  "forceLoginMethod": "gateway",
  "forceLoginGatewayUrl": "https://<GatewayEndpoint>"
}
```

> [!IMPORTANT]
> 게이트웨이 주소는 managed settings 에서만 읽습니다. 개인 `~/.claude/settings.json` 에 넣으면 적용되지 않습니다. 사용자 경로(`~/Library/Application Support/ClaudeCode/`)에도 파일이 있으면 시스템 경로가 우선합니다.

1. VPN 을 연결한 채 새 터미널에서 `claude` 를 실행합니다.
2. "Trust gateway" 프롬프트의 호스트명이 방금 배포한 게이트웨이인지 확인합니다.
3. device 인증을 승인하고 프롬프트를 하나 실행합니다.
4. 콘솔 Audit 에 추론 기록(모델, 사용자)이 남는지 봅니다.

다음: [7. 관리 콘솔 사용법](07-admin-console.md)
