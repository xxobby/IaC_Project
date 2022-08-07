# IaC를 이용하여 Azure 클라우드에 WordPress 자동화 배포
---

## 역할 분담
| 팀원   | 역할                  |
| ------ | --------------------- |
| 도효주 | Terraform, Ansible, 보고서, 발표 |
| 박능력 | Tarraform, 보고서     |
| 성나영 | Terraform, PPT, 발표  |
| 장현오 | 코드 테스트, PPT      |

<br>

## 아키텍쳐
---

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/KakaoTalk_20220509_165545839.png" width=800/>

Bastion Host와 Web Server, DB 서버를 서로 다른 Subnet에 구성함으로써 보안을 강화하고 Application Load Balancer와 VMSS를 활용하여 외부 트래픽 분산, 클라우드의 유연성을 강화하여 가용성을 확보하였다.

<br>

## 프로젝트 환경
---

### 1. 테스트 컴퓨터 환경

Vagrant 가상 머신을 Master Server로 사용한다

|Resource|Configuration|
|-|-|
|CPU|2|
|Memory|4096MB|
|Disk|40GB|
|OS|CentOS 7|
|Hostname|controller|

<br>

### 2. 구성 관리 / 배포 도구

>위 도구들을 Managed Server(현재 Controller)에 설치한다.

| Name | Version|
|-|-|
|Terraform| v 1.1.9 on linux_amd64|
| Ansible | v 2.9.27|
||python v 2.7.5 필요|
|Azure CLI|v 2.36.0|
||Python (Linux) 3.6.8 필요|

<br>
<br>

# Terraform _Azure 구성
## 1. 환경 구성

<br>

###  1-1. Ansible 설치

rpm 파일을 다운로드한다.

```bash
$ sudo yum install -y centos-release-ansible-29
```

Ansible 패키지를 설치한다.

```bash
$ sudo yum install -y ansible
```

Ansible 패키지 설치를 확인한다.

```bash
$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/vagrant/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Nov 16 2020, 22:23:17) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
```

<br>

### 1-2. Terraform 설치

커널을 업데이트한다.

```bash
$ sudo yum install -y yum-utils
```

Hashicorp 사이트에서 Terraform을 다운로드한다.

```bash
$ sudo yum-config-manager --add-repo
https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
```

Terraform 패키지를 설치한다.

```bash
$ sudo yum -y install terraform
```

Terraform 설치를 확인한다.

```bash
$ terraform --version
Terraform v1.1.9
on linux_amd64
+ provider registry.terraform.io/hashicorp/aws v3.75.1
```

<br>

### 1-3. Azure CLI 설치 및 로그인

Microsoft로 부터 레포지토리 키를 가져온다.

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
```

CentOS 7의 경우, Azure-cli 리포지토리를 추가한다.

```bash
echo -e "[azure-cli]

name=Azure CLI

baseurl=https://packages.microsoft.com/yumrepos/azure-cli

enabled=1

gpgcheck=1

gpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/azure-cli.repo
```

dnf(리눅스 패키지 관리도구)을 설치한다.

```bash
$ sudo yum install epel-release

$ sudo yum install dnf
```

Azure CLI를 설치한다.

```shell
$ sudo dnf install azure-cli
```

Azure CLI 도구 사용을 위해 구독 계정을 인증 한다.

```bash
$ az login --use-device-code
```

실행 결과에서 계정 `id` 를 확인 한다.

```bash
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code H3CT4VJ8L to authenticate.
[
 {
 ...
 "id": "fbb2f75b-3d1e-4dfe-943e-d46d2f84cf2d",
 ...
 }
```

구독 계정을 설정 한다.

```bash
az account set --subscription "35akss-subscription-id"
```

작업을 대신 수행할 서비스 주체를 생성 한다.

```bash
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/<SUBSCRIPTION_ID>"
```
```shell
Creating 'Contributor' role assignment under scope '/subscriptions/35akss-subscription-id'

The output includes credentials that you must protect. Be sure that you do not include these credentials in your code or check the credentials into your source control. For more information, see https://aka.ms/azadsp-cli
{
 "appId": "xxxxxx-xxx-xxxx-xxxx-xxxxxxxxxx",
 "displayName": "azure-cli-2022-xxxx",
 "password": "xxxxxx~xxxxxx~xxxxx",
 "tenant": "xxxxx-xxxx-xxxxx-xxxx-xxxxx"
}
```

위 실행 결과 값을 사용하여 환경 변수를 설정 한다.

```shell
$ $Env:ARM_CLIENT_ID = "<APPID_VALUE>"

$ $Env:ARM_CLIENT_SECRET = "<PASSWORD_VALUE>"

$ $Env:ARM_SUBSCRIPTION_ID = "<SUBSCRIPTION_ID>"

$ $Env:ARM_TENANT_ID = "<TENANT_VALUE>"
```

<br>
<br>

---

## 2. 리소스 생성 및 배포

<br>

### 사용한 리소스 목록

|Resource|Description|
|-|-|
|resource_group|Azure 솔루션에 관련된 리소스를 보유하는 컨테이너|
|virtual_network|Azure Cloud에 격리된 네트워크 구성|
|subnet|Azure 서브넷 생성 리소스|
|network_security_group|Azure 가상 네트워크의 Azure 리소스와 주고받는 네트워크 트래픽 필터링|
|public_ip|  Azure 리소스가 인터넷 및 공용 Azure 서비스와 통신하기 위한 IP 주소|
|network_interface|Azure Virtual Machine와 인터넷, Azure 및 온-프레미스 리소스와 통신 담당|
|linux_virtual_machine|확장 가능한 주문형 컴퓨팅 리소스를 제공하는 이미지 서비스 인스턴스 |
|linux_virtual_machine_scale_set|다수의 VM을 중앙에서 관리, 구성 및 업데이트할 수 있게하여 고가용성 보장하는 서비스|
|AzureDatabase for Maira DB Server|오픈 소스 MariaDB 서버 엔진을 기반으로 하는 관계형 데이터베이스 서비스|
|mariadb_database|클라우드용 최신 관계형 데이터베이스 서비스|
|lb|웹 애플리케이션에 대한 트래픽 관리, 백 엔드 리소스 또는 서버의 그룹에서 로드를 효율적으로 분산|


<br>

### 2-1. Azure_Provider 지정
---

 [azure](https://registry.terraform.io/providers/hashicorp/azurerm/latest) Provider를 통해 Infra Structure를 사용할 것임을 지정한다. 3.0.2이상 3.1.0 미만 버전의 Provider를 사용하며,해당 파일이 존재하는 디렉토리에서 `terraform init` 명령을 실행하여 Provider를 설치한다.

##### `provider.tf`

```
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0.2"
    }
  }

  required_version = ">= 1.1.0"
}

provider "azurerm" {
  features {}
}
```


```shell
[vagrant@controller azure]$ terraform init
```
<br>
<br>

### 2-2. 리소스 그룹 생성
---

Azure는 모든 리소스를 리소스 그룹으로 관리할 수 있다. 리소스 그룹은 리소스를 조직적으로 관리할 수 있도록 해준다.


#####  `main.tf`

>리소스 : azurerm_resource_group

```
resource "azurerm_resource_group" "wp-rg" {

 name     = var.rg-name
 location = var.az-region

}
```

##### `variable.tf`

```
variable "rg-name" {

 description = "resource_group_name"
 type        = string
 default     = "wp-resource-group"

}

  
variable "az-region" {

 description = "azurerm_virtual_network Region"
 type        = string
 default     = "koreacentral"

}
```
<br>

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/%EB%A6%AC%EC%86%8C%EC%8A%A4%20%EA%B7%B8%EB%A3%B9.png" width=800/>

<br>
<br>

### 2-3. 보안그룹 생성
---
#### 2-3-1. Bastion Host 보안그룹

SSH 프로토콜을 사용하여 Bastion Host에 로그인하고 구성할 수 있도록 현재 IP 주소(사용자 IP)에서 들어오는 SSH 트래픽을 허용한다.

##### `security_group.tf`

 >리소스 : azurerm_network_security_group

```
resource "azurerm_network_security_group" "bastion-sg" {

 name                = "bastion-sg"
 location            = azurerm_resource_group.wp-rg.location
 resource_group_name = azurerm_resource_group.wp-rg.name
  
 security_rule {

 name                       = "SSH"
 priority                   = 1001
 direction                  = "Inbound"
 access                     = "Allow"
 protocol                   = "Tcp"
 source_port_range          = "*"
 destination_port_range     = "22"
 source_address_prefix      = "[사용자 IP]/32"
 destination_address_prefix = "*"

 }
}
```

<br>

#### 2-3-2. Web Server VMSS 보안그룹

SSH 및 HTTP 프로토콜을 사용하여 VMSS의 VM에 접속할 수 있도록  SSH와 HTTP 트래픽의 인바운드를 허용한다.

##### `security_group.tf`

 >리소스 : azurerm_network_security_group

```
resource "azurerm_network_security_group" "vmss-sg" {
  name                = "wmss-sg"
  location            = azurerm_resource_group.wp-rg.location
  resource_group_name = azurerm_resource_group.wp-rg.name

  security_rule {
    name                       = "Allow SSH" # 이름
    description                = "Allow SSH"
    priority                   = 1001 # 규칙 우선순위
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow HTTP"
    description                = "Allow HTTP"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}
```

<br>
<br>

## 2-4. Virtual Network 구성
---

### 2-4-1. Virtual Network 생성

 Wordrpress 리소스를 실행하기 위한 가상 네트워크를 생성한다.

##### `main.tf`

 >리소스 : azurerm_virtual_network

```
 resource "azurerm_virtual_network" "wp_vnet" {

 name                = "myTFVnet"
 address_space       = ["10.0.0.0/16"]
 location            = var.az-region
 resource_group_name = azurerm_resource_group.wp-rg.name

}
```

|Argument|Value|Description|type|
|-|-|-|-|
|name|myTFVnet|가상 네트워크의 이름|string|
|address_space|10.0.0.0/16|가상 네트워크의 CIDR 블록|list
|location|koreacentral|가상 네트워크가 생성 될 지역|string|
|resource_group_name|wp-sg|해당 네트워크를 설치할 리소스 그룹명|string|

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/%EA%B0%80%EC%83%81%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC.png" width=800/>

<br>

### 2-4-2. Public Subnet 생성

Public Subnet을 생성하여 외부에서 해당 CIDR 블록 내의 리소스로의 접근을 허용한다. 해당 서브넷에는 Bastion Host가 위치한다.  

##### `pub.tf`

>리소스 : azurerm_subnet

```
resource "azurerm_subnet" "wp-public-subnet" {

 name                 = "wp-pub-subnet"
 resource_group_name  = azurerm_resource_group.wp-rg.name
 virtual_network_name = azurerm_virtual_network.wp_vnet.name
 address_prefixes     = ["10.0.0.0/24"]
 
}
```

|Argument|Value|Description|type|
|-|-|-|-|
|name|wp-pub-subnet|Public Subnet 이름|string|
|address_prefixes|10.0.0.0/24|Public Subnet CIDR|list|

<br>

#### Public Subnet IP 생성
Public Subnet에 공용 IP를 동적으로 할당한다.

##### `pub.tf`

>리소스 : azurerm_public_ip

```
resource "azurerm_public_ip" "wp-publicip" {

 name                = "wp-publicip"
 location            = azurerm_resource_group.wp-rg.location
 resource_group_name = azurerm_resource_group.wp-rg.name
 allocation_method   = "Dynamic"

}
```

<br>

#### Public Subnet NIC 생성

Public Subnet에 연결할 네트워크 인터페이스를 생성한다. 해당 인터페이스에는 Public Subnet의 공용 IP가 부여된다.

##### `pub.tf`

>리소스 : azurerm_network_interface

```
resource "azurerm_network_interface" "wp-pub-nip" {

 name                = "wp-pub-nic"
 location            = azurerm_resource_group.wp-rg.location
 resource_group_name = azurerm_resource_group.wp-rg.name
  
 ip_configuration {

 name                          = "internal"
 subnet_id                     = azurerm_subnet.wp-public-subnet.id
 private_ip_address_allocation = "Dynamic"
 public_ip_address_id          = azurerm_public_ip.wp-publicip.id

 }
}
```

<br>

#### Public Subnet NIC에 보안 그룹 연결

Public 네트워크 인터페이스와 Bastion Host의 보안 그룹을 연결한다.

##### `pub.tf`

>리소스 : azurerm_network_interface_security_group_association

```
resource "azurerm_network_interface_security_group_association" "pub-association" {

 network_interface_id      = azurerm_network_interface.wp-pub-nip.id
 network_security_group_id = azurerm_network_security_group.bastion-sg.id

}
```

<br>
<br>

## 2-5. Virtual Machine 생성
---

### 2-5-1. Bastion VM 생성

Web Server의 보안 향상을 위하여 SSH 접속용 Bastion Host VM을 생성한다.<br>해당 VM은 Public IP가 연결된 Public Subnet에 존재하며, 앞서 SSH 프로토콜을 통해 관리자만이 접속할 수 있도록 구성한 네트워크 인터페이스 `wp-pub-nip`를 연결한다.

#### 하드웨어 구성

- 사이즈는 기본으로 제공되는 Standard_F2를 선택한다.

- 내부 OS 디스크는 읽고 쓰기가 가능하도록 설정하고 이를 백업할 저장소는 Standard_LRS로 선택한다.

- CentOS 7.5 운영체제 이미지를 사용하여 리눅스 기반의 VM을 생성한다.

- Public Subnet의 네트워크 인터페이스를 연결한다.

##### `main.tf`  

>리소스 : azurerm_linux_virtual_machine

```
resource "azurerm_linux_virtual_machine" "bastion-vm" {

 name                = "bastion-vm"
 resource_group_name = azurerm_resource_group.wp-rg.name
 location            = azurerm_resource_group.wp-rg.location
 size                = "Standard_F2"
 admin_username      = "adminuser"

 network_interface_ids = [
 azurerm_network_interface.wp-pub-nip.id,
 ]

 admin_ssh_key {

 username   = "adminuser"
 public_key = file("~/.ssh/id_rsa.pub")

 }

 os_disk {
 caching              = "ReadWrite"
 storage_account_type = "Standard_LRS"
 }


 source_image_reference {
 publisher = "OpenLogic"
 offer     = "CentOS"
 sku       = "7.5"
 version   = "latest"
 }
```

<br>

#### SSH 설정
- Bastion Vm에 접속할 Host name을  adminuser로 설정한다.
- 외부에서 SSH 접속할 수 있도록 Public IP를 부여받는다.
- Master Server의 공개키를 Bastion VM의 개인 키로 등록한다.
- SSH 점프 접속 시, 편의를 위해 Privisoiner를 사용하여 Master Server의 개인 키를 Bastion host에 복사한다.

```
 connection {

 user        = "adminuser"
 host        = self.public_ip_address
 private_key = file("/home/vagrant/.ssh/id_rsa")
 timeout     = "1m"

 }

 provisioner "file" {

 source      = "/home/vagrant/.ssh/id_rsa"
 destination = "/tmp/key_file"

 }

 provisioner "remote-exec" {

 inline = [

 "sudo cp /tmp/key_file /home/adminuser/.ssh/id_rsa",
 "sudo chmod 400 /home/adminuser/.ssh/id_rsa"

 ]
 }
}
```

<br>
<br>

### 2-5-2. Web VM 생성

Wordpress가 실행 될 Web Server VM을 생성한다.
해당 VM은 Packer를 통해 생성 된 이미지를 사용하며, VMSS를 통해 인스턴스의 크기를 auto-scaling한다.

#### 2-5-2-1.  Packer 이미지 생성

##### `centos.pkr.hcl`

- **환경 변수 및 playbook 실행 경로의 변수 설정**
환경변수를 참조하여 Web Server 이미지를 생성할 수 있도록 변수를 설정한다.

```
variable "playbook" {
  type    = string
  default = "/home/vagrant/[실행 디렉토리]/wp/wordpress.yaml"
}

variable "client_id" {
  type    = string
  default = env("AZ_ID")
}

variable "client_secret" {
  type    = string
  default = env("AZ_PASSWORD")
}

variable "subscription_id" {
  type    = string
  default = env("AZ_SUBSCRIPTION_ID")
}

variable "tenant_id" {
  type    = string
  default = env("AZ_TENANT")
}
```


- **하드웨어 구성**
기본 하드웨어 구성은 Bastion VM과 거의 동일하게 구성된다.
환경변수를 통해 지정된 Active Directory 서비스 주체를 사용할 수 있도록 한다.
Packer 빌드 결과가 저장 될 이미지의 이름과 리소스 그룹을 지정하고, SSH username을 지정한다.

```
source "azure-arm" "wp-azure-arm" {

  client_id       = var.client_id
  client_secret   = var.client_secret
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id

  managed_image_name                = "wp-centos"
  managed_image_resource_group_name = "myimg"
  ssh_username = "vagrant"

  os_type         = "Linux"
  image_publisher = "OpenLogic"
  image_offer     = "CentOS"
  image_sku       = "7_9"
  image_version   = "latest"

  location = "Korea Central"
  vm_size  = "Standard_DS2_v2"
}
```


##### `data.tf`

`image`라는 데이터 소스를 사용해 기존 리소스 그룹 정보에 접근한다.

```
data "azurerm_resource_group" "image" {
  name       = "myimg"
  depends_on = [azurerm_resource_group.wp-rg] 
}
```

`image`라는 데이터 소스를 사용해 기존 이미지 정보에 접근한다.

```
data "azurerm_image" "image" {
  name                = "wp-centos"                            
  resource_group_name = data.azurerm_resource_group.image.name 
  depends_on          = [azurerm_resource_group.wp-rg]         
}
```


- **이미지 빌드**
`ansible` Provisioner를 사용하여 관리자 권한으로 지정 경로에 위치하는 playbook을 실행시킨 VM의 이미지를 생성한다. 

```
build {
  sources = [ "source.azure-arm.wp-azure-arm"]

   provisioner "ansible" {
     extra_arguments= ["--become"]
     playbook_file = var.playbook
   }
}
```

<br>

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/image-20220509084244385.png" width=800/>



<br>

#### 2-5-2-2. VMSS (Virtual Machine Scale Sets)

##### `vmss.tf`

###### Web Server 전용 서브넷 생성
Web Server가 Auto-Scaling 될 서브넷을 생성한다.

>리소스 : azurerm_subnet

```
resource "azurerm_subnet" "vmss-prv-subnet" {
  name                 = "vmss-prv-subnet"
  resource_group_name  = azurerm_resource_group.wp-rg.name
  virtual_network_name = azurerm_virtual_network.wp_vnet.name
  address_prefixes     = ["10.0.50.0/24"]
}
```

|Argument|Value|Description|type|
|-|-|-|-|
|name|vmss-prv-subnet|VMSS Subnet 이름|string|
|address_prefixes|10.0.50.0/24|VMSS Subnet CIDR|list|


###### VMSS 구성

>리소스 : azurerm_linux_virtual_machine_scale_set

기본 구성은  Bastion Host VM과 동일하다.

```
resource "azurerm_linux_virtual_machine_scale_set" "wp-vmss" {
  name                            = "wp-vmss"
  resource_group_name             = azurerm_resource_group.wp-rg.name
  location                        = azurerm_resource_group.wp-rg.location
  sku                             = "Standard_F2"
  instances                       = 2
  admin_username                  = "adminuser"
  disable_password_authentication = true #비밀번호 인증X

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }
```

SELinux 설정을 해제해야 Database와 연결이 가능하다.

```
  custom_data = (base64encode(
    <<-EOF
      #!/bin/bash
      sudo setenforce 0
      EOF
  ))
```

Packer에서 빌드한 이미지를 기반으로 VM을 생성한다.

```
source_image_id = data.azurerm_image.image.id
```

생성될 VM에 연결할 NIC를 생성하여 VMSS 보안그룹에 연결해 SSH와 HTTP 인바운드를 허가하고, Load Balancer의 backend pool과 연결해 Web Server로 들어올 트래픽을 분산할 수 있게 설정한다.

```
  network_interface {
    name                      = "nic-vmss"
    primary                   = true
    network_security_group_id = azurerm_network_security_group.vmss-sg.id

    ip_configuration {
      name                                   = "IPConfiguration"
      primary                                = true
      subnet_id                              = azurerm_subnet.vmss-prv-subnet.id
      load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.lb-pool.id]
    }
  }
}
```
<br>

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/image-20220509084323230.png" width=800/>

<br>

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/image-20220509084311507.png" width=800/>



<br>
<br>

## 2-6. Database 구성
---

### 2-6-1. DB Subnet 생성

MariaDB가 위치할 Subnet을 생성한다. 서브넷과 연결할 Storage Endpoint를 생성한다.

##### `db-mysql.tf`

>리소스 : azurerm_subnet

```
resource "azurerm_subnet" "db-subnet" {

 name                 = "db-subnet"
 resource_group_name  = azurerm_resource_group.wp-rg.name
 virtual_network_name = azurerm_virtual_network.wp_vnet.name
 address_prefixes     = ["10.0.2.0/24"]
 service_endpoints    = ["Microsoft.Storage"]

 delegation {

 name = "fs"
 service_delegation {
 name = "Microsoft.DBforMySQL/flexibleServers"

 actions = [
 
 "Microsoft.Network/virtualNetworks/subnets/join/action",

 ]
 }
 }
}
```

|Argument|Value|Description|type|
|-|-|-|-|
|name|db-subnet|DB Subnet 이름|string|
|address_prefixes|10.0.2.0/24|DB Subnet CIDR|list|
|service_endpoints|Microsoft.Storage|DB Subnet Endpoint|list|

생성할 서브넷을 Azure 서비스에 위임한다.
|Argument:Delegation|Sub-Argument|Value|Description|
|-|-|-|-|
|name||fs|위임 이름|
|service_delegation|name|Microsoft.DBforMySQL/flexibleServers|위임할 서비스
||actions|Microsoft.Network/virtualNetworks/subnets/join/action|위임해야하는 작업 목록|

<br>

### 2-6-2. MairaDB  Server생성

MariaDB Server를 생성한다. DB Sever의 가격, 용량, 백업 및 중복관련 설정을 한다.
해당 서버의 데이터베이스에 접속할 수 있는 사용자명과  패스워드를 지정한다.
HTTP를 통해 서버에 접속하기 위해 SSL 설정을 비활성화 한다.

>Wordpress를 실행하기 위해서 MairaDB는 10.2 버전 이상이 설치되어야한다.

#####  `db-mysql.tf`

>리소스 : azurerm_mariadb_server

```
resource "azurerm_mariadb_server" "wp-mdb-server" {

 name                = var.db-server
 location            = azurerm_resource_group.wp-rg.location
 resource_group_name = azurerm_resource_group.wp-rg.name

 sku_name = "B_Gen5_2"

 storage_mb                   = 51200
 backup_retention_days        = 7
 geo_redundant_backup_enabled = false

 administrator_login          = var.db-user    
 administrator_login_password = var.db-password   
 version                      = "10.2"
 ssl_enforcement_enabled      = false

}
```


##### `variable.tf`

```
variable "db-server"{

 description = "Azure MariaDB Server Name"
 type = string
 default = "wp-mdb-server-hj"

} 

variable "db-user" {

 description = "Azure MariaDB Wordpress User"
 type        = string
 default     = "wordpress"

}

variable "db-password" {

 description = "Azure MariaDB Password"
 type        = string
 default     = "p@ssw0rd"

}
```

<br>

### 2-6-3. MairaDB  데이터베이스 생성

MariaDB 데이터베이스 `wordpress` 를 생성한다. MariaDB의 Charset(기호 및 인코딩 집합)과 Collation(문자열 비교, 정렬 위해 정의 된 규칙)을 설정한다.

##### `db-mysql.tf`

>리소스 : azurerm_mariadb_database

```
resource "azurerm_mariadb_database" "wordpress" {

 name                = "wordpress"
 resource_group_name = azurerm_resource_group.wp-rg.name
 server_name         = azurerm_mariadb_server.wp-mdb-server.name
 charset             = "utf8"
 collation           = "utf8_general_ci"

}
```

<br>

### 2-6-4. DB Firewall Rule 생성

MariaDB 서버의  방화벽을 생성한다.

##### `db-mysql.tf`

>리소스 : azurerm_mariadb_firewall_rule

```
resource "azurerm_mariadb_firewall_rule" "db-fw-rule" {

 name                = "db-fw-rule"
 resource_group_name = azurerm_resource_group.wp-rg.name
 server_name         = azurerm_mariadb_server.wp-mdb-server.name
 start_ip_address    = "0.0.0.0"
 end_ip_address      = "255.255.255.255"

}
```

<br>

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/mariadb.png" width=800/>

<br>
<br>

## 2-7. Load Balancer 구성
---

### 2-7-1. LB Public IP 생성

외부와의 통신을 위해 LB의 Public IP를 정적으로 할당한다.

##### `lb.tf`

>리소스 : azurerm_public_ip

```
resource "azurerm_public_ip" "lb-pubip" {

 name                = "lb-pubip"
 location            = azurerm_resource_group.wp-rg.location
 resource_group_name = azurerm_resource_group.wp-rg.name
 allocation_method   = "Static"
 sku = "Standard"

}
```

<br>

### 2-7-2. LB 생성

Web Server 부하 분산을 위한 LB를 생성한다.
앞서 생성한 LB Public IP를 할당한다

>LB Public IP 생성이 선행 되어야한다.

##### `lb.tf`

>리소스 : azurerm_lb

```
resource "azurerm_lb" "vmss-lb" {
  name                = "vmss-lb"
  location            = azurerm_resource_group.wp-rg.location
  resource_group_name = azurerm_resource_group.wp-rg.name
  
  sku = "Standard"

  #Front
  frontend_ip_configuration {
    name                 = "lb-publicip"
    public_ip_address_id = azurerm_public_ip.lb-pubip.id
    #subnet_id            = azurerm_subnet.lb-subnet.id
  }

  depends_on = [
    azurerm_public_ip.lb-pubip
  ]
}
```

<br>

### 2-7-3. LB Backend Address Pool 생성

LB Backend Address Pool을 생성하여 LB와 연결한다.

>LB 생성이 선행되어야한다

##### `lb.tf`

>리소스 : azurerm_lb_backend_address_pool

```
resource "azurerm_lb_backend_address_pool" "lb-pool" {
  loadbalancer_id = azurerm_lb.vmss-lb.id
  name            = "lb-pool"

  depends_on = [
    azurerm_lb.vmss-lb
  ]
}
```

<br>

### 2-7-4. LB Probe 설정

LB Endpoint 부하 분산을 위해 상태 확인을 위해 Probe를 생성하고 LB와 연결한다.
HTTP 프로토콜을 사용하여 Probe를 진행하기 위해 80번 포트를 지정한다.

> LB 생성이 선행되어야한다.

###### `lb.tf`

>리소스 : azurerm_lb_probe

```
resource "azurerm_lb_probe" "lb-probe" {
  loadbalancer_id = azurerm_lb.vmss-lb.id
  name            = "lb-probe"
  port            = 80

  depends_on = [
    azurerm_lb.vmss-lb
  ]
}
```

<br>

### 2-7-5. LB 규칙 관리

LB 접속 규칙을 관리한다.
외부 Endpoint 포트를 HTTP 포트로 설정하고 LB의 Public IP와 연결한다.
해당 규칙이 적용될 LB Backend Address Pool을 참조하고 상태 확인을 위한 Probe도 연결한다.

> LB와 LB Probe 생성이 선행되어야한다.

##### `lb.tf`

>리소스 : azurerm_lb_rule

```
resource "azurerm_lb_rule" "lb-rule" {
  loadbalancer_id                = azurerm_lb.vmss-lb.id
  name                           = "lb-rule"
  protocol                       = "Tcp"
  frontend_port                  = 80
  backend_port                   = 80
  frontend_ip_configuration_name = "lb-publicip"
  backend_address_pool_ids       = [azurerm_lb_backend_address_pool.lb-pool.id]
  probe_id                       = azurerm_lb_probe.lb-probe.id
  depends_on = [
    azurerm_lb.vmss-lb,
    azurerm_lb_probe.lb-probe
  ]
}
```

<br>

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%84%9C.png" width=800/>

<br> 
<br> 

## 2-8. Output
---
#####  `output.tf`

`output`은 terraform으로 생성시에 정의한 내용들을 출력해준다.

- `bastion_host - public_ip`
- `lb - frontend_ip`

접속 테스트를 쉽게 하기위해 위 내용을 작성한다.

```
output "bastion-publicip" {
  value = azurerm_linux_virtual_machine.bastion-vm.public_ip_address
}

output "lb-frontend_ip" {
  value = azurerm_public_ip.lb-pubip.ip_address
}
```

<br>
<br>

---

## 3. 결과
<br>
### 3-1. 실행 화면

<img style="border-radius: 7px;-moz-border-radius: 7px;-khtml-border-radius: 7px;-webkit-border-radius: 7px;" src="https://raw.githubusercontent.com/na3150/typora-img/main/img/image-20220509084126401.png" width=800/>

<br>
### 3-2. AS-IS, TO-BE

#### AS-IS

- Azure Cloud 활용 가능 (포털 사용 및 리소스 코드화)
- Bastion Host 접속을 통한 Web Server 접속을 구현하여 보안 강화
- Web Server의 VMSS 구현으로 가용성 및 안정성 확보
- Load Balancer를 이용하여 Web 부하 분산
- Ansible Playbook을 이용하여 Web Server 구성 관리 자동화
- Terraform을 이용하여 Wordpress 배포 자동화

  
#### TO-BE

- Bastion Host Server의 VMSS 구현으로 단일 장애점 해결
- Web Server 보안그룹의 인바운드 SSH 접속 IP를 Bastion Host 보안 그룹으로 제한하여 보안 강화
- Ansible Playbook 변수 파일 Vault 암호화를 통한 보안 강화
- 추가 변수 처리를 통한 실행 파일 재사용성 확보
