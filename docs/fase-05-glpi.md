# Fase 5 — GLPI (Service Desk / ITSM)

> **Objetivo:** instalar o GLPI 11 na mesma VM, endurecer a instalação, estruturar a central de suporte (usuários, perfis, categorias, ativo e SLAs) e validar o fluxo completo de um chamado, da abertura ao fechamento.

| Item | Valor |
| :--- | :--- |
| GLPI | **11.0.11** (pacote oficial `.tgz` do GitHub) |
| Diretório | `/var/www/glpi` |
| Acesso | `http://192.168.100.86/glpi/` |
| Servidor web | Apache + PHP 8.5.4 |
| Banco | MariaDB 11.8.6, banco `glpi` e usuário dedicado `glpi` |
| Período | 12/10 a 15/10/2026 |

## Arquitetura da fase

```text
   Navegador ──http :80──► Apache ──Alias /glpi──► /var/www/glpi/public (PHP)
                                                        │
                                                        ▼
                                            MariaDB ── banco "glpi"
                                  (o banco "zabbix" convive na mesma instância)
```

---

## Linha do tempo

| Data | Evento |
| :--- | :--- |
| 12/10 | Snapshot do VirtualBox **Antes do GLPI 11** |
| 13/10 | Pré-requisitos: pacotes PHP, módulos do Apache, ajustes do `php.ini`, banco e usuário `glpi` |
| 13/10 – 14/10 | Download, extração, configuração do Apache e assistente de instalação |
| 14/10 | Endurecimento (senhas padrão, permissões, instalador bloqueado), snapshot, usuários, perfis, categorias e ativo |
| 15/10 | Chamado #1 de ponta a ponta, níveis de serviço (SLA) e fechamento |

---

## Etapa 1 — Pré-requisitos

Antes de começar foi criado o snapshot **Antes do GLPI 11** no VirtualBox, para poder voltar atrás.

```bash
sudo apt install php-cli php-curl php-gd php-intl php-mysql php-bcmath \
  php-mbstring php-xml php-zip php-bz2 php-ldap php-apcu
```

O `php-opcache` não entrou na lista final: no PHP 8.5 o OPcache já vem embutido (ver Problemas, item 1). Conferências feitas: `php -v` (8.5.4), extensões `fileinfo`, `dom`, `filter`, `session`, `tokenizer` e `zlib` presentes, Apache e MariaDB ativos.

Módulos do Apache e reescrita de URL:

```bash
sudo apachectl -M | grep -E 'rewrite|php|proxy_fcgi'
sudo a2enmod rewrite
```

![Módulos do Apache](../images/f5-01-apache-modulos.png)

O banco `glpi` e o usuário `glpi@localhost` já existiam (0 tabelas). Ajustes no `/etc/php/8.5/apache2/php.ini`:

| Diretiva | Antes | Depois |
| :--- | :-: | :-: |
| `max_execution_time` | 30 | 120 |
| `memory_limit` | 128M | 256M |
| `post_max_size` | 8M | 20M |
| `upload_max_filesize` | 2M | 20M |
| `session.cookie_httponly` | vazio | 1 |

![php.ini ajustado](../images/f5-02-php-ini-ajustado.png)

## Etapa 2 — Download e instalação dos arquivos

```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/11.0.11/glpi-11.0.11.tgz
sudo tar -xzf /tmp/glpi-11.0.11.tgz -C /var/www
sudo chown -R root:root /var/www/glpi
sudo chown -R www-data:www-data /var/www/glpi/config /var/www/glpi/files
```

![Download do GLPI](../images/f5-03-download-glpi.png)

O código fica com `root` (somente leitura para o servidor web); só `config` e `files`, onde o GLPI escreve, pertencem ao `www-data`.

## Etapa 3 — Apache

Arquivo `/etc/apache2/conf-available/glpi.conf`, apontando o alias para a pasta `public` (não para a raiz do GLPI):

```apache
Alias /glpi /var/www/glpi/public
<Directory /var/www/glpi/public>
    Options FollowSymLinks
    AllowOverride None
    Require all granted
    RewriteEngine On
    RewriteCond %{HTTP:Authorization} ^(.+)$
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%1]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ index.php [QSA,L]
</Directory>
```

```bash
sudo a2enconf glpi
sudo apachectl configtest      # Syntax OK
sudo systemctl reload apache2
curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1/glpi/   # HTTP 200
```

O aviso `AH00558` sobre `ServerName` aparece no `configtest` e não afeta o funcionamento.

## Etapa 4 — Assistente de instalação

Acesso em `http://192.168.100.86/glpi/install/install.php`: escolha do idioma, aceite da licença e verificação de compatibilidade. Os itens obrigatórios passaram; o único aviso restante era o diretório do marketplace (ver Problemas, item 3).

![Verificação de requisitos](../images/f5-06-requisitos-instalador.png)

Na etapa do banco foram informados servidor `localhost` e o usuário `glpi`. O assistente criou as tabelas e a instalação terminou na tela de login.

![Tela de login](../images/f5-07-login.png)

## Etapa 5 — Endurecimento (hardening)

**Senhas padrão.** O primeiro login mostrou o aviso de segurança do Dashboard sobre as contas padrão `post-only`, `tech` e `normal`. Foram trocadas as senhas dessas contas e a do usuário `glpi`, e o aviso desapareceu.

![Dashboard com aviso](../images/f5-08-dashboard-aviso-seguranca.png)
![Dashboard sem aviso](../images/f5-09-dashboard-sem-aviso.png)

**Conta administrativa pessoal.** Criado o usuário `admin.pessoal` com perfil Super-Admin e testado em janela anônima.

**Instalador bloqueado.** Depois da instalação, `install.php` responde com HTTP 302 para `/glpi/index.php`.

**Permissões.**

```bash
sudo chmod 750 /var/www/glpi/config
sudo chmod 640 /var/www/glpi/config/config_db.php /var/www/glpi/config/glpicrypt.key /var/www/glpi/config/oauth.pub
sudo chmod 600 /var/www/glpi/config/oauth.pem
sudo chmod 750 /var/www/glpi/files /var/www/glpi/files/_log
curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1/glpi/   # HTTP 200
```

![chmod em config](../images/f5-10-permissoes-config.png)
![Permissões em config](../images/f5-11-config-ls.png)
![Permissões em files](../images/f5-12-files-ls.png)

Depois das conferências foi criado um novo snapshot, **GLPI 11 instalado — segurança validada**.

![Snapshots no VirtualBox](../images/f5-13-snapshot-virtualbox.png)

---

## Etapa 6 — Usuários e perfis

| Usuário | Perfil | Função |
| :--- | :--- | :--- |
| `maria.solicitante` | Self-Service | Solicitante (abre chamados) |
| `tecnico.ti` (exibido como **TI Técnico**) | Technician | Atende os chamados |
| `admin.pessoal` | Super-Admin | Administração |

![Lista inicial de usuários](../images/f5-14-usuarios-iniciais.png)
![Usuário Maria](../images/f5-15-usuario-maria.png)

O `tecnico.ti` foi criado como Self-Service e a lista "Perfil padrão" só oferecia esse perfil. Foi adicionada uma autorização **Technician** na entidade raiz (sem recursão) e testada com login. Em seguida a autorização Self-Service foi removida, mantendo Technician como padrão.

![Autorização Technician](../images/f5-16-tecnico-autorizacao.png)

## Etapa 7 — Categorias ITIL

Criadas em **Configurar → Listas suspensas → Categorias ITIL**: Hardware, Software, Rede e conectividade, Acesso e permissões, Impressoras e Servidores.

![Categorias ITIL](../images/f5-17-categorias-itil.png)

## Etapa 8 — Ativo no inventário

Computador **SRV-MONITORAMENTO**, número de inventário `LAB-SRV-001`, com o comentário: *"Servidor virtual Ubuntu Server executado no VirtualBox. Endereço IP: 192.168.100.86. Serviços: Zabbix, Grafana e GLPI 11."*

![Ativo SRV-MONITORAMENTO](../images/f5-18-ativo-computador.png)

## Etapa 9 — Níveis de serviço (SLA)

Criado o nível de serviço **Central de Suporte - Homelab**, com calendário 24/7 e dois SLAs:

| SLA | Tipo | Prazo |
| :--- | :--- | :-: |
| TTO - Atribuição em até 24 horas | Tempo para atendimento | 24 h |
| TTR - Solução em até 48 horas | Tempo para solução | 48 h |

![Nível de serviço](../images/f5-21-nivel-servico.png)
![SLAs](../images/f5-22-slas.png)

## Etapa 10 — Chamado #1 de ponta a ponta

**Abertura (14/10).** Logada como Maria (Self-Service), a solicitante abriu o chamado **#1 "Computador não liga"**, categoria Hardware, urgência, impacto e prioridade médias. Status: **Novo**.

![Maria na central de serviços](../images/f5-19-maria-self-service.png)
![Chamado novo](../images/f5-20-chamado-novo.png)

**Atendimento (15/10).** Pelo perfil de administrador, o chamado foi atribuído ao **TI Técnico** (campo Atores), recebeu acompanhamento com o diagnóstico inicial e os dois SLAs (TTO e TTR). Ao associar um SLA, o GLPI avisa que as datas serão recalculadas. Por fim foi registrada uma **solução fictícia** (simulação de laboratório) e a Maria aprovou.

![Conversa do chamado](../images/f5-23-chamado-conversa.png)

**Fechamento.** Com a solução aprovada, o status passou a **Fechado**.

![Chamado fechado](../images/f5-24-chamado-fechado.png)

---

## ⚠️ Problemas encontrados e como foram resolvidos

| # | Problema | Causa | Solução |
| :-: | :--- | :--- | :--- |
| 1 | `apt install` falhou com `Package 'php-opcache' has no installation candidate` | No PHP 8.5 o OPcache é embutido e não existe mais como pacote separado | Repetir o comando sem `php-opcache` |
| 2 | `apt install` ficou aguardando o *lock* do dpkg | O `unattended-upgrades` (processo 2485) estava usando o gerenciador de pacotes | Aguardar o fim e repetir; o `php-apcu` foi instalado na nova tentativa |
| 3 | Aviso no instalador: o GLPI não conseguiu criar `/var/www/glpi/marketplace` | O diretório pertencia ao `root` (código somente leitura para o servidor web) | `sudo chown www-data:www-data /var/www/glpi/marketplace` |
| 4 | O assistente respondeu "A ação que você requisitou não é permitida" | Token de sessão/CSRF rejeitado pelo GLPI na sessão do navegador | Refazer o passo em janela anônima |
| 5 | Travamentos da VM (*soft lockup*) em 12/10, 14/10 e 15/10 durante o trabalho | Instabilidade da VM no VirtualBox, registrada no README (Limitações) | Aguardar a VM voltar; em 15/10 o kernel 7.0.0-34 foi iniciado pelo GRUB como teste, sem novas mensagens na primeira checagem. Não considero definitivamente resolvido |

---

## ✅ Checklist da Fase 5

- [x] Snapshot antes da instalação
- [x] Pacotes PHP, módulos do Apache e `php.ini` ajustados
- [x] GLPI 11.0.11 baixado e extraído em `/var/www/glpi`
- [x] Alias do Apache apontando para `public`, respondendo HTTP 200
- [x] Instalação concluída pelo assistente
- [x] Senhas das contas padrão trocadas e aviso de segurança removido
- [x] Instalador bloqueado e permissões de `config` e `files` restritas
- [x] Usuários e perfis (Self-Service, Technician, Super-Admin)
- [x] Seis categorias ITIL
- [x] Ativo `SRV-MONITORAMENTO` (`LAB-SRV-001`)
- [x] SLAs TTO (24 h) e TTR (48 h)
- [x] Chamado #1 aberto, atribuído, atendido, resolvido, aprovado e fechado

## 🎓 O que esta fase demonstra

- Instalação de uma aplicação PHP sobre Apache e MariaDB, com diagnóstico de dependências e permissões.
- Hardening básico pós-instalação: senhas padrão, instalador bloqueado e permissões mínimas.
- Operação de um service desk: perfis, categorias, inventário, SLA e ciclo completo de um chamado.

**Próxima fase:** Fase 6 — Integração entre Zabbix, Grafana e GLPI.
