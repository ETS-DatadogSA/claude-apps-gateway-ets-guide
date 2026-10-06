# 아키텍처

![Claude Apps Gateway on AWS — Entra ID 배포 구성](architecture.drawio.png)

원본: [architecture.drawio](architecture.drawio). 업스트림 3bb468f 의 CDK 소스(`cdk/lib/*.ts`) 기준으로 그렸습니다.

## 요청 흐름

1. 개발자는 Claude Code CLI 로 Client VPN(split-tunnel, 클라이언트 CIDR `10.100.0.0/16`)에 붙어 내부 ALB 를 거쳐 게이트웨이에 도달합니다.
2. 게이트웨이는 사용자 인증을 Microsoft Entra ID 에 OIDC 로 위임합니다. Entra 로 나가는 트래픽은 NAT Gateway 를 지납니다.
3. 게이트웨이는 VPC 엔드포인트(`bedrock-runtime`)를 통해 Amazon Bedrock 의 `us.anthropic.*` 추론 프로파일을 호출합니다.
4. 사용량·비용 한도 같은 상태는 Aurora Serverless v2 에 저장하고, 비밀값은 VPC 엔드포인트(`secretsmanager`)로 Secrets Manager 에서 읽습니다.
5. 관리자는 인터넷용 ALB 를 거쳐 관리 콘솔에 접속합니다. 콘솔은 게이트웨이의 관리 API 를 호출합니다. 사인인 중 브라우저가 게이트웨이의 `/device` 페이지로 이동하므로 관리자도 VPN 이 필요합니다.

## 배포 흐름

1. 운영자 PC 에서 `npx cdk deploy --all` 을 실행하면 CloudFormation 이 스택 7개를 만듭니다.
2. 임시 BuildMachine EC2(퍼블릭 서브넷, t3.small)가 S3 에 올라간 `gateway/`·`admin-console/` 소스로 이미지를 빌드해 ECR 리포 2개에 푸시하고 종료됩니다.
3. ECS Express Mode 가 두 서비스를 만들면서 각자의 ALB 를 함께 만듭니다. 게이트웨이는 프라이빗 서브넷에 있어 내부 ALB, 관리 콘솔은 퍼블릭 서브넷에 있어 인터넷용 ALB 를 받습니다.

## 구성 요소

| 구성 요소 | 위치 | 역할 | 스택 |
| --- | --- | --- | --- |
| Claude apps gateway (ECS Express, Fargate) | Private subnet | 인증, 비용 한도, Bedrock 프록시 | `ClaudeGatewayStack` |
| ALB (internal) | Private subnet | 게이트웨이 앞단. ECS Express 가 생성 | `ClaudeGatewayStack` |
| 관리 콘솔 (ECS Express, Fargate) | Public subnet | 비용 한도·모델 접근 관리 UI | `ClaudeGatewayAdminConsoleStack` |
| ALB (internet-facing) | Public subnet | 관리 콘솔 앞단. ECS Express 가 생성 | `ClaudeGatewayAdminConsoleStack` |
| Client VPN | Private subnet 연결 | 게이트웨이 접속 경로 | `ClaudeGatewayVpnStack` |
| Aurora Serverless v2 (PostgreSQL) | Private subnet | 게이트웨이 DB | `ClaudeGatewayDatabaseStack` |
| VPC 엔드포인트 | Private subnet | `bedrock-runtime`, `secretsmanager` | `ClaudeGatewayNetworkStack` |
| NAT Gateway | Public subnet | 프라이빗 서브넷의 외부 통신(Entra, ECR 등) | `ClaudeGatewayNetworkStack` |
| BuildMachine EC2 | Public subnet | 이미지 빌드 후 종료 | `ClaudeGatewayBuildMachineStack` |
| ECR | 리전 | gateway·admin-console 이미지 | `ClaudeGatewayBuildMachineStack` |
| Secrets Manager | 리전 | OIDC client secret, JWT 키, DB URL 등 | `ClaudeGatewaySecretsStack`, `ClaudeGatewayDatabaseStack` |
| Amazon Bedrock | 리전 | Claude 모델 추론 | (외부 서비스) |
| Microsoft Entra ID | AWS 밖 | OIDC IdP | (외부 서비스) |

## 설계 포인트

- 게이트웨이는 퍼블릭 경로 없이 VPN 으로만 노출합니다. 관리 콘솔은 네트워크 위치가 아니라 Entra 그룹 소속으로 접근을 통제합니다.
- 이미지를 로컬이 아니라 x86_64 EC2 에서 빌드합니다. 게이트웨이 Dockerfile 이 x86_64 전용 바이너리를 받기 때문에 Apple Silicon 에서 빌드한 이미지는 Fargate 에서 돌지 않습니다.
- Bedrock 과 Secrets Manager 호출은 VPC 엔드포인트로 VPC 안에 머뭅니다.

## 다이어그램 수정

draw.io 에서 `architecture.drawio` 를 고친 뒤 PNG 를 다시 찍습니다(macOS).

```bash
/Applications/draw.io.app/Contents/MacOS/draw.io -x -f png -e -b 10 -o docs/architecture.drawio.png docs/architecture.drawio
```
