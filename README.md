# Azure Container Apps 핸즈온랩
Azure Container Apps 핸즈온랩에 오신 것을 환영합니다! 이 핸즈온랩은 엔터프라이즈 수준에서 Azure Container Apps 도입을 고려할 때 확인이 필요한 항목들을 안내합니다. Azure Container Apps에 대한 기초적인 이해를 가진 상태에서 시작하면 더욱 좋습니다.

## 목차

1. [소개 및 목표](#1-소개-및-목표)
2. [사전 준비 사항](#2-사전-준비-사항)
3. [Azure 리소스 그룹 및 가상 네트워크(VNet) 설정](#3-azure-리소스-그룹-및-가상-네트워크vnet-설정)
4. [서브넷 및 네트워크 보안 그룹(NSG) 구성](#4-서브넷-및-네트워크-보안-그룹nsg-구성)
5. [Azure Container Registry(ACR) 및 Key Vault 설정](#5-azure-container-registryacr-및-key-vault-설정)
6. [Container Apps 환경 생성](#6-container-apps-환경-생성)
7. [ACA를 외부 및 내부로 배포하기](#7-aca를-외부-및-내부로-배포하기)
8. [프라이빗 엔드포인트 구성 및 보안 설정](#8-프라이빗-엔드포인트-구성-및-보안-설정)
9. [VM에서 ACA에 접근하여 내부 IP 확인](#9-vm에서-aca에-접근하여-내부-ip-확인)
10. [서비스 부하 분산 구성](#10-서비스-부하-분산-구성)
11. [마무리](#11-마무리)
---

## 1. 소개 및 목표

### 목표

- **목적**: Azure Container Apps(ACA)를 사용하여 엔터프라이즈급 애플리케이션을 도입 및 개발하고자 할 때, 반드시 고려해야 할 보안 설정을 확인하고 네트워크 접근 방법을 이해합니다.

### 학습 내용

- Virtual Network 및 Subnet 구성의 이유와 NSG의 역할을 이해합니다.
- 내부 및 외부 ACA의 배포 방법과 차이점을 학습합니다.
- Azure Key Vault와 Azure Container Registry(ACR)의 필요성과 설정 방법을 알아봅니다.
- 프라이빗 엔드포인트의 사용 이유와 설정 방법을 이해합니다.
- 서비스의 안정적인 운영을 위한 부하 분산에 대한 세션을 설정합니다.

---

## 2. 사전 준비 사항

- **Azure 구독**: Azure 리소스를 사용할 수 있는 활성화된 Azure 구독.
- **기본 지식**: Azure 서비스, 네트워킹 개념, 컨테이너에 대한 기본 이해.
- **도구**: Azure CLI, Docker CLI, 그리고 Azure Portal 

---

## 3. Azure 리소스 그룹 및 가상 네트워크(VNet) 설정

### 3.1. 리소스 그룹 생성

1. [Azure Portal](https://portal.azure.com)에 로그인합니다.
2. **리소스 그룹**을 검색하고 **만들기**를 선택합니다.
3. 다음 정보를 입력합니다.:
   - **구독**: 사용할 구독 선택.
   - **리소스 그룹 이름**: `RG-ACA-HoL`.
   - **지역**: 원하는 지역 선택 (예: `Korea Central`).
4. **검토** 후 리소스를 생성합니다.

### 3.2. 가상 네트워크(VNet) 생성
1. Azure Portal에서 **Virtual Network**를 검색하고 리소스를 생성합니다.
2. 다음 정보를 입력합니다:
   - **이름**: `ACA-VNet`.
   - **지역**: 리소스 그룹과 동일하게 설정.
   - **주소 공간**: `10.0.0.0/16`.
3. 서브넷 추가:
   - **서브넷 이름**: `ACA-Subnet-External`.
   - **서브넷 주소 범위**: `10.0.0.0/24`.
4. (Internal과 구분하여 추가 희망 시) :
   - **서브넷 이름**: `ACA-Subnet-Internal`.
- **서브넷 주소 범위**: `10.0.0.1/24`.
4. **검토 + 만들기**를 클릭하고 **만들기**를 클릭합니다.

---

## 4. 서브넷 및 네트워크 보안 그룹(NSG) 구성

### 4.1. 네트워크 보안 그룹(NSG) 생성
> 💡 **Note:** ACA를 도입할 때 NSG를 반드시 사용할 필요는 없습니다. NSG 사용 여부는 ACA의 네트워크 구성과 보안 요구 사항에 따라 결정됩니다. 기본적으로 격리된 상태에서 ACA가 운영되며, Ingress 설정만으로 외부 접근을 차단하거나 허용할 수 있습니다.
1. Azure Portal에서 **Network security group**을 검색하고 **만들기**를 선택합니다.
2. 다음 정보를 입력합니다:
   - **이름**: `NSG-ACA-HOL`.
   - **지역**: VNet과 동일하게 설정.
   - **리소스 그룹**: `RG-ACA-HoL`.
3. **만들기**를 클릭합니다.

### 4.2. NSG를 서브넷에 연결

1. **ACA-VNet**으로 이동하여 **서브넷**을 선택합니다.
2. **ACA-Subnet**을 선택하고 **네트워크 보안 그룹**을 설정합니다.
3. **NSG-ACA-HOL**를 서브넷에 연결합니다.

### 4.3. NSG 규칙 구성

1. **NSG-ACA-HOL**로 이동합니다.
2. 필요한 인바운드 보안 규칙을 추가합니다 (예: 자주 사용되는 HTTP 포트 80 및 HTTPS 443 허용).

---

## 5. Azure Container Registry(ACR) 및 Key Vault 설정

### 5.1. Azure Container Registry(ACR) 생성

1. **컨테이너 레지스트리**를 검색하고 **만들기**를 선택합니다.
2. 다음 정보를 입력합니다:
   - **레지스트리 이름**: `ACAHoL`.
   - **리소스 그룹**: `RG-ACA-HoL`.
   - **위치**: 리소스 그룹과 동일하게 설정.
   - **SKU**: `Premium`. (프라이빗 엔드포인트 생성하기 위함)
3. **검토 + 만들기**를 클릭하고 **만들기**를 클릭합니다.

### 5.2. Azure Key Vault 생성

1. **Key Vault**를 검색하고 **만들기**를 선택합니다.
2. 다음 정보를 입력합니다:
   - **이름**: `ACA-HoL-KeyVault`.
   - **지역**: 리소스 그룹과 동일하게 설정.
   - **리소스 그룹**: `RG-ACA-HoL`.
3. **검토 + 만들기**를 클릭하고 **만들기**를 클릭합니다.

### 5.3. ACR 구성

1. 로컬에서 Docker 이미지 파일을 받습니다. (예, Nginx 웹 서버)
- `docker pull nginx:latest`
  - 컨테이너 이미지를 레지스트리에 푸시합니다.
2. Azure CLI를 통해 생성한 ACR에 접속합니다.
- `az acr login --name ACAHoL `
3. Docker 이미지에 태그를 설정합니다.
- `docker tag nginx:latest acahol.azurecr.io/nginx:latest`
4. Docker이미지를 Push합니다.
- `docker push acahol.azurecr.io/nginx:latest`

## 6. Container Apps 환경 생성

### 6.1. Container Apps 환경 생성

1. **Container Apps Environments**를 검색하고 **만들기**를 선택합니다.
2. 다음 정보를 입력합니다:
   - **이름**: `ACA-Env-External`.
   - **리소스 그룹**: `RG-ACA-HoL`.
   - **지역**: 리소스 그룹과 동일하게 설정.
3. **가상 네트워크 구성**:
   - **VNet 통합 사용**: 예.
   - **VNet**: `ACA-VNet` 선택.
   - **서브넷**: `ACA-Subnet-External` 선택.
   - **가상 IP 유형**: 외부 ACA의 경우 `Public` 선택.
4. **검토** 후 리소스를 생성합니다.

---

## 7. ACA를 외부 및 내부로 배포하기

### 7.1. 외부 ACA 배포

1. **Container Apps**를 검색하고 **만들기**를 선택합니다.
2. 다음 정보를 입력합니다:
   - **이름**: `ACA-External`.
   - **Container Apps 환경**: `ACA-Env-External`.
   - **컨테이너 이미지**: ACR에서 이미지 선택.
3. **Ingress 설정**:
   - **Ingress 활성화**: 예.
   - **Ingress 유형**: `Public`.
   - **모든 트래픽 허용**: 예.
   - **포트**: `80`.
4. **검토 + 만들기**를 클릭하고 **만들기**를 클릭합니다.

### 7.2. 내부 ACA 배포

1. 내부 ACA를 위한 새로운 서브넷 생성 (예: `ACA-Subnet-Internal`).
2. 새로운 Container Apps 환경 생성:
   - **이름**: `ACA-Env-Internal`.
   - **가상 IP 유형**: `Internal`.
   - **서브넷**: `ACA-Subnet-Internal`.
3. 새로운 컨테이너 앱 배포:
   - **이름**: `ACA-Internal`.
   - **Container Apps 환경**: `ACA-Env-Internal`.
   - **Ingress 유형**: `Internal`.

---

## 8. 프라이빗 엔드포인트 구성 및 보안 설정

### 8.1. Key Vault와 ACR에 프라이빗 엔드포인트 구성

1. **Key Vault**와 **ACR** 리소스로 이동합니다.
2. **네트워킹** 설정에서 **프라이빗 엔드포인트**를 추가합니다.
3. VNet(`ACA-VNet`)과 적절한 서브넷을 선택합니다.

### 8.2. NSG 규칙 업데이트

- NSG 규칙이 프라이빗 엔드포인트로의 트래픽을 허용하는지 확인합니다.

---

## 9. VM에서 ACA에 접근하여 내부 IP 확인

### 9.1. 가상 머신(VM) 생성

1. `ACA-VNet` 내에 별도의 서브넷(예: `VM-Subnet`)에 VM을 생성합니다.
2. VM이 `ACA-Subnet-Internal`에 접근할 수 있도록 설정합니다.

### 9.2. VM에 연결

- **SSH** 또는 **Azure Bastion**을 사용하여 VM에 연결합니다.

### 9.3. Internal ACA에 접근 확인

- `curl` 또는 `wget`을 사용하여 Internal ACA의 IP 또는 도메인에 요청을 보냅니다.
- 예 : - `curl 12.34.56.78  `
---

## 10. 서비스 부하 분산 구성

### 10.1. ACA 스케일링 설정 업데이트

1. **ACA-Internal**로 이동합니다.
2. **스케일링** 설정에서 인스턴스를 2개 이상으로 조정합니다.

### 10.2. Session Affinity 설정

1. **ACA-Internal**의 **Ingress** 설정에서 **Session Affinity**를 활성화합니다.
2. 변경 사항을 저장합니다.
3. 클라이언트의 요청이 동일한 백엔드 인스턴스로 라우팅되는지 확인합니다.
4. VM에서 ACA에 접근하여 세션이 동일한 인스턴스에 유지되는지 확인합니다.
- **L4 로드밸런서**: 전송 계층(TCP/UDP)에서 동작.
- **L7 로드밸런서**: 응용 계층(HTTP/HTTPS)에서 동작.



- **질문**: 궁금한 점이나 추가 설명이 필요한 부분이 있으면 질문해 주세요.

---

# 참조

- **키리스 인증**:
  - Managed Identity를 사용하여 Key Vault와 ACR에 키 없이 접근할 수 있습니다.
- **Dedicated ACA**:
  - 전용 인프라에서 애플리케이션을 배포하고 실행할 수 있도록, 성능과 격리를 향상시키기 위해 전용 ACA를 사용할 수 있습니다.

---

## 11. 마무리
이번 핸즈온랩을 통해 보안과 네트워킹을 고려하여 Azure Container Apps를 배포하는 방법을 대해 알아봤습니다. 엔터프라이즈 보안 요구 사항을 충족하면서 필요한 경우 내부 및 외부에서 접근할 수 있도록 ACA를 설정하는 방법에 대해 다뤘습니다.

또한, 핸즈온랩을 끝마치고 나서는 Azure 내 리소스를 정리하여 불필요한 비용이 발생하지 않도록 유의하시기 바랍니다.

# 작성자
[어거스트 리]( https://github.com/CodeinAugust)

---
