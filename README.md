# Claude Apps Gateway on AWS — Entra ID 배포 가이드

[aws-samples/sample-claude-apps-gateway-on-aws](https://github.com/aws-samples/sample-claude-apps-gateway-on-aws)를 **Microsoft Entra ID** 를 IdP 로, **us-east-1** 리전에 배포하는 절차입니다. 결과물은 Amazon Bedrock 앞단의 Claude apps gateway, 관리 콘솔, 접속용 AWS Client VPN 입니다.

원본 리포의 문서([docs/original/](docs/original/README.md))는 Okta 기준입니다. 이 가이드는 원본 문서의 내용을 옮기면서 Entra 로 갈 때 달라지는 지점과
배포가 실제로 어디서 일어나는지를 더해, Entra 앱을 만드는 단계부터 끝까지 다룹니다.

## 문서 순서

| 순서 | 문서 | 내용 |
| --- | --- | --- |
| 1 | [Entra ID 앱 등록](docs/01-entra-id.md) | 앱 등록, groups 클레임, client secret, 어드민 그룹, issuer |
| 2 | [AWS 배포 준비](docs/02-aws-preparation.md) | 리포 clone, 도구, 리전·자격증명, Bedrock·Aurora 확인 |
| 3 | [Entra 용 설정값 반영](docs/03-configure-source.md) | Okta 기준 설정 세 곳과 Entra 값, 어드민 그룹 GUID 채우기 |
| 4 | [CDK 배포](docs/04-deploy.md) | `cdk bootstrap`, `cdk deploy --all`, 출력값 기록 |
| 5 | [VPN 연결과 리다이렉트 URI](docs/05-vpn-and-redirect.md) | VPN 프로필 연결, Entra 리다이렉트 URI 를 실제 값으로 교체 |
| 6 | [배포 확인](docs/06-verify.md) | 헬스체크, 콘솔 사인인, 비용 한도, 모델, 실제 추론 |
| 7 | [관리 콘솔 사용법](docs/07-admin-console.md) | 사용액 대시보드, 비용 한도, 모델 접근 관리와 감사 방식 |
| 8 | [커스텀 추론 프로파일 (선택)](docs/08-custom-inference-profile.md) | 특정 모델을 application inference profile 로 보내기 |
| 9 | [업데이트와 삭제](docs/09-update-and-cleanup.md) | 재배포, `cdk destroy`, Entra 정리 |
| 10 | [문제 해결](docs/10-troubleshooting.md) | 증상별 원인과 조치 |

모든 단계는 같은 셸에서 이어서 진행하는 것을 전제로 합니다. 앞 단계의 셸 변수(`APP`, `SECRET`, `GRP`,
`ISSUER`)를 뒤에서 씁니다. 셸을 새로 열었다면 [4. CDK 배포](docs/04-deploy.md#45-새-셸에서-변수-복원)의 복원 방법을 따릅니다.

## 어디서 배포하나

명령은 운영자 PC 의 이 리포 clone 에서 실행하고, 실제 리소스는 배포 계정의 us-east-1 에 CloudFormation 스택
7개로 올라갑니다. 컨테이너 이미지는 PC 가 아니라 AWS 안의 임시 EC2 가 빌드합니다.

![Claude Apps Gateway on AWS — Entra ID 배포 구성](docs/architecture.drawio.png)

편집용 파일은 [docs/architecture.drawio](docs/architecture.drawio) 이고, PNG 에도 draw.io XML 이 들어 있어 draw.io 에서
바로 열어 고칠 수 있습니다. 구성 요소와 흐름 설명은 [docs/architecture.md](docs/architecture.md) 에 있습니다.

게이트웨이는 일부러 프라이빗으로 둡니다. 서브넷에 인터넷 경로가 없어 ECS Express Mode 가 내부 로드밸런서를 붙이고,
개발자는 VPN 을 거쳐 CLI 의 device 로그인으로만 닿습니다. 관리 콘솔은 일부러 퍼블릭으로 둡니다. 네트워크 위치가 아니라
Entra 그룹 소속으로 접근을 통제합니다. 다만 사인인 과정에서 브라우저가 게이트웨이를 거치므로 관리자도 VPN 이 필요합니다.

| 항목 | 값 |
| --- | --- |
| 소스 | 이 리포. 원본 [aws-samples/sample-claude-apps-gateway-on-aws](https://github.com/aws-samples/sample-claude-apps-gateway-on-aws) `main`(3bb468f)에 Entra 설정 두 곳 반영 |
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
| `README.md`, `docs/01`~`10` | 이 가이드 |
| `admin-console/`, `cdk/`, `gateway/` | 배포 소스. 원본 3bb468f 기준이며 Entra 용으로 바꾼 곳은 [3단계](docs/03-configure-source.md) 표에 정리 |
| `docs/original/` | 원본 리포의 README·문서·이미지·LICENSE(MIT-0), Okta 기준. 소스 주석에 나오는 `docs/0N-*.md` 경로는 이 폴더의 파일을 가리킵니다 |
| `docs/images/`, `docs/architecture.*` | 이 가이드의 다이어그램 |

## 배포되는 스택

| 스택 | 내용 |
| --- | --- |
| `ClaudeGatewayNetworkStack` | 전용 VPC: 게이트웨이용 프라이빗 서브넷 2개, 관리 콘솔용 퍼블릭 서브넷 2개, NAT Gateway 1개, Bedrock·Secrets Manager 인터페이스 엔드포인트, 이들을 잇는 보안 그룹 3개 |
| `ClaudeGatewayDatabaseStack` | Aurora Serverless v2 PostgreSQL. 저장 시 암호화, 30분 유휴 시 자동 일시정지. RDS 가 관리하는 자격증명에서 `postgres_url` 연결 시크릿 하나를 만드는 Lambda Custom Resource 포함 |
| `ClaudeGatewaySecretsStack` | JWT 서명 키, 관리 API 쓰기 키, 콘솔 세션 서명 키 생성, OIDC client secret 저장. client secret 은 배포 시 필수 파라미터(`-c oidcClientSecret`)입니다. 게이트웨이가 부팅 때 설정을 검증하므로 나중에 채우는 자리표시자로 둘 수 없습니다 |
| `ClaudeGatewayBuildMachineStack` | 두 컨테이너 이미지를 빌드해 ECR 에 푸시하고 스스로 내려가는 임시 Linux x86_64 EC2. 게이트웨이 Dockerfile 이 x86_64 전용 바이너리를 받기 때문에, Apple Silicon 에서 로컬로 빌드하면 Fargate 가 실행할 수 없는 이미지가 됩니다 |
| `ClaudeGatewayStack` | 게이트웨이. BuildMachine 이 만든 이미지, 게이트웨이용 IAM 롤, 프라이빗 서브넷의 `AWS::ECS::ExpressGatewayService` |
| `ClaudeGatewayAdminConsoleStack` | 관리 콘솔. 같은 구성을 퍼블릭 서브넷에 두고, 게이트웨이의 실제 주소를 자동으로 연결 |
| `ClaudeGatewayVpnStack` | 셀프 서비스 AWS Client VPN 엔드포인트. 배포 중에 만든 상호 TLS 인증서와 바로 가져올 수 있는 `.ovpn` 프로필을 Secrets Manager 로 전달 |

## 들어 있는 것

- **게이트웨이**: Docker 빌드 시점에 `claude` 바이너리를 내려받아 Anthropic 이 공개한 매니페스트로 검증합니다(GPG 서명 + SHA256 체크섬). 미리 준비해 둘 바이너리가 없습니다.
- **관리 콘솔**(FastAPI + 서버 렌더링 HTML): 실사용액 대시보드, 비용 한도 CRUD, 모델 접근 관리. 모델 목록은 고정 목록이 아니라 Bedrock 의 실제 Anthropic 모델 카탈로그이고, 변경은 이미지 재빌드 없이 ECS 파라미터 갱신으로 적용됩니다. 동작 방식과 감사 측면의 차이는 [7. 관리 콘솔 사용법](docs/07-admin-console.md)에 있습니다.
- **셀프 서비스 AWS Client VPN**: 배포 중에 상호 TLS 인증서 체인을 만들고 `.ovpn` 프로필을 Secrets Manager 에 넣어 둡니다. PKI 를 따로 구성하거나 인증서를 손으로 다룰 일이 없습니다.
- **CDK 앱 전체**: ECS Express Mode 서비스는 AWS CDK 의 네이티브 L1 구성인 `CfnExpressGatewayService` 로 만듭니다. 직접 짠 `AwsCustomResource` 가 필요 없습니다.

## 관리 콘솔 화면

관리 콘솔은 원래라면 지원 요청, CLI 명령, 재배포가 필요했던 비용 통제와 모델 접근 관리를 플랫폼 관리자가 직접 하게 해 줍니다.
모든 작업은 관리자 본인의 ID 로 기록되고 게이트웨이가 감사합니다. 아래 화면은 원본 리포의 스크린샷입니다.

**누가 얼마를 쓰는지 실시간으로 봅니다.**

![관리 콘솔 사용액 대시보드](docs/original/images/admin-console-spend-dashboard.png)

**한도를 몇 초 만에 정합니다. 요청이나 재배포 없이 바로 다음 요청부터 적용됩니다.**

![관리 콘솔 비용 한도](docs/original/images/admin-console-spend-limits.png)

**모든 변경이 공용 자격증명이 아니라 실제로 변경한 관리자에게 기록됩니다.**

![관리 콘솔 감사 로그](docs/original/images/admin-console-audit-log.png)

**새 Claude 모델을 체크박스 하나로 조직 전체에 엽니다. 목록은 Bedrock 카탈로그에서 실시간으로 가져옵니다.**

![관리 콘솔 모델 접근](docs/original/images/admin-console-model-access.png)

## 준비물

- Node 20 이상, AWS CLI v2, Azure CLI(`az`), `jq`, AWS VPN Client
- Docker 데몬 또는 로컬 `esbuild` (Lambda 번들링용, [2단계](docs/02-aws-preparation.md) 참고)
- IAM 롤 생성 권한이 있는 AWS 자격증명
- 앱 등록과 그룹 생성이 가능한 Entra ID 계정
- 쓰려는 Anthropic Claude 모델에 대한 [Amazon Bedrock 모델 액세스](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)

## 비용

과금되는 리소스가 만들어집니다.

| 리소스 | 과금 |
| --- | --- |
| NAT Gateway | 시간당 약 $0.045 + 데이터 처리량 (원본 README 기준) |
| Aurora Serverless v2 | 사용 중일 때 과금. 30분 유휴 후 자동 일시정지 |
| ECS Express Mode 서비스 2개 | 서비스마다 Application Load Balancer 를 하나씩 만듦 |
| Client VPN | 엔드포인트 연결과 접속 시간 |
| Bedrock | 실제 추론 사용량 |

AWS 프리티어 대상이 아닙니다. 예상 사용량에 맞춘 견적은 [AWS Pricing Calculator](https://calculator.aws/)로 내고,
평가가 끝나면 [9. 업데이트와 삭제](docs/09-update-and-cleanup.md)를 따라 지웁니다.

## 범위 밖

원본 리포는 추론 게이트웨이까지만 다룹니다. Claude Desktop 사용자별 설정 전달은 별도 샘플인
[claude-apps-gateway-bootstrap](https://github.com/aws-samples/anthropic-on-aws/tree/main/claude-apps-gateway-bootstrap)
에 있으나, 그 애드온은 이 리포의 스택 출력과 바로 결합되지 않습니다.

## License

이 가이드는 [MIT](LICENSE), 원본 리포에서 가져온 소스와 문서는 [MIT-0](docs/original/LICENSE) 입니다.
