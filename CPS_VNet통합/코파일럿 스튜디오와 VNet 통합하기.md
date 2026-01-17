
# Power Platform VNet Integration & Azure VNet Peering Guide

## 🔗 참고 문서

[VNet Integration 전체 가이드](https://learn.microsoft.com/ko-kr/power-platform/admin/vnet-support-setup-configure#option-1-use-the-power-platform-admin-center)

## 기본 설명

### 1. 목적
- Copilot Studio 에이전트가 Azure 리소스나 내부 시스템에 연결할 때 **공용 인터넷을 거치지 않고 프라이빗 네트워크(VNet)**를 통해 안전하게 통신.
- Public SaaS인 Copilot Studio가 외부 인터넷에 공개되지 않은 **사내 내부망(Private VNet)**의 데이터/API에 안전하게 접근할 수 있도록 네트워크 경로 개방.



### 2. 작동 방식
- Power Platform 환경에 Virtual Network 지원 활성화.
- Copilot Studio 에이전트가 프라이빗 엔드포인트를 통해 Azure Key Vault, Application Insights, Azure SQL 등과 연결.
- SQL Server 커넥터 등 VNet 지원 커넥터 사용 가능.



### 3. 장점
- 데이터가 인터넷에 노출되지 않음 → 보안 강화.
  - 참고: CPS 커넥터를 그냥 쓰면 Public Endpoint를 통해 연결된다.

- 기업 네트워크 정책 준수 가능. Private DNS 등 네트워크 제어 가능 


### 4. 필수 조건
- Power Platform 환경이 **Managed Environment**여야 함.
- VNet과 서브넷을 Azure에서 설정 후 Power Platform에 연결.
- 관리자 권한 필요.


### 5. 사전 준비
1. Managed Environment 설정
2. 권한:
   - Power Platform Tenant Admin 또는 Environment Admin
   - Azure Network Contributor
3. Azure Subscription: VNet, Subnet, Private Endpoint 생성 가능
4. PowerShell 준비: Azure PowerShell 모듈 설치, ARM 템플릿 배포 가능



### 6. 구성 고려 사항
- VNet과 Power Platform 환경 지역 일치
- 서브넷 크기 및 Delegation 설정



### 7. 샘플 아키텍처
- PP 환경: 미국(us)
- Primary VNet: eastus
- Secondary VNet: westus
- 두 VNet 간 피어링 설정 후 Enterprise Policy 연결

<img width="1019" height="627" alt="Image" src="https://github.com/user-attachments/assets/2bd413b1-8907-464f-becc-d73d022d40f7" />


##  Step-by-Step Guide

### 1. Power Platform 준비
- Managed Environment 설정
- 기존 커넥터가 Public Endpoint 사용 시 Private Endpoint로 변경
- Microsoft Entra에서 Power Platform Administrator 역할 할당
  -  Microsoft Entra 관리 센터에서 Power Platform 관리자 역할 할당 (to Vnet Integration 진행할 테넌트 내 계정)
  Roles & Admins에서 Power Platform Administrator 로 add assignments 

### 2. Azure 준비
- Azure portal에서 Resource Group에 Network Contributor 역할 부여
  - VNet이 들어 있는 Resource Group 에 
        (IAM) → Add → Add role assignment
        Role: Network Contributor
  *연습용이니까 Active, permanent로 설정 
- VNet 생성 및 서브넷 생성 (PP 환경과 동일 지역)
- 서브넷 Delegation 설정:
```bash
az network vnet subnet create \
  --name {primarySubnetName} \
  --resource-group {resourceGroup} \
  --vnet-name {primaryVnetName} \
  --address-prefix 10.0.1.0/24 \
  --delegations Microsoft.PowerPlatform/enterprisePolicies
```
- VNet Peering 설정 (장애조치용으로 미국 내 VNet 2개 각각 만들었으니까 Peering 필요) - 양방향 Peering 생성 필요. 아래에는 하나만 기재. 

      $RG = "practice-260107"
      $VNET_EAST = "VnIntegrationWithPP"
      $VNET_WEST = "VNetIntegrationPP_WestUS"
      $SUB_ID = (az account show --query id -o tsv)
      
      az network vnet peering create --resource-group $RG  --vnet-name $VNET_WEST   --name "peer-$VNET_WEST-to-$VNET_EAST" --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET_EAST" --allow-vnet-access true --allow-forwarded-traffic true



### 3. PowerShell 스크립트 실행
- 관리자 권한으로 PowerShell 실행
- 스크립트 위치에 따라 실행. 스크립트는 [링크](https://learn.microsoft.com/ko-kr/power-platform/admin/vnet-support-setup-configure#option-1-use-the-power-platform-admin-center) 참조
  - (1) Setup.ps1 : Powershell 실행 권한 확보 
  - (2 )InstallPowerAppsCmdlets.ps1: 필수 모듈 설치
  - (3) SetupSubscriptionForPowerPlatform.ps1: Azure 구독 등록 (이 구독에서 vnet 기능 쓰겠다고 리소스 공급자 등록)
  - (4) CreateSubnetInjectionEnterprisePolicy.ps1
- 실행 예시:
```powershell
.\CreateSubnetInjectionEnterprisePolicy.ps1 `
  -Location unitedstates `
  -VnetIds @("/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Network/virtualNetworks/{primaryVnetName}",
             "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Network/virtualNetworks/{secondaryVnetName}")
```
### 4. Power Platform Admin Center에서 정책 연결
- Option 1. NewSubnetInjection.ps1 위 방식과 흡사하게 실행
- Option 2.Power Platform Admin Center에서 
환경 → 설정 → Data & Privacy → Azure Virtual Network policies 선택 후 저장 
<img width="999" height="610" alt="Image" src="https://github.com/user-attachments/assets/c630052c-6f60-43e7-b6a3-e0a9e0e45e88" />


### 5. 연결 확인 
- PPAC에서 활동 로그(Activity Log) 확인 → 상태가 성공으로 표시되면 정상.
<img width="2039" height="571" alt="Image" src="https://github.com/user-attachments/assets/89842a72-ad24-405d-b0c2-88aef12c7f0d" />
- PowerShell:

```powershell
Get-AzResource -ResourceType "Microsoft.PowerPlatform/enterprisePolicies"
```

## Tip 정리

### 1. VNet 통합은 환경 단위로 적용 

- Enterprise Policy는 Power Platform 환경에 연결되므로, 한 번 A 환경에 정책을 연결하면 그 환경에서 실행되는 모든 앱, 플로우, Copilot Studio 에이전트가 동일한 네트워크 경로를 사용합니다.
- 따라서 User 1이 동일한 테넌트에서 A 환경에 에이전트를 만든다면, 별도의 VNet 설정을 다시 할 필요는 없습니다. 이미 A 환경이 VNet 정책을 통해 Private Endpoint로 Azure 리소스에 접근할 수 있도록 구성되어 있기 때문입니다.
- 단, 주의할 점:

  - 에이전트가 사용하는 커넥터가 VNet 지원 커넥터여야 합니다.
  - 퍼블릭 엔드포인트는 반드시 차단하고 Private Endpoint가 정상 연결 상태인지 확인해야 합니다.
  - 환경이 Managed Environment로 유지되고, 정책이 활성화 상태인지 PPAC(Admin Center)에서 확인하세요.


  
#  Power Platform VNet Integration + Azure SQL Database 테스트 가이드

## 개요
Power Platform 환경(Copilot Studio 포함)을 **Azure VNet**과 통합하고, **Azure SQL Database**를 Private Endpoint로 연결하여 퍼블릭 네트워크를 차단한 상태에서 사설 IP로 접근하는 시나리오를 테스트합니다.

Public SaaS인 Copilot Studio가 내부망(Private VNet) 데이터/API에 접근
Power Platform Virtual Network 지원 활성화
Private Endpoint + Private DNS Zone으로 Azure SQL Database 연결
퍼블릭 네트워크 접근 차단 → 기업 보안 정책 준수

## 🔗 참고 문서
- [Microsoft Entra-Only 인증을 사용하도록 설정된 서버 만들기](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-overview)
- [Microsoft Entra 인증에 대한 Azure 정책](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-overview)

## 🛠 사전 준비
- Azure 권한: `구독 소유자` 또는 `SQL Server Contributor` 역할 필요
- Power Platform 환경: Managed Environment + Enterprise Policy 연결
- Azure 리소스 공급자 등록: `Microsoft.Sql`

```bash
az provider register --namespace Microsoft.Sql
```

- 변수 설정

```bash
SUBSCRIPTION_ID="<YOUR_SUBSCRIPTION_ID>"
RG_NAME="<YOUR_RESOURCE_GROUP>"
VNET_PRIMARY="<PRIMARY_VNET_NAME>"
VNET_SECONDARY="<SECONDARY_VNET_NAME>"
SUBNET_PRIVATE="<PRIVATE_ENDPOINT_SUBNET>"
SQL_SERVER_NAME="<SQL_SERVER_NAME>"
DB_NAME="TestDatabase"
ADMIN_UPN="<ADMIN_UPN>" # 예: admin@<tenant>.onmicrosoft.com
ADMIN_OBJECT_ID="<ADMIN_OBJECT_ID>"
LOCATION="westus" # SQL 서버 생성 위치
```

##  Step-by-Step Guide

### 1. Azure SQL Server 생성 (Microsoft Entra 인증)
```bash
az sql server create   --enable-ad-only-auth   --external-admin-principal-type User   --external-admin-name $ADMIN_UPN   --external-admin-sid $ADMIN_OBJECT_ID   -g $RG_NAME   -n $SQL_SERVER_NAME   -l $LOCATION
```

### 2. 데이터베이스 생성
```bash
az sql db create   --resource-group $RG_NAME   --server $SQL_SERVER_NAME   --name $DB_NAME   --service-objective Basic
```

### 3. Private Endpoint 생성 (SQL 서버와 동일 리전 VNet 서브넷)
```bash
az network private-endpoint create   --name practicePE   --resource-group $RG_NAME   --vnet-name $VNET_SECONDARY   --subnet $SUBNET_PRIVATE   --private-connection-resource-id "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME/providers/Microsoft.Sql/servers/$SQL_SERVER_NAME"   --group-id sqlServer   --connection-name "conn-sql-west"   --location $LOCATION
```

### 4. Private DNS Zone 생성
```bash
az network private-dns zone create   --resource-group $RG_NAME   --name "privatelink.database.windows.net"
```

### 5. DNS Zone을 VNet에 연결
```bash
az network private-dns link vnet create   --resource-group $RG_NAME   --zone-name "privatelink.database.windows.net"   --name "dns-link-west"   --virtual-network $VNET_SECONDARY   --registration-enabled false
```
> **중요:** East US VNet도 링크해야 함 (Peering 환경)

### 6. DNS Zone 그룹 생성 (Private Endpoint와 DNS 자동 동기화)
```bash
az network private-endpoint dns-zone-group create   --resource-group $RG_NAME   --endpoint-name practicePE   --name "practice-group"   --private-dns-zone "privatelink.database.windows.net"   --zone-name "privatelink.database.windows.net"
```

### 7. 퍼블릭 네트워크 접근 차단
```bash
az sql server update   --resource-group $RG_NAME   --name $SQL_SERVER_NAME   --enable-public-network false
```


##  테스트: SQL 데이터 삽입 및 차단 확인

```sql
# 1.
az provider show --namespace Microsoft.Sql --query registrationState  

# 2.
az sql server create --enable-ad-only-auth --external-admin-principal-type User --external-admin-name {adminName} --external-admin-sid {adminSID} -g {resourceGroup} -n pp-sql-westus -l westus  

# 3.
az sql db create --resource-group practice-260107 --server pp-sql-westus --name "TestDB" --service-objective Basic   
``` 

### [단계 1] 테스트 데이터 넣기 (잠깐 문 열기)
1. Azure Portal → SQL Server 접속
2. Networking → Public network access → **Selected networks**
3. Firewall rules → **Add client IPv4 address**
4. Query editor (preview) → 로그인 (AD 인증)

SQL 실행:
```sql
CREATE TABLE Employees (
    EmployeeID int,
    FirstName varchar(255),
    JobTitle varchar(255),
    Salary int
);
INSERT INTO Employees VALUES (1, 'Iron Man', 'Hero', 10000);
INSERT INTO Employees VALUES (2, 'Captain America', 'Leader', 500);
SELECT * FROM Employees;
```

### [단계 2] 문 잠그기 (핵심)
- Networking → Public network access → **Disabled**
- Save → Query Editor 접속 불가 확인 ✅


##  결과
- SQL 서버는 Private Endpoint + DNS Zone을 통해 VNet 사설 IP로 접근 가능
- 퍼블릭 엔드포인트는 완전히 차단됨
- Power Platform Copilot Studio에서 VNet 지원 커넥터로 연결 테스트 성공

