# Projeto-SiteDR

# Implementação de Disaster Recovery (DR) com Oracle Cloud Infrastructure (OCI)


## 1. O Desafio: Continuidade de Negócios para uma Grande Universidade

Este projeto foi iniciado para endereçar uma necessidade crítica de negócios: garantir a **continuidade dos serviços essenciais** de uma grande universidade privada brasileira em caso de desastre no datacenter on-premises.

**Estado Inicial (Antes do Projeto):**

- **Infraestrutura 100% On-Premises:** Todos os serviços críticos (Active Directory, Bancos de Dados Oracle/PostgreSQL/MySQL/MongoDB, plataformas de ensino, portais web, ERP acadêmico, cluster Openshift e File Servers) rodavam em servidores **VMware vCenter** locais.
- **Ponto Único de Falha:** Um incidente grave no datacenter principal resultaria em **paralisação total das operações** acadêmicas e administrativas por tempo indeterminado.
- **RTO/RPO indefinidos:** Não havia métricas claras de recuperação, significando potencial perda permanente de dados e dias ou semanas de downtime.

O objetivo era claro: **projetar e implementar uma solução de DR robusta na OCI**, garantindo alta disponibilidade e capacidade de failover para mais de 40 servidores críticos mapeados, além dos bancos de dados Oracle com Data Guard e réplicas PostgreSQL e MySQL.

---

## 2. A Arquitetura da Solução Híbrida (Hub-Spoke)

A solução implementada foi uma **arquitetura Hub-Spoke** em uma região OCI no Brasil, com conectividade híbrida via **VPN Site-to-Site** entre o datacenter on-premises e a Oracle Cloud, utilizando um firewall **FortiGate** como ponto central de controle.

### Diagrama da Arquitetura

**Componentes Principais da Arquitetura:**

1. **Ambiente On-Premises (Datacenter Principal):**
   - Servidores host **VMware vCenter 8.x** com múltiplos Storage Pods e 3 clusters de virtualização.
   - Mais de 40 VMs críticas mapeadas em 11 waves de replicação, cobrindo serviços de diretório, plataformas de ensino, ERP, portais web, object storage, bancos de dados e aplicações diversas.
   - 3 Ambientes de Domain Controller com sincronia Ativo -> Passivo.
   - 4 bancos Oracle em produção (RAC e Single Instance) com versões 12.2 e 19.x.
   - Firewall on-premises com VPN IPSec para a OCI.

2. **Oracle Cloud Infrastructure (Site de DR):**
   - **DRG (Dynamic Routing Gateway):** Ponto central de roteamento interconectando VCNs e VPN on-premises. O gateway padrão de todas as VCNs aponta para a instância de firewall FortiGate na nuvem.
   - **FortiGate VM:** Instância FortiGate-VM 7.6.x PAYG do marketplace OCI, controlando todos os acessos Internet e comunicação VPN com o ambiente on-premises.
   - **Rackware RMM:** Servidor de replicação Oracle Linux 8 com pool de armazenamento ZFS de 30 TB, 60 licenças de replicação contratadas, com renovação anual.
   - **NFS Server:** Instância Oracle Linux 9 com sincronismo rsync a cada 2 horas para dados de plataformas de ensino (~8 TB), portais web (~150 GB) e volumes persistentes OpenShift (~1 TB).
   - **HAProxy Load Balancer:** Instância Oracle Linux 9 com HAProxy, publicada com IP público, transcrevendo regras do balanceador on-premises para o ambiente de containers na OCI. Backup diário com retenção de 7 dias.
   - **Cluster Openshift:** Implantado via Terraform Stack com 3 nós Control Plane (3 OCPUs, 16 GB cada) e 3 nós Compute (8 OCPUs, 64 GB cada), em VCN dedicada.
   - **Bancos de Dados Standby:** 4 Oracle Data Guard (MaxPerformance), 2 PostgreSQL streaming replication (read-only), 1 MySQL replicação assíncrona GTID e 1 MongoDB com ReplicaSet.

3. **Prestador de Serviços (Monitoramento):**
   - VPN Site-to-Site dedicada entre OCI e o NOC do prestador para comunicação com Zabbix Proxy.

---

## 3. Organização Lógica de Compartimentos

O tenant OCI foi estruturado em compartimentos para melhor segmentação e governança:

- **NetworkInfra:** VCNs, Subnets, DNS, Tabelas de rotas, Listas de segurança, instância de firewall, Zabbix Proxy, NLB de gerenciamento e DRG.
- **Applications:** Servidores AD, Rackware, NFS e sub-compartimentos por aplicação (11 sub-compartimentos nomeados por sistema — plataforma de ensino, portal web, ERP, object storage, etc.).
- **Databases:** Servidores IaaS e PaaS de bancos Oracle, PostgreSQL, MongoDB e MySQL.
- **Containers:** Cluster Openshift, Load Balancers (OCI nativos e HAProxy) e Terraform Stacks.

---

## 4. Stack de Tecnologias Utilizadas

- **Plataforma de Nuvem (DR):**
  - Oracle Cloud Infrastructure (OCI) — Região Brasil
  - Compute (VM.Standard.E3.Flex / E4.Flex / E5.Flex)
  - Block Storage (Block Volumes, Boot Volumes)
  - Networking (VCN, DRG, VPN Site-to-Site, Load Balancers, DNS Zones)
  - DB Systems (Oracle Base Database Service)

- **Replicação e DR:**
  - Rackware RMM v7.4.x (replicação de VMs — VMware to OCI)
  - Oracle Data Guard (replicação de bancos Oracle — MaxPerformance)
  - PostgreSQL Streaming Replication (pg_basebackup + WAL shipping)
  - MySQL GTID Async Replication (via MySQL Shell)
  - MongoDB (replicação via ReplicaSet)
  - rsync (sincronismo de volumes NFS a cada 2 horas)

- **Virtualização On-Premises:**
  - VMware vCenter Server 8.x

- **Sistemas e Serviços Replicados:**
  - Oracle Linux 8 e 9
  - Red Hat OpenShift (implantado via Terraform)
  - Windows Server 2016, 2022 com Active Directory
  - Oracle Database 12.2 e 19.x (RAC e Single)
  - PostgreSQL 15.x e 16.x
  - MySQL 8.x (ReplicaSet com MySQL Shell)
  - MongoDB, Object Storage distribuído
  - HAProxy
  - Plataforma LMS, Portal Web, ERP Acadêmico, ERP Financeiro, Sistemas de Gestão, Formulários Web e diversas aplicações corporativas

- **Networking e Segurança:**
  - VPN Site-to-Site (IPSec — Hub-Spoke)
  - FortiGate-VM 7.6.x PAYG (Firewall central na OCI)
  - Network Load Balancer (NLB) para gerenciamento externo
  - OCI Load Balancers para cluster de containers
  - DNS Privado com Private Resolver e regras de forward

---

## 5. Metodologia de Implementação (Passo a Passo)

O projeto foi executado em 5 fases principais para garantir uma implementação segura e validada.

### Fase 1: Assessment e Design

1. **Mapeamento de VMs Críticas:** Identificação de mais de 40 servidores essenciais organizados em 11 waves de replicação, agrupadas por sistema/aplicação.
2. **Definição do Modelo de Replicação:** Classificação em **STAGE-1** (Rackware armazena imagem, instância criada sob demanda em minutos a ~1 hora) e **STAGE-1+2** recomendado para workloads de containers e object storage (servidor destino sempre ativo via Rackware Micro Kernel, para cargas com tempo de recuperação superior a 4 horas e grande volume de dados).
3. **Design de Rede:** Arquitetura Hub-Spoke com 3 VCNs (HUB, Produção e Containers) interconectadas via DRG com firewall FortiGate como gateway padrão de todo o ambiente.

### Fase 2: Configuração da Infraestrutura de Rede

1. **Provisionamento das VCNs:** Criação das redes HUB (WAN + LAN), Produção (sub-rede pública + privada) e Containers (sub-redes para containers, bare metal e pública).
2. **Configuração do DRG:** Deploy do Dynamic Routing Gateway com attachments para as VCNs e route table com default gateway apontando para o firewall.
3. **Deploy do Firewall:** Instância FortiGate do marketplace OCI com interface WAN (IP público) e LAN (IP privado), atuando como gateway central de Internet e VPN.
4. **Configuração de VPN Site-to-Site:** Túnel IPSec entre o datacenter principal e a OCI (túnel primário ativo + túnel redundante), e VPN secundária do prestador de serviços para monitoramento.
5. **Configuração de DNS:** Zonas privadas para resolução interna com Private Resolver e regras de forward para Domain Controllers on-premises.

### Fase 3: Configuração do Rackware (Replicação de VMs)

1. **Deploy do Servidor Rackware:** Instância Oracle Linux 8 com 16 OCPUs, 128 GB RAM, pool ZFS de 30 TB (15 discos de 2 TB em read/write).
2. **Configuração do Clouduser:** Integração com OCI via API key para provisionamento automático de instâncias durante o failover.
3. **Integração com vCenter:** Conexão com VMware vCenter para descoberta automática das VMs de produção.
4. **Criação das Waves:** 11 waves configuradas com policies de replicação individuais, cada uma com 1 ponto de retenção (imagem sobrescrita a cada sincronização).
5. **Habilitação da Replicação:** Início da replicação de todos os servidores críticos, com scripts de auto-start-shutdown para servidores de object storage (otimização de custos de CPU/RAM na nuvem).

### Fase 4: Configuração dos Bancos de Dados (Data Guard e Réplicas)

1. **Oracle Data Guard:** Configuração de 4 Data Guards em modo MaxPerformance (Fast-Start Failover desabilitado), contemplando bancos de diferentes versões (12.2 e 19.x), arquiteturas (CDB e NoCDB) e tipos de armazenamento (ASM e File System). Um dos bancos foi provisionado como DB Systems (PaaS gerenciado pela OCI), enquanto os demais utilizam VMs IaaS com a mesma versão de SO e arquitetura de storage do ambiente de origem.
2. **PostgreSQL Streaming Replication:** Cópia física via `pg_basebackup` seguida de habilitação de streaming replication em modo read-only, com usuário dedicado de replicação e autenticação md5/scram-sha-256.
3. **MySQL GTID Replication:** Configuração de ReplicaSet com replicação assíncrona GTID via MySQL Shell, garantindo que o standby trabalhe em modo R/O com SSL habilitado. O uso de GTID permite identificar automaticamente o ponto de interrupção no binlog e retomar a replicação de forma íntegra.

### Fase 5: Configuração do Cluster de Containers e Load Balancers

1. **Deploy via Terraform:** 2 Stacks no OCI Resource Manager provisionando 3 nós Control Plane e 3 nós Compute para o cluster de containers.
2. **Deploy do HAProxy:** Instância Oracle Linux 9 com HAProxy 2.8, transcrição das regras do balanceador on-premises (originalmente Citrix Netscaler) para frontends/backends do cluster, com certificado SSL wildcard e páginas de erro personalizadas. Acesso externo restrito às portas TCP 80 e 443.
3. **Configuração de Backup:** Política de backup incremental diário com retenção de 7 dias para o boot volume do load balancer, dada sua criticidade operacional.

---

## 6. Desafios Chave e Lições Aprendidas

### 1. Limitações do Load Balancer Nativo da OCI

- **Desafio:** Durante a fase de POC, foram identificadas restrições técnicas do Load Balancer nativo da OCI para atender todas as regras de roteamento necessárias ao funcionamento das aplicações, originalmente configuradas em um balanceador Citrix Netscaler com regras complexas de URL rewriting e backend routing.
- **Lição:** O HAProxy demonstrou-se uma solução madura e flexível, atendendo todos os requisitos técnicos sem limitações. A decisão de usar uma instância Linux com HAProxy ao invés do serviço gerenciado foi fundamental para o sucesso do projeto. A dashboard de estatísticas do HAProxy também se mostrou valiosa para troubleshooting.

### 2. Capacidade de Armazenamento do Rackware

- **Desafio:** O zpool de 30 TB atingiu mais de 90% de ocupação, impossibilitando a ativação de uma política com 2 pontos de retenção. O sistema exibe alertas de espaço na interface de gerenciamento.
- **Lição:** O dimensionamento de armazenamento para DR deve considerar crescimento futuro e margem operacional. A política de 1 ponto de retenção (imagem sobrescrita a cada sync) é um trade-off entre custo e resiliência que precisa ser reavaliado periodicamente. É recomendável planejar expansão de discos ou limpeza proativa.

### 3. Licenças de Replicação Consumidas por VMs Desativadas

- **Desafio:** Quinze servidores foram replicados e posteriormente desativados durante a fase de implantação, consumindo licenças desnecessariamente.
- **Lição:** É possível recuperar essas licenças via chamado técnico junto ao fornecedor. O processo de descomissionamento de VMs deve incluir a liberação da licença de replicação como etapa obrigatória no runbook operacional.

### 4. Comunicação Oracle Data Guard com Restrições de Hostname

- **Desafio:** Para um dos bancos de dados associados a um ERP, foi necessário manter o mesmo hostname na máquina standby (cópia da primária), forçando toda a comunicação Data Guard a ser feita exclusivamente via IP ao invés de resolução por nome.
- **Lição:** Requisitos de aplicação podem impor restrições inesperadas na arquitetura de DR. A comunicação por IP é funcional, mas exige documentação rigorosa e atenção redobrada na manutenção. Esse tipo de requisito deve ser identificado na fase de assessment.

### 5. Scripts de Automação sem Validação com o Cliente

- **Desafio:** Scripts de auto-start-shutdown foram criados para servidores de object storage com o objetivo de otimizar custos de nuvem (manter VMs ligadas apenas durante a janela de cópia). Porém, a execução desses scripts não é visível na interface web da ferramenta de replicação, e não houve validação conjunta com a equipe técnica do cliente.
- **Lição:** Automações críticas devem ser validadas em conjunto com o cliente e ter visibilidade operacional clara. Os scripts foram mantidos disponíveis porém desativados até validação formal.

---

## 7. Recomendações Operacionais

Para garantir que o ambiente DR permaneça saudável e pronto para ativação, as seguintes atividades de monitoramento contínuo são recomendadas:

- **Monitoramento de conectividade:** Links de Internet e VPN responsáveis pela replicação (Rackware, NFS, bancos de dados e Active Directory).
- **Acompanhamento da replicação de VMs:** Dashboards e alertas por e-mail para status das waves no Rackware.
- **Verificação de espaço em disco:** Servidores de replicação (zpool), NFS e bancos de dados — alertas proativos antes de atingir limites críticos.
- **Renovação de certificados SSL:** Certificados wildcard utilizados nos load balancers.
- **Verificação periódica da replicação de bancos:** Status dos Data Guards Oracle, PostgreSQL streaming replication e MySQL GTID replication.
- **Validação de registros DNS:** Sincronismo entre o balanceador on-premises e os registros DNS na OCI.
- **Atualização do cluster de containers:** Manter versionamento alinhado com o ambiente on-premises.
- **Testes de DR regulares:** Execução periódica de testes de failover para validar o ambiente, atualizar documentações, preencher cadernos de teste e treinar equipes técnicas envolvidas.

---


## 8. Impacto e Conclusão

**Resultado:** A instituição saiu de um estado de **risco total de continuidade** para uma arquitetura de DR robusta na Oracle Cloud, com replicação ativa de mais de 40 VMs via Rackware, 4 bancos Oracle protegidos por Data Guard, réplicas PostgreSQL e MySQL em streaming, sincronismo automatizado de volumes NFS e cluster Openshift pronto para failover.

A implementação foi concluída com sucesso, garantindo:

- **Alta disponibilidade** dos serviços críticos acadêmicos e administrativos.
- **Replicação segura** de dados com múltiplas tecnologias complementares (Rackware, Data Guard, Streaming Replication, rsync).
- **Conectividade híbrida** entre OCI e datacenter via VPN IPSec com arquitetura Hub-Spoke.
- **Capacidade de failover** testada e documentada em caso de desastre.

Este projeto demonstra a viabilidade de uma estratégia híbrida Hub-Spoke (On-Premises + OCI) combinada com múltiplas tecnologias de replicação para criar uma solução de continuidade de negócios completa e resiliente, adequada a ambientes universitários de grande porte com alta diversidade tecnológica.

---

### Segmento

- **Educação Superior — Universidade Privada de Grande Porte (Brasil)**

### Tecnologias-Chave

`Oracle Cloud Infrastructure` · `Oracle Data Guard` · `Rackware RMM` · `FortiGate VM` · `Red Hat OpenShift` · `HAProxy` · `PostgreSQL Streaming Replication` · `MySQL GTID` · `VMware vCenter` · `VPN Site-to-Site` · `Terraform`
