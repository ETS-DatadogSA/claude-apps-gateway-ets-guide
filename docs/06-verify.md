# 6. 배포 확인

다섯 항목이 모두 통과하면 인증, 비용 한도, 모델 관리, Bedrock 추론까지 확인된 것입니다. 모두 VPN 을 연결한 상태에서 진행합니다.

![다섯 가지 확인이 지나가는 경로](images/06-verify.drawio.png)

| 확인 | 방법 | 기대 결과 |
| --- | --- | --- |
| 게이트웨이 헬스 | `curl -s "$GWEP/healthz"` | `{"status":"ok"}` 류의 응답 |
| 관리 콘솔 사인인 | 브라우저로 `<AdminConsoleEndpoint>/signin` | Entra 로그인 후 `/spend/dashboard` 도착 |
| 비용 한도 | Limits 에서 테스트 한도 생성, Audit 확인 | actor 가 `oidc:<로그인한 관리자>` |
| 모델 접근 | Models 화면 | Bedrock Anthropic 모델 목록, 기본 3종 활성 |
| 실제 추론 | Claude Code 로 게이트웨이 로그인 후 프롬프트 1회 | 응답 완료, Audit 에 추론 기록 |

원본의 화면 설명은 [original/03-verify.md](original/03-verify.md) 에 있습니다.

## 6.1 게이트웨이 헬스

```bash
curl -s "$GWEP/healthz"
```

실패하면 CloudWatch 의 게이트웨이 로그 그룹에서 부팅 순서(설정 로드 → DB 마이그레이션 → `claude gateway listening`)를 확인합니다.

## 6.2 관리 콘솔 사인인

1. 브라우저로 `<AdminConsoleEndpoint>/signin` 을 엽니다.
2. device 인증 흐름을 따라 Entra 로그인 페이지로 넘어갑니다.
3. [어드민 그룹](01-entra-id.md#15-어드민-그룹-생성)에 속한 계정으로 로그인합니다.
4. `/spend/dashboard` 에 도착하면 성공입니다.

`/not-authorized` 로 가면 → [10장](10-troubleshooting.md#사인인은-되는데-not-authorized)

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
