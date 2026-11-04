# VNet 보안·운영 옵션

실무 질문이다. NSG 인바운드·아웃바운드 규칙 말고 VNet에 흔히 붙이는 옵션을 정리했다. 대부분 AZ-104 가상 네트워킹 영역에도 나온다.

```mermaid
flowchart LR
    U["인터넷 사용자"] --> FD["Front Door 또는 Application Gateway + WAF"]
    FD --> LB["부하 분산 장치"]
    subgraph VNET["VNet"]
        LB --> VM["VM 서브넷 · NSG"]
        VM -->|"UDR 0.0.0.0/0"| FW["Azure Firewall"]
        BA["Bastion"] --> VM
        PE["프라이빗 엔드포인트"]
    end
    FW --> OUT["인터넷"]
    VM --> PE
    ADM["관리자"] --> BA
```

## 국가별 IP 차단

NSG에는 국가 조건이 없다. NSG가 아는 건 IP 주소, 포트, 서비스 태그(Internet, AzureCloud, Storage 같은 Azure 서비스 IP 묶음)뿐이다.

| 트래픽 | 방법 |
|---|---|
| 웹(HTTP·HTTPS) | Azure Front Door나 Application Gateway의 WAF에서 지역 필터(geo-match) 사용자 지정 규칙으로 국가를 막는다. 가장 흔한 방법이다 |
| 웹이 아닌 TCP·UDP | Azure 기본 기능으로는 국가 단위 차단이 없다. 국가별 IP 목록을 NSG나 Azure Firewall IP 그룹에 직접 넣거나, 서드파티 방화벽 장비를 쓴다 |

## WAF를 붙이는 곳과 비용

Azure WAF는 혼자 동작하는 제품이 아니다. 규칙 묶음인 WAF 정책을 만들어 Application Gateway나 Front Door에 붙인다.

| 붙이는 곳 | 조건 |
|---|---|
| Application Gateway | WAF_v2 등급이어야 한다 |
| Front Door Standard | 직접 만든 규칙(IP·국가 차단 등)만 된다 |
| Front Door Premium | Microsoft 관리형 규칙(SQL 인젝션, XSS 등)까지 된다. 백엔드를 Private Link로 연결하는 기능도 들어 있다 |

대략적인 미국 기준가다. 실제 금액은 가격 계산기로 확인한다.

| 구성 | 기본 요금 | 추가 |
|---|---|---|
| Front Door Standard | 월 약 35달러 | 요청 수, 전송량. 직접 만든 WAF 규칙은 정책·규칙 수만큼 소액 |
| Front Door Premium | 월 약 330달러 | 요청 수, 전송량 |
| Application Gateway Standard_v2 | 시간 과금, 월 약 180달러 | 용량 단위 |
| Application Gateway WAF_v2 | 시간 과금, 월 약 320달러 | 용량 단위 |

Application Gateway는 리전마다 하나씩 둬야 해서 리전 두 개면 WAF_v2만으로 월 600달러가 넘는다. Front Door는 전 세계에 하나만 두므로 여러 리전 웹 서비스는 Front Door Premium 하나가 더 싸고 단순한 경우가 많다. 작은 서비스는 Front Door Standard에 국가 차단 같은 직접 만든 규칙만 넣는 구성이 가장 싸다. 리전 하나에서만 운영하거나 VNet 안에서만 쓰는 내부 앱 앞에는 Application Gateway를 둔다.

## 로그 남기기

| 무엇을 | 어디서 | 남는 내용 |
|---|---|---|
| VNet을 지나는 트래픽 | Network Watcher의 VNet 흐름 로그 | 출발지·목적지 IP, 포트, 허용·거부 여부. 스토리지 계정에 쌓는다 |
| 트래픽 요약과 시각화 | 트래픽 분석(Traffic Analytics) | 흐름 로그를 Log Analytics로 모아 상위 통신 대상, 차단된 시도, 외부 접속 국가를 대시보드로 보여 준다 |
| 웹 요청과 차단 기록 | WAF 로그(진단 설정) | 요청 URL, 걸린 규칙, 차단 여부 |
| 방화벽을 지난 트래픽 | Azure Firewall 로그(진단 설정) | 허용·거부된 연결, 위협 인텔리전스 탐지 |
| 리소스 설정 변경 | 활동 로그(Activity Log) | 누가 언제 NSG 규칙을 바꿨는지 같은 관리 작업 기록 |

NSG 흐름 로그는 은퇴 예정이라 새로 만들 수 없다. 지금은 VNet 흐름 로그를 쓴다. 활동 로그는 기본 90일만 보관되므로 오래 남기려면 진단 설정으로 Log Analytics나 스토리지 계정에 내보낸다.

## 관리 접속: Azure Bastion

VM에 공용 IP를 붙이고 RDP(3389)나 SSH(22)를 인터넷에 여는 건 가장 흔한 공격 대상이다. Bastion을 VNet에 두면 관리자가 Azure 포털(또는 네이티브 클라이언트)로 Bastion을 거쳐 VM의 사설 IP로 접속한다. VM에는 공용 IP가 필요 없다. Bastion은 AzureBastionSubnet이라는 이름의 전용 서브넷이 있어야 한다.

## 나가는 트래픽 통제

들어오는 쪽만 막으면 VM 안에서 시작된 연결은 아무 데나 나간다. 나가는 쪽을 통제하는 이유는 다음과 같다.

| 이유 | 예 |
|---|---|
| 침투 뒤 피해 줄이기 | Log4Shell은 들어온 요청 하나로 서버가 공격자 서버에 접속해 악성 코드를 내려받게 했다. 나가는 연결이 막혀 있으면 이 단계에서 끊긴다 |
| 데이터 유출 막기 | 악성 패키지나 탈취된 계정이 DB 내용을 외부 저장소로 올리는 것을 막는다 |
| 허용한 곳만 나가기 | 패키지 저장소, 결제 API처럼 필요한 도메인만 열고 나머지는 거부한다. NSG는 IP와 포트만 알아서 도메인 단위는 Azure Firewall이 한다 |
| 나가는 IP 고정 | 외부 업체가 접속 허용 IP를 등록하라고 할 때 NAT Gateway나 방화벽의 공용 IP 하나로 모은다 |
| 기록 | 어느 VM이 언제 어디로 나갔는지 방화벽 로그에 남는다. 사고 조사와 감사에 쓴다 |

방화벽은 통과시키기만 하는 문이 아니다. 기본이 전부 거부이고, 허용 목록에 있는 목적지만 내보낸다. 공격자 서버는 목록에 없으니 연결이 방화벽에서 버려진다. 막히는 건 목록 밖으로 가는 연결뿐이다. 허용한 목적지를 거쳐 빼내는 경우(허용된 GitHub이나 스토리지 계정에 올리기, DNS 질의에 데이터 싣기)는 통과하므로 허용 목록을 좁게 잡고, 필요하면 TLS 검사(Azure Firewall Premium)로 내용까지 본다.

```mermaid
flowchart LR
    VM["감염된 VM"] --> FW["Azure Firewall · 기본 거부"]
    FW -->|"허용 목록 pypi.org"| OK["패키지 저장소"]
    FW -.-x|"목록에 없음 · 버림"| ATK["공격자 서버"]
```

들어오는 쪽의 리버스 프록시나 부하 분산 장치는 목적지 주소를 내부 주소로 바꿔서 넘긴다. 경로는 주소를 바꾸지 않고 다음에 어디로 넘길지만 정한다. 그래서 나가는 트래픽을 방화벽에 모으는 데 경로(UDR 0.0.0.0/0)를 쓰고, 실제 허용·거부와 주소 변환은 방화벽이 한다.

| 옵션 | 하는 일 |
|---|---|
| NAT Gateway | 서브넷의 나가는 트래픽을 고정 공용 IP로 내보낸다. 외부 업체에 우리 IP를 등록해야 할 때 쓴다 |
| Azure Firewall | 나가는 트래픽을 도메인(FQDN) 단위로 허용·차단하고, 알려진 악성 IP·도메인을 막는다. 여러 VNet의 출구를 한곳에 모은다 |
| 사용자 정의 경로(UDR) | 경로 테이블로 0.0.0.0/0을 방화벽으로 보내서 모든 외부 트래픽이 방화벽을 거치게 한다 |

새로 만드는 VNet은 기본 아웃바운드 액세스가 꺼진 비공개 서브넷이 기본이 되는 쪽으로 바뀌고 있다. VM이 인터넷으로 나가야 하면 NAT Gateway, 공용 IP, 부하 분산 장치 아웃바운드 규칙 중 하나를 명시적으로 붙인다.

## NSG를 관리하기 쉽게

| 옵션 | 하는 일 |
|---|---|
| 서비스 태그 | Azure 서비스의 IP 묶음을 이름으로 쓴다. 예: 아웃바운드에서 Storage.KoreaCentral만 허용 |
| 애플리케이션 보안 그룹(ASG) | VM을 웹, DB 같은 역할로 묶고 규칙을 IP 대신 역할로 쓴다. 예: 웹 ASG에서 DB ASG로 1433 허용 |

## 그 밖에 자주 붙이는 것

| 옵션 | 하는 일 |
|---|---|
| DDoS 보호 | 기본 인프라 보호는 무료로 항상 켜져 있다. DDoS Network Protection(유료)은 공용 IP별 맞춤 탐지, 공격 보고서, 공격 때 늘어난 비용 보상을 더한다 |
| 프라이빗 엔드포인트 + 프라이빗 DNS 영역 | PaaS 서비스를 VNet 사설 IP로 붙이고 공용 접속을 끈다 |
| Network Watcher 진단 도구 | IP 흐름 확인(어느 NSG 규칙에 걸렸는지), 유효 보안 규칙, 연결 문제 해결, 다음 홉. 연결이 안 될 때 원인을 찾는다 |

## 무료와 유료

VNet의 기본 통제 기능은 무료이고, 트래픽 내용을 들여다보거나 대신 처리해 주는 기능은 유료다. 유료 서비스는 대부분 켜 둔 시간만큼 시간 단위로 과금되고, 처리한 데이터 양만큼 추가로 붙는다. 정확한 금액은 리전과 시점에 따라 바뀌므로 Azure 가격 계산기로 확인한다.

| 구분 | 옵션 |
|---|---|
| 무료 | NSG, 애플리케이션 보안 그룹, 서비스 태그, 사용자 정의 경로, DDoS 기본 인프라 보호, 활동 로그 90일 보관 |
| 거의 무료 | 프라이빗 DNS 영역(영역과 쿼리 수 기준으로 소액), Network Watcher 진단 도구(무료 사용량 있음) |
| 데이터 양만큼 유료 | VNet 피어링(오가는 데이터 GB당), VNet 흐름 로그(수집 GB당 + 스토리지), 트래픽 분석(Log Analytics 수집량) |
| 켜 둔 시간 + 데이터 양만큼 유료 | 프라이빗 엔드포인트, NAT Gateway, Bastion, Application Gateway WAF, Front Door, Azure Firewall |
| 월 고정으로 비싼 편 | DDoS Network Protection |

Azure Firewall과 DDoS Network Protection은 작은 서비스에는 부담되는 금액이라 보통 허브에 하나만 두고 여러 VNet이 같이 쓴다. Bastion은 일부 리전에서 기능이 제한된 무료 Developer 계층을 제공한다.
