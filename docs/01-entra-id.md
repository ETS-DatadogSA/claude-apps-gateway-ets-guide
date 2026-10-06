# 1. Entra ID 앱 등록

Claude Apps Gateway용 Entra ID App과 Admin을 만들고, cdk 배포에 사용할 환경변수를 확보합니다. \
모든 명령은 Azure CLI(`az`)로 실행합니다.

| 변수 | 내용 | 쓰이는 곳 |
| --- | --- | --- |
| `APP` | 앱(client) ID | CDK 컨텍스트 `oidcClientId` |
| `OBJ` | 앱 object ID | 1.3 groups 클레임 설정 |
| `SECRET` | client secret | CDK 컨텍스트 `oidcClientSecret` |
| `GRP` | 어드민 그룹 object ID (GUID) | CDK 컨텍스트 `adminOktaGroupName`, [3단계](03-configure-source.md) |
| `ISSUER` | OIDC issuer (Entra v2) | CDK 컨텍스트 `oidcIssuer` |

## 1.1 로그인

앱 등록·그룹 생성 권한이 있는 계정으로 로그인하고, 테넌트를 확인합니다.

**로그인 명령어**
```bash
az login
```

1. 브라우저에 Microsoft **계정 선택** 화면이 열립니다. 앱 등록 권한이 있는 계정을 고릅니다. \
   브라우저가 열리지 않으면 `az login --use-device-code` 로 다시 실행합니다.

   ![az login 계정 선택 화면](images/01-az-login-account.png)

2. 터미널로 돌아오면 구독·테넌트 선택 표가 나옵니다. 앱을 등록할 테넌트의 번호를 입력합니다.

```text
No     Subscription name     Subscription ID                       Tenant
-----  --------------------  ------------------------------------  --------
[1] *  Azure subscription 1  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  기본 디렉터리

Select a subscription and tenant (Type a number or Enter for no changes): 1
```
> [!TIP]
> 개인 계정으로 Azure 에 가입하면 `기본 디렉터리` 테넌트가 자동으로 생기고, 가입한 계정이 그 테넌트의 관리자가 됩니다. \
> 구독이 없는 테넌트에는 `az login --allow-no-subscriptions` 로 로그인합니다. 선택한 테넌트와 계정을 확인합니다.

**테넌트 확인 명령어**
```bash
az account show --query "{tenant:tenantId, user:user.name}" -o table
```

**테넌트 확인 결과 예시**
```text
Tenant                                User
------------------------------------  ---------------------
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  user@example.com
```

## 1.2 앱 등록

Gateway는 client secret 을 쓰는 confidential client 입니다. \
Redirect URI 는 배포 후에 정해지므로 임시값으로 두고 [5.2](05-vpn-and-redirect.md#52-entra-리다이렉트-uri-교체)에서 바꿉니다.

**앱 등록 명령어**
```bash
APP=$(az ad app create --display-name "Claude Apps Gateway" --sign-in-audience AzureADMyOrg --web-redirect-uris "https://placeholder.invalid/oauth/callback" --query appId -o tsv) && echo "APP=$APP"
OBJ=$(az ad app show --id "$APP" --query id -o tsv) && echo "OBJ=$OBJ"
az ad sp create --id "$APP"
```

마지막 명령은 서비스 주체를 JSON 으로 출력합니다. `appId` 가 `APP` 과 같고 `replyUrls` 가 임시값이면 됩니다. \
하단의 예시는 실행 결과 예시입니다:

```json
APP=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
OBJ=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#servicePrincipals/$entity",
  "accountEnabled": true,
  "appDisplayName": "Claude Apps Gateway",
  "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "appOwnerOrganizationId": "<tenant-id>",
  ...
  "id": "<service-principal-id>",
  ...
  "replyUrls": [
    "https://placeholder.invalid/oauth/callback"
  ],
  ...
  "servicePrincipalType": "Application",
  "signInAudience": "AzureADMyOrg",
  ...
}
```

`OBJ`(앱 object ID)와 서비스 주체의 `id` 는 서로 다른 값입니다. 1.3 에는 `OBJ` 를 씁니다.

포털 **Microsoft Entra ID → 관리 → 앱 등록 → 모든 애플리케이션**에도 앱이 보입니다. \
같은 이름의 앱이 이미 있으면 `--display-name` 인자의 파라미터를 변경하여 구분합니다. \
(하단의 예시 이미지는 `Claude Apps Gateway Sample` 로 생성 하였습니다) 

![앱 등록 목록](images/01-app-registrations.png)

앱을 열면 **개요**에서 지원되는 계정 유형이 `내 조직만`, 리디렉션 URI 가 `1 웹` 으로 나옵니다.

![앱 개요](images/01-app-overview.png)

## 1.3 Group claim 활성화

로그인 토큰에 사용자가 속한 그룹 목록(`groups` 클레임)을 넣도록 앱 설정을 바꿉니다.

**왜 하나요?** Claude Apps Gateway와 관리 콘솔은 토큰의 그룹 목록으로 관리자를 판단하는데, Entra ID 는 기본값으로 그룹을 토큰에 넣지 않습니다. \
건너뛰면 어드민 그룹에 넣은 사용자도 관리자로 인식되지 않습니다.

<details>
<summary>자세히: 관리자 판별 방식</summary>

로그인하면 Entra 가 발급하는 토큰(JWT) 안에 사용자 정보가 JSON 으로 들어 있습니다. \
해당 단계를 적용하면 여기에 `groups` 항목이 생기고, 사용자가 속한 보안 그룹의 GUID 가 나열됩니다. \
아래는 JWT에 들어있는 사용자 정보의 예시입니다.

**JWT 사용자 정보 예시**
```json
{
  "email": "sample@example.com",
  "name": "Alice",
  "groups": [
    "1111aaaa-....",
    "2222bbbb-...."
  ]
}
```

위 예시에서 `2222bbbb-....` 가 Admin Group (1.5 단계에서 만드는 그룹)의 GUID 라면 이 사용자는 관리자입니다. \
Claude Apps Gateway와 관리 콘솔은 로그인한 사용자의 `groups` 목록에 어드민 그룹 GUID 가 **있으면 관리자**, **없으면 일반 사용자**로 판단합니다.

비교 기준이 되는 Admin Group GUID는 cdk 배포 전에 각 설정에 넣어 둡니다.

| 구성 요소 | 비교 기준 설정 | 값을 넣는 단계 |
| --- | --- | --- |
| Claude Apps Gateway | `gateway/gateway.yaml` 의 `admin.admin_groups` | [4단계](04-deploy.md) `-c adminOktaGroupName="$GRP"` |
| 관리 콘솔 | `admin-console/app/auth.py` 의 `ADMIN_GROUP_NAME` | [3단계](03-configure-source.md) |

Entra ID는 기본값으로 토큰에 그룹을 넣지 않습니다. 토큰에 `groups` 항목이 아예 없으면 비교할 대상이 없으므로 모든 사용자가 일반 사용자가 됩니다. 이 단계를 건너뛰면 1.5 에서 어드민 그룹에 넣은 사용자도 [6단계](06-verify.md) 콘솔 사인인에서 비관리자로 표시됩니다.

</details>

**Group claim 명령어**
```bash
az rest --method PATCH --url "https://graph.microsoft.com/v1.0/applications/$OBJ" --body "{\"groupMembershipClaims\": \"SecurityGroup\", \"optionalClaims\": {\"idToken\": [{\"name\": \"groups\", \"essential\": false}], \"accessToken\": [{\"name\": \"groups\", \"essential\": false}]}}"
```

성공하면 아무것도 출력하지 않습니다.

> [!IMPORTANT]
> 토큰에 들어가는 값은 그룹 이름(`claude-gateway-admins`)이 아니라 그룹의 **object ID(GUID)** 입니다. \
> 그래서 1.5 에서 그룹 이름이 아니라 GUID 를 `GRP` 로 받아 두고, 3·4단계 설정에도 GUID 를 넣습니다.

<details>
<summary>명령이 바꾸는 설정</summary>

| 설정 | 의미 |
| --- | --- |
| `groupMembershipClaims: "SecurityGroup"` | 사용자가 속한 보안 그룹을 토큰에 넣음 |
| `optionalClaims` 의 `idToken`·`accessToken` 에 `groups` | ID 토큰과 액세스 토큰 모두에 넣음 |

포털에서 **앱 등록 → (앱) → 토큰 구성 → 그룹 클레임 추가 → 보안 그룹**을 선택하는 것과 같습니다.

</details>

## 1.4 client secret 생성

게이트웨이용 client secret 을 만듭니다. secret 은 이때 한 번만 나옵니다. [4.3](04-deploy.md#43-컨텍스트-파일로-저장-선택)에서 파일로 저장합니다.

**왜 하나요?** 게이트웨이는 로그인 처리 중 Entra 에 토큰을 요청할 때 이 앱이 맞다는 것을 secret 으로 증명합니다(confidential client). 4단계의 `-c oidcClientSecret` 으로 넘깁니다.

> [!CAUTION]
> `--append` 를 빼면 이 앱의 기존 자격증명이 전부 삭제됩니다.

**Client Secret 생성 명령어**
```bash
SECRET=$(az ad app credential reset --id "$APP" --append --display-name gateway --years 1 --query password -o tsv) && echo "secret 생성됨 (길이 ${#SECRET})"
```

secret 값은 화면에 출력하지 않고 길이만 보여 줍니다.

## 1.5 어드민 그룹 생성

관리 콘솔 관리자를 넣을 그룹을 만들고, 현재 계정을 추가합니다.

**왜 하나요?** 1.3 에서 설명한 관리자 판별의 기준이 되는 그룹입니다. 이 그룹의 GUID(`GRP`)를 3·4단계 설정에 넣고, 이 그룹에 든 사용자만 관리 콘솔에서 관리자가 됩니다.

**Admin Group 생성 명령어**
```bash
GRP=$(az ad group create --display-name "claude-gateway-admins" --mail-nickname "claude-gateway-admins" --query id -o tsv) && az ad group member add --group "$GRP" --member-id "$(az ad signed-in-user show --query id -o tsv)" && echo "GRP=$GRP"
```

<details>
<summary>다른 관리자 추가</summary>

```bash
az ad group member add --group "$GRP" --member-id "$(az ad user show --id <user@your-domain> --query id -o tsv)"
```

</details>

## 1.6 issuer 확인

게이트웨이가 토큰 발급자로 신뢰할 Entra v2 issuer 주소를 만듭니다.

**Issuer 확인 명령어**
```bash
ISSUER="https://login.microsoftonline.com/$(az account show --query tenantId -o tsv)/v2.0" && echo "ISSUER=$ISSUER"
```

## 1.7 확인

다섯 변수가 모두 채워졌는지 봅니다. secret 은 길이만 출력합니다.

**변수 확인 명령어**
```bash
echo "APP=$APP OBJ=$OBJ GRP=$GRP ISSUER=$ISSUER SECRET_LEN=${#SECRET}"
```

다음: [2. AWS 배포 준비](02-aws-preparation.md)
