# Configuração de Rede WAN

Projeto da unidade curricular de Redes de Longa Distância (WAN), CTeSP em Redes e Sistemas Informáticos - ESTIG, Instituto Politécnico de Beja.

**Autor:** Manuel Engrola (nº 24154)
**Docente:** Prof. Armando Ventura
**Data:** 15 de janeiro de 2024

---

## Descrição do projeto

O objetivo deste trabalho foi montar e configurar uma topologia WAN heterogénea, combinando routers MikroTik e Cisco no GNS3, com ligação a um cliente Windows. A ideia era simular um cenário próximo do real: dois equipamentos de fabricantes diferentes a comunicar entre si, com routing dinâmico (OSPF), NAT para acesso à internet, DHCP, acesso remoto por Telnet e ligações VPN (PPTP e L2TP/IPsec).

## O que foi implementado

- Topologia com router MikroTik (RouterOS CHR 6.49.10) e router Cisco (C2691)
- Routing dinâmico com OSPF entre os dois routers
- NAT/masquerade para acesso à internet a partir da rede do cliente
- Servidor DHCP no router Cisco para atribuição automática de IPs
- Acesso remoto via Telnet em ambos os routers
- VPN PPTP e L2TP/IPsec configuradas no MikroTik, com 3 utilizadores cada
- Interfaces loopback nos dois routers para identificação
- Backup das configurações de ambos os equipamentos

## Topologia

```
                    INTERNET
                        |
                  MikroTik R1
              (RouterOS CHR 6.49.10)
                        |
                Rede A: 14.20.20.0/30
                        |
                   Cisco R2
                  (C2691)
                        |
                Rede B: 45.14.45.0/24
                        |
                Cliente Windows
                  45.14.45.2/24
```

## Esquema de endereçamento

O endereçamento foi calculado a partir do número de aluno (24154), usando a fórmula fornecida pelo docente:

```
F = 24154 mod 75 + 10 = 14
```

| Rede | Endereço | Observações |
|---|---|---|
| Rede A (ponto-a-ponto) | 14.20.20.0/30 | Liga R2 (.1) a R1 (.2) |
| Rede B (LAN cliente) | 45.14.45.0/24 | Gateway .1, cliente .2 via DHCP |
| Loopback R1 (MikroTik) | 35.35.35.1/24 | Identificação do router |
| Loopback R2 (Cisco) | 35.35.35.2/24 | Identificação do router |

## Equipamentos e função de cada um

| Equipamento | Modelo | Interfaces | Função |
|---|---|---|---|
| R1 | MikroTik RouterOS CHR 6.49.10 | ether1 (internet), ether2 (Rede A) | Gateway para a internet, peer OSPF |
| R2 | Cisco C2691 | Fa0/0 (Rede A), Fa0/1 (Rede B) | Peer OSPF, servidor DHCP, servidor VPN |
| Cliente | Windows 10 64-bit | Adaptador único | Máquina de teste |

---

## Configuração

### 1. Preparação do ambiente

- GNS3 (versão 2.x ou superior)
- VirtualBox (ou VMware)
- Imagem RouterOS CHR 6.49.10 para o MikroTik
- Imagem IOS fornecida para o Cisco C2691
- VM Windows 10 64-bit

No GNS3: adicionar o router Cisco com a IOS fornecida, criar a VM do MikroTik no VirtualBox com dois adaptadores de rede (um em modo bridge para internet, outro ligado à cloud do GNS3), e importar a VM Windows ligada à topologia.

### 2. MikroTik (R1)

Configuração de endereços e NAT:

```
/ip dhcp-client add interface=ether1 disabled=no
/ip address add address=14.20.20.2/30 interface=ether2
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

OSPF:

```
/routing ospf instance add name=ospf1 router-id=1.1.1.1
/routing ospf area add name=ospf1 area-id=0.0.0.0 instance=ospf1
/routing ospf network add network=14.20.20.0/30 area=ospf1
/routing ospf network add network=35.35.35.0/24 area=ospf1
```

Loopback:

```
/interface bridge add name=loopback
/ip address add address=35.35.35.1/24 interface=loopback
```

Telnet:

```
/ip service set telnet disabled=no
/ip service set telnet address=0.0.0.0/0
```

### 3. Cisco (R2)

Interfaces:

```
interface FastEthernet0/0
 ip address 14.20.20.1 255.255.255.252
 no shutdown
!
interface FastEthernet0/1
 ip address 45.14.45.1 255.255.255.0
 no shutdown
```

NAT:

```
interface FastEthernet0/0
 ip nat outside
!
interface FastEthernet0/1
 ip nat inside
!
access-list 1 permit 45.14.45.0 0.0.0.255
ip nat inside source list 1 interface FastEthernet0/0 overload
ip route 0.0.0.0 0.0.0.0 14.20.20.2
```

OSPF:

```
router ospf 1
 router-id 2.2.2.2
 network 14.20.20.0 0.0.0.3 area 0
 network 45.14.45.0 0.0.0.255 area 0
 network 35.35.35.0 0.0.0.255 area 0
```

Loopback:

```
interface loopback0
 ip address 35.35.35.2 255.255.255.0
```

Telnet:

```
line vty 0 4
 password <sua_password>
 login
```

DHCP:

```
ip dhcp excluded-address 45.14.45.1
ip dhcp pool redeb
 network 45.14.45.0 255.255.255.0
 default-router 45.14.45.1
 dns-server 8.8.8.8
```

### 4. Cliente Windows

Configurar o adaptador de rede para obter IP e DNS automaticamente (Painel de Controlo > Central de Rede e Partilha > Alterar definições do adaptador > Propriedades do IPv4).

Testes de conectividade:

```
ipconfig
ping 45.14.45.1      (gateway)
ping 14.20.20.1      (Cisco)
ping 14.20.20.2      (MikroTik)
ping 8.8.8.8         (internet)
ping google.com      (DNS)
```

Ativar cliente Telnet (PowerShell como administrador):

```
dism /online /Enable-Feature /FeatureName:TelnetClient
```

### 5. VPN

L2TP/IPsec no MikroTik:

```
/interface l2tp-server server set enabled=yes ipsec-secret=<seu_segredo_ipsec>
/ppp secret add name=user1 password=<password> service=l2tp
/ppp secret add name=user2 password=<password> service=l2tp
/ppp secret add name=user3 password=<password> service=l2tp
```

PPTP no MikroTik:

```
/interface pptp-server server set enabled=yes
/ppp secret add name=pptp1 password=<password> service=pptp
/ppp secret add name=pptp2 password=<password> service=pptp
/ppp secret add name=pptp3 password=<password> service=pptp
```

No cliente Windows, criar a ligação VPN em Definições > Rede e Internet > VPN, indicando o servidor 14.20.20.2, o tipo de VPN (L2TP/IPsec com chave pré-partilhada, ou PPTP) e as credenciais dos utilizadores criados.

### 6. Backup das configurações

MikroTik:

```
/system backup save name=engrola
/export file=engrola-config
```

Cisco (via TFTP para o cliente Windows):

```
copy running-config tftp:
Address or name of remote host []? 45.14.45.2
Destination filename [router-confg]? cisco-backup.txt
```

---

## Testes realizados

A partir do cliente Windows, foi confirmada a conectividade a todos os pontos da rede (gateway, ambos os routers, loopbacks e internet), a resolução de DNS, o funcionamento do OSPF (verificado com `/routing ospf neighbor print` no MikroTik e `show ip ospf neighbor` no Cisco), e o estabelecimento das ligações VPN com atribuição de IP e tráfego a passar pelo túnel.

## Notas de segurança

O Telnet é um protocolo sem cifragem e foi usado apenas por ser um ambiente de laboratório; num cenário real seria substituído por SSH. Entre as duas VPNs configuradas, o L2TP/IPsec é preferível ao PPTP para produção. Em ambiente real seria também de considerar autenticação MD5 no OSPF.

## Estrutura do repositório

```
WAN-Network-Configuration/
├── README.md
├── configs/
│   ├── mikrotik-backup.backup
│   ├── cisco-running-config.txt
│   ├── ospf-mikrotik.txt
│   └── ospf-cisco.txt
└── docs/
    ├── network-diagram.png
    └── relatorio-wan.md   (relatório completo do trabalho)
```

O relatório completo com o passo-a-passo detalhado está em `docs/relatorio-wan.md`. O PDF original entregue na unidade curricular não faz parte deste repositório, por conter capturas de ecrã com passwords e segredos usados no laboratório.
