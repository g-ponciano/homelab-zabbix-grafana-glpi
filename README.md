# 📡 Homelab: Central de Monitoramento e Suporte de TI com Zabbix, Grafana e GLPI

![Ubuntu](https://img.shields.io/badge/Ubuntu%20Server-26.04.1%20LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Zabbix](https://img.shields.io/badge/Zabbix-Monitoramento-D40000?style=for-the-badge&logo=zabbix&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-13.2.3-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![GLPI](https://img.shields.io/badge/GLPI-11-33B5E5?style=for-the-badge)
![MariaDB](https://img.shields.io/badge/MariaDB-11.8-003545?style=for-the-badge&logo=mariadb&logoColor=white)

---

## 📌 Visão Geral do Projeto

Este é o **Projeto 2** do meu portfólio de infraestrutura e suporte de TI, e complementa o [Projeto 1 (Active Directory, DNS, DHCP, GPO e NTFS)](https://github.com/g-ponciano](https://github.com/g-ponciano/active-directory-homelab))

Enquanto o primeiro projeto mostra que sei **montar** uma infraestrutura, este mostra que sei **monitorá-la, identificar problemas e tratá-los em um fluxo de chamados**: o Zabbix detecta, o Grafana mostra, o técnico investiga, o GLPI registra e o incidente é encerrado com documentação.

Tudo roda em **uma única VM Ubuntu** dentro de um computador com 16 GB de RAM, dividindo recursos com as VMs do Projeto 1.

---

## 📐 Arquitetura

```text
                SEU PC (16 GB RAM) — Oracle VirtualBox
                                │
                  ┌─────────────┴──────────────┐
                  │   SRV-MONITORAMENTO        │
                  │   Ubuntu Server 26.04.1    │
                  │   6 GB RAM · 4 vCPU · 60 GB│
                  │   IP 192.168.100.86        │
                  └─────────────┬──────────────┘
                                │
        ┌───────────────┬───────┴───────┬────────────────┐
        │               │               │                │
     Zabbix          Grafana          GLPI 11        MariaDB
   (detecta e     (dashboards)     (chamados,     (bancos zabbix
    alerta)                         ativos, SLA)     e glpi)
        │               │               │                │
        └───────────────┴───────┬───────┴────────────────┘
                                │
                         Apache + PHP
```

| Serviço | Endereço |
| :--- | :--- |
| Zabbix | `http://192.168.100.86/zabbix/` |
| Grafana | `http://192.168.100.86:3000` |
| GLPI | `http://192.168.100.86/glpi/` |

---

## ⚙️ Ambiente

| Componente | Versão / Detalhe |
| :--- | :--- |
| Virtualização | Oracle VirtualBox |
| Sistema operacional | Ubuntu Server 26.04.1 LTS (`resolute`) |
| Banco de dados | MariaDB 11.8.6 |
| Servidor web / PHP | Apache 2 / PHP 8.5.4 |
| Zabbix | 8.0.0beta2 (pré-lançamento) |
| Grafana | 13.2.3 |
| Plugin Zabbix para Grafana | `alexanderzobnin-zabbix-datasource` 6.9.1 |
| GLPI | 11 (em `/var/www/glpi`) |

---

## 🗺️ Fases do Projeto

| Fase | Tema | O que foi feito | Documentação |
| :---: | :--- | :--- | :--- |
| **1** | Ubuntu | VM, SSH, atualização, LVM, ferramentas de diagnóstico | [📄 fase-01-ubuntu.md](docs/fase-01-ubuntu.md) |
| **2** | Banco de dados | MariaDB, bancos `zabbix` e `glpi`, usuários próprios, hardening básico | [📄 fase-02-banco-de-dados.md](docs/fase-02-banco-de-dados.md) |
| **3** | Zabbix | Repositório oficial, Server, Agent 2, frontend, host monitorado (CPU, RAM, disco, rede, serviços), triggers | *a documentar* |
| **4** | Grafana | Instalação, integração com o Zabbix, dashboard de CPU, RAM, disco e rede | *a documentar* |
| **5** | GLPI | Apache, PHP, GLPI 11, usuários e perfis, categorias ITIL, ativos, chamados, SLA | *a documentar* |
| **6** | Integração | Alerta no Zabbix, investigação e registro do incidente no GLPI | *a documentar* |
| **7** | Simulação profissional | Incidentes provocados, alerta, investigação, solução e chamado | [📄 fase-07-simulacao.md](docs/fase-07-simulacao.md) |

---

## 🧱 Destaques por Fase

### Fase 1 — Servidor Ubuntu
- VM dimensionada para um host de 16 GB: 6 GB de RAM, 4 vCPUs e 60 GB de disco.
- Acesso remoto por SSH e teste de conectividade em camadas (IP com `8.8.8.8` e DNS com `google.com`).
- **Problema resolvido:** o instalador com LVM só alocou ~29 GB dos 60 GB. O volume raiz foi estendido para ~58 GB com `lvextend -r`, sem desmontar o disco.

### Fase 2 — Banco de dados
- Uma única instância MariaDB com dois bancos separados, `zabbix` (`utf8mb4_bin`) e `glpi` (`utf8mb4_unicode_ci`), cada um com usuário próprio (princípio do menor privilégio).
- `mariadb-secure-installation`: remoção de usuários anônimos, do banco `test` e do login remoto do root.

### Fase 3 — Zabbix
- Zabbix Server, Agent 2 e frontend sobre Apache e MariaDB.
- Host `SRV-MONITORAMENTO` monitorado com itens de CPU, memória, disco, rede e serviços.

### Fase 4 — Grafana
- Plugin do Zabbix configurado como fonte de dados e consultas validadas no Explore.
- Dashboard com painéis de CPU, memória, disco e tráfego de rede (`enp0s3`).

### Fase 5 — GLPI 11 (Central de Suporte)
- Perfis **Self-Service** (Maria Solicitante) e **Technician** (TI Técnico).
- Seis categorias ITIL: Hardware, Software, Rede e conectividade, Acesso e permissões, Impressoras e Servidores.
- Ativo `SRV-MONITORAMENTO` (inventário `LAB-SRV-001`).
- SLAs de **TTO** (atribuição em até 24 h) e **TTR** (solução em até 48 h), em calendário 24/7.
- Chamado nº 1 conduzido por todo o ciclo, da abertura ao status **Fechado**.

### Fase 6 — Integração
- Alerta detectado no Zabbix, investigado e registrado como chamado nº 2 no GLPI.

### Fase 7 — Simulação profissional
Incidentes que dispararam alerta, com recuperação e chamado fechado:

| Incidente | Alerta (BRT) | Recuperação (BRT) | Chamado |
| :--- | :---: | :---: | :---: |
| Serviço `cron` parado | 19:59:03 | 20:15:03 | nº 3 |
| Disco a 86% | 20:37:42 | 20:57:42 | nº 4 |
| Sem conectividade externa | 21:28:28 | 21:43:28 | nº 5 |

---

## ⚠️ Limitações e Problemas Conhecidos

- **Instabilidade da VM (Problema nº 1):** a VM apresentou *soft lockups* e *RCU stalls*. A causa provável é o VirtualBox 7.2.16 em modo NEM no Windows 11, mas **não foi confirmada**. Durante os travamentos usava-se o kernel `7.0.0-38-generic`; ao iniciar o `7.0.0-34-generic` não surgiram novas mensagens na primeira consulta, e a observação continuou. Decidi não reduzir a segurança do Windows, então o problema segue aberto.
- **Teste de CPU inconclusivo:** o teste com `stress-ng` deixou a VM sem responder por um tempo, e por isso foi registrado como inconclusivo na Fase 7.

---

## 🛠️ Tecnologias e Ferramentas

* **Virtualização:** Oracle VirtualBox
* **Sistema operacional:** Ubuntu Server 26.04.1 LTS
* **Monitoramento e visualização:** Zabbix, Grafana
* **ITSM:** GLPI 11
* **Banco e web:** MariaDB, Apache, PHP
* **Ferramentas de diagnóstico:** `systemctl`, `journalctl`, `ss`, `htop`, `ping`, `df`, `free`, SSH

---

## 🎓 Habilidades Demonstradas

- Administração Linux (Ubuntu Server, `apt`, `systemd`, LVM, SSH)
- Monitoramento de infraestrutura (Zabbix: itens, triggers, problemas)
- Visualização de métricas (Grafana)
- Service Desk e ITSM (GLPI: chamados, categorias, prioridade, SLA, ativos)
- Troubleshooting e análise de logs
- Documentação técnica de incidentes

---

## ✍️ Autor

Desenvolvido por **Guilherme Ponciano**  
*Profissional de Tecnologia da Informação com foco em Infraestrutura, Suporte Avançado e SysAdmin.*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/g-ponciano)
[![GitHub](https://img.shields.io/badge/GitHub-100000?style=for-the-badge&logo=github&logoColor=white)](https://github.com/g-ponciano)
