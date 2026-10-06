# Claude Apps Gateway on AWS — Entra ID 배포 가이드

[aws-samples/sample-claude-apps-gateway-on-aws](https://github.com/aws-samples/sample-claude-apps-gateway-on-aws) 를
**Microsoft Entra ID** 를 IdP 로, **us-east-1** 리전에 배포하는 절차입니다. 결과물은 Amazon Bedrock 앞단의
Claude apps gateway, 관리 콘솔, 접속용 AWS Client VPN 입니다.

업스트림 리포의 기본 문서([docs/upstream/](docs/upstream/README.md))는 Okta 기준입니다. 이 가이드는 Entra 로 갈 때 달라지는 지점과
배포가 실제로 어디서 일어나는지를 중심으로, Entra 앱을 만드는 단계부터 끝까지 다룹니다.

## 문서 순서

| 순서 | 문서 | 내용 |
| --- | --- | --- |
| 1 | [Entra ID 앱 등록](docs/01-entra-id.md) | 앱 등록, groups 클레임, client secret, 어드민 그룹, issuer |
| 2 | [AWS 배포 준비](docs/02-aws-preparation.md) | 리포 clone, 도구, 리전·자격증명, Bedrock·Aurora 확인 |
| 3 | [Entra 용 설정값 반영](docs/03-configure-source.md) | Okta 기준 설정 세 곳과 Entra 값, 어드민 그룹 GUID 채우기 |
| 4 | [CDK 배포](docs/04-deploy.md) | `cdk bootstrap`, `cdk deploy --all`, 출력값 기록 |
| 5 | [VPN 연결과 리다이렉트 URI](docs/05-vpn-and-redirect.md) | VPN 프로필 연결, Entra 리다이렉트 URI 를 실제 값으로 교체 |
| 6 | [배포 확인](docs/06-verify.md) | 헬스체크, 콘솔 사인인, 비용 한도, 모델, 실제 추론 |
| 7 | [업데이트와 삭제](docs/07-update-and-cleanup.md) | 재배포, `cdk destroy`, Entra 정리 |
| 8 | [문제 해결](docs/08-troubleshooting.md) | 증상별 원인과 조치 |

모든 단계는 같은 셸에서 이어서 진행하는 것을 전제로 합니다. 앞 단계의 셸 변수(`APP`, `SECRET`, `GRP`,
`ISSUER`)를 뒤에서 씁니다. 셸을 새로 열었다면 [4. CDK 배포](docs/04-deploy.md#45-새-셸에서-변수-복원)의 복원 방법을 따릅니다.

## 어디서 배포하나

명령은 운영자 PC 의 이 리포 clone 에서 실행하고, 실제 리소스는 배포 계정의 us-east-1 에 CloudFormation 스택
7개로 올라갑니다. 컨테이너 이미지는 PC 가 아니라 AWS 안의 임시 EC2 가 빌드합니다.

![Claude Apps Gateway on AWS — Entra ID 배포 구성](docs/architecture.drawio.png)

다이어그램 원본은 [docs/architecture.drawio](docs/architecture.drawio) 이고, PNG 에도 draw.io XML 이 들어 있어 draw.io 에서
바로 열어 고칠 수 있습니다. 구성 요소와 흐름 설명은 [docs/architecture.md](docs/architecture.md) 에 있습니다. 관리자도 사인인
과정에서 게이트웨이를 거치므로 VPN 이 필요합니다.

| 항목 | 값 |
| --- | --- |
| 소스 | 이 리포. 업스트림 [aws-samples/sample-claude-apps-gateway-on-aws](https://github.com/aws-samples/sample-claude-apps-gateway-on-aws) `main`(3bb468f)에 Entra 설정 두 곳 반영 |
| 명령 실행 위치 | 운영자 PC. 3단계는 리포 루트, 4단계부터 `cdk/` |
| AWS 계정 | IAM 롤을 만들 수 있는 주체로 인증한 배포 계정 |
| 리전 | `us-east-1` (게이트웨이 설정과 IAM 정책이 `us.anthropic.*` 추론 프로파일을 전제) |
| IdP | `az login` 한 Entra ID 테넌트 |
| 이미지 빌드 | 임시 x86_64 EC2(BuildMachine 스택)가 gateway·admin-console 이미지를 만들어 ECR 에 푸시 후 종료 |
| Lambda 번들링 | 운영자 PC (로컬 esbuild, 없으면 Docker) |
| 접속 경로 | 게이트웨이는 프라이빗 서브넷이라 Client VPN 으로만 도달. 관리 콘솔은 퍼블릭이지만 사인인이 게이트웨이를 거치므로 VPN 필요 |

`gateway/` 와 `admin-console/` 는 CDK S3 asset 으로 PC 의 로컬 트리에서 패키징됩니다. PC 에 남아 있는
수정은 그대로 배포되므로, 배포용 트리는 다른 작업 트리와 분리해 새로 clone 합니다.

## 리포 구성

| 경로 | 내용 |
| --- | --- |
| `README.md`, `docs/01`~`08` | 이 가이드 |
| `admin-console/`, `cdk/`, `gateway/` | 배포 소스. 업스트림 3bb468f 기준이며 Entra 용으로 바꾼 곳은 [3단계](docs/03-configure-source.md) 표에 정리 |
| `docs/upstream/` | 업스트림 README·문서·이미지·LICENSE(MIT-0) 원본 (Okta 기준) |

## 배포되는 스택

| 스택 | 내용 |
| --- | --- |
| `ClaudeGatewayNetworkStack` | 전용 VPC, 프라이빗·퍼블릭 서브넷, NAT Gateway, Bedrock·Secrets Manager 인터페이스 엔드포인트 |
| `ClaudeGatewayDatabaseStack` | Aurora Serverless v2 PostgreSQL |
| `ClaudeGatewaySecretsStack` | JWT 서명 키, 관리 API 키, 콘솔 세션 키, OIDC client secret |
| `ClaudeGatewayBuildMachineStack` | 이미지 빌드용 임시 EC2 |
| `ClaudeGatewayStack` | 게이트웨이 (ECS Express Mode, 프라이빗) |
| `ClaudeGatewayAdminConsoleStack` | 관리 콘솔 (ECS Express Mode, 퍼블릭) |
| `ClaudeGatewayVpnStack` | AWS Client VPN 엔드포인트와 `.ovpn` 프로필 |

## 준비물

- Node 20 이상, AWS CLI v2, Azure CLI(`az`), `jq`, AWS VPN Client
- Docker 데몬 또는 로컬 `esbuild` (Lambda 번들링용, [2단계](docs/02-aws-preparation.md) 참고)
- IAM 롤 생성 권한이 있는 AWS 자격증명
- 앱 등록과 그룹 생성이 가능한 Entra ID 계정

## 비용

NAT Gateway, Aurora Serverless v2, ALB 2개(ECS Express Mode 가 서비스마다 하나씩 생성), Client VPN 이
켜 둔 시간만큼 과금되고, Bedrock 추론은 사용량만큼 과금됩니다. 프리티어 대상이 아닙니다. 평가가 끝나면
[7. 업데이트와 삭제](docs/07-update-and-cleanup.md)를 따라 지웁니다.

## 범위 밖

업스트림 리포는 추론 게이트웨이까지만 다룹니다. Claude Desktop 사용자별 설정 전달은 별도 샘플인
[claude-apps-gateway-bootstrap](https://github.com/aws-samples/anthropic-on-aws/tree/main/claude-apps-gateway-bootstrap)
에 있으나, 그 애드온은 이 리포의 스택 출력과 바로 결합되지 않습니다.

## License

[MIT](LICENSE)
