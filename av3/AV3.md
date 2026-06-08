# Relatório AV3

Pedro H. Gaya - 2224655

---

## Resumo

Para este projeto, foram estabelecidos dois domínios - OSPF e BGP. Ambos os domínios compartilham duas VLANs: 10 e 20. Em cada domínio foram criadas subnets, com um roteador por subnet. Toda subnet possui dois PCs, um por VLAN.

- OSPF:
  - A: Switch0, R1, PC0, PC7
  - B: Switch1, R2, PC1, PC6
- BGP:
  - C: Switch2, R3 (AS100), PC2, PC11, DHCP
  - D: Switch3, R4 (AS300), PC3, PC8
- Distribuidores:
  - OSPF/BGP: R_BORDER (AS200)

![Topologia](topologia.jpeg)

## Cálculos de Sub-redes

### Máscara /26 - LAN

Cada rede /24 foi dividida em sub-redes /26, fornecendo 62 hosts utilizáveis por sub-rede.

- Bits de host: 6 -> 2^6 - 2 = **62 hosts**
- Máscara: 255.255.255.192
- Incremento de bloco: 64

| Sub-rede         | Endereço de Rede | Primeiro Host | Último Host | Broadcast |
| ---------------- | ---------------- | ------------- | ----------- | --------- |
| 1ª /26 (VLAN 10) | x.x.x.0          | x.x.x.1       | x.x.x.62    | x.x.x.63  |
| 2ª /26 (VLAN 20) | x.x.x.64         | x.x.x.65      | x.x.x.126   | x.x.x.127 |

### Máscara /30 - WAN

Usada nos links ponto a ponto entre roteadores, fornecendo exatamente 2 hosts utilizáveis.

- Bits de host: 2 -> 2^2 - 2 = **2 hosts**
- Máscara: 255.255.255.252
- Incremento de bloco: 4

---

## Plano de Endereçamento IP

### Redes WAN

| Roteador A | IP Lado A | Roteador B | IP Lado B | Rede        |
| ---------- | --------- | ---------- | --------- | ----------- |
| R1         | 10.0.1.1  | R2         | 10.0.1.2  | 10.0.1.0/30 |
| R1         | 10.0.2.1  | R_BORDER   | 10.0.2.2  | 10.0.2.0/30 |
| R_BORDER   | 10.0.3.1  | R3         | 10.0.3.2  | 10.0.3.0/30 |
| R3         | 10.0.4.1  | R4         | 10.0.4.2  | 10.0.4.0/30 |

### Redes LAN

| Site         | Roteador | VLAN | Rede         | Máscara | Gateway      |
| ------------ | -------- | ---- | ------------ | ------- | ------------ |
| A            | R1       | 10   | 192.168.1.0  | /26     | 192.168.1.1  |
| A            | R1       | 20   | 192.168.1.64 | /26     | 192.168.1.65 |
| B            | R2       | 10   | 192.168.2.0  | /26     | 192.168.2.1  |
| B            | R2       | 20   | 192.168.2.64 | /26     | 192.168.2.65 |
| C            | R3       | 10   | 192.168.3.0  | /26     | 192.168.3.1  |
| C            | R3       | 20   | 192.168.3.64 | /26     | 192.168.3.65 |
| D            | R4       | 10   | 192.168.4.0  | /26     | 192.168.4.1  |
| D            | R4       | 20   | 192.168.4.64 | /26     | 192.168.4.65 |
| C (Servidor) | R3       | 10   | 192.168.3.10 | /26     | 192.168.3.1  |

---

## Implementação

### Switches

Configuração idêntica em SW1, SW2, SW3 e SW4:

- Criação das VLANs 10 (Dados) e 20 (Gestão)
- Portas de acesso associadas a cada VLAN para os hosts
- Porta trunk para o roteador carregando ambas as VLANs
- SW3 recebe configuração adicional na porta do SRV-DHCP como acesso VLAN 10

### Roteadores

Duas subinterfaces são criadas, uma por VLAN. Cada link recebe um endereço /30 em cada extremidade.

O OSPF configurado em R1, R2 e R_BORDER, todos na área 0. Cada roteador anuncia suas redes com wildcards correspondentes às máscaras utilizadas.

O BGP configurado com sessões eBGP entre três sistemas autônomos distintos:

- **eBGP** entre R_BORDER (AS 200) e R3 (AS 100)
- **eBGP** entre R3 (AS 100) e R4 (AS 300)

Cada roteador BGP anuncia suas redes com o comando `network` e redistribui as interfaces conectadas.

R_BORDER é o ponto de integração entre os dois domínios:

- OSPF redistribui as rotas aprendidas via BGP para dentro do domínio OSPF
- BGP redistribui as rotas aprendidas via OSPF para dentro do domínio BGP

### DHCP

R1 serve DHCP diretamente para as VLANs 10 e 20 do Site A (rede própria) e também para as VLANs 10 e 20 do Site B (rede distinta). R2 utiliza `ip helper-address` apontando para R1 para encaminhar os pedidos do Site B.

SRV-DHCP (192.168.3.10) centraliza a distribuição de endereços para o Site C e o Site D. R3 utiliza relay na VLAN 20 (pois a VLAN 10 está na mesma sub-rede do servidor). R4 utiliza relay em ambas as VLANs.

---

## Comandos Utilizados

### VLANs e Trunk (Switch)

| Comando                               | Função                                       |
| ------------------------------------- | -------------------------------------------- |
| `vlan 10`                             | Cria a VLAN 10                               |
| `switchport mode access`              | Define a porta como acesso                   |
| `switchport access vlan 10`           | Associa a porta à VLAN 10                    |
| `switchport mode trunk`               | Define a porta como trunk                    |
| `switchport trunk allowed vlan 10,20` | Permite apenas as VLANs necessárias no trunk |

O mesmo foi feito para vlan 20.

### Subinterfaces

| Comando                                  | Função                                                   |
| ---------------------------------------- | -------------------------------------------------------- |
| `interface Fa0/0.10`                     | Cria subinterface para VLAN 10                           |
| `encapsulation dot1Q 10`                 | Define encapsulamento 802.1Q com VLAN 10                 |
| `ip address 192.168.1.1 255.255.255.192` | Atribui IP à subinterface, que atua como gateway da VLAN |

### OSPF

| Comando                               | Função                                       |
| ------------------------------------- | -------------------------------------------- |
| `router ospf 1`                       | Ativa o processo OSPF com identificador 1    |
| `network 192.168.1.0 0.0.0.63 area 0` | Anuncia a sub-rede na área 0 usando wildcard |
| `redistribute bgp 200 subnets`        | Adiciona rotas BGP no domínio OSPF           |

### BGP

| Comando                                    | Função                                        |
| ------------------------------------------ | --------------------------------------------- |
| `router bgp 100`                           | Ativa o processo BGP no AS 100                |
| `neighbor 10.0.3.1 remote-as 200`          | Define um peer eBGP no AS 200                 |
| `neighbor 10.0.4.2 remote-as 300`          | Define um peer eBGP no AS 300                 |
| `network 192.168.3.0 mask 255.255.255.192` | Anuncia a sub-rede no BGP                     |
| `redistribute connected`                   | Anuncia todas as redes diretamente conectadas |
| `redistribute ospf 1`                      | Adiciona rotas OSPF no BGP                    |

### DHCP

| Comando                               | Função                                                    |
| ------------------------------------- | --------------------------------------------------------- |
| `ip dhcp excluded-address`            | Reserva IPs que não serão distribuídos automaticamente    |
| `ip dhcp pool NOME`                   | Cria um pool DHCP com o nome especificado                 |
| `network 192.168.1.0 255.255.255.192` | Define a rede atendida pelo pool                          |
| `default-router 192.168.1.1`          | Define o gateway entregue aos clientes                    |
| `dns-server 8.8.8.8`                  | Define o servidor DNS entregue aos clientes               |
| `lease 1`                             | Define o tempo de concessão em dias                       |
| `ip helper-address 192.168.3.10`      | Encaminha broadcasts DHCP ao servidor remoto especificado |

## Bibliografia

Uma demonstração do funcionamento da rede pode ser encontrada [aqui](https://youtu.be/98CfqgySZlA).
Todos os arquivos utilizados nesse projeto estão disponíveis no [GitHub](https://github.com/PedroGaya/av2-redes/tree/main/av3).
