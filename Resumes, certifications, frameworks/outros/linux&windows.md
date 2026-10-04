# Linux — Fundamentos para Análise de Segurança

## Modelo de permissões (dono/grupo/outros)

Todo arquivo e diretório no Linux tem três entidades associadas: um **dono** (owner), um **grupo** (group) e **outros** (others — todo o resto). Para cada uma dessas três categorias existem três permissões possíveis: **r** (read), **w** (write), **x** (execute). Isso é representado numericamente (notação octal): r=4, w=2, x=1, somados por categoria. Um arquivo `644` significa dono com leitura+escrita (6), grupo com leitura (4), outros com leitura (4) — é a permissão padrão de um arquivo comum. Um `755` é o padrão de um executável ou diretório navegável.

O ponto de atenção real para um analista: permissões `777` (todo mundo pode ler, escrever e executar) ou arquivos *world-writable* são bandeira vermelha imediata — qualquer usuário do sistema, mesmo sem privilégio nenhum, pode alterar aquele arquivo.

Existem três bits especiais além do básico, e eles importam especificamente para segurança:
- **SUID** (Set User ID): um executável com esse bit roda com a permissão do *dono* do arquivo, não de quem o executou. É assim que o comando `passwd` (que precisa escrever em `/etc/shadow`, área só de root) funciona para um usuário comum — e é também o vetor clássico de escalonamento de privilégio quando mal configurado.
- **SGID**: equivalente, mas herda o grupo.
- **Sticky bit**: comum em diretórios compartilhados tipo `/tmp` — impede que um usuário apague arquivo de outro usuário mesmo tendo permissão de escrita no diretório.

## Root e sudo

Root é o UID 0 — o único usuário sem restrição nenhuma no sistema. A prática correta é nunca logar como root diretamente; em vez disso, usuários comuns usam `sudo` para executar comandos pontuais com privilégio elevado, mediante senha própria. Quem pode usar `sudo` e para quê é definido em `/etc/sudoers` (editado com o comando `visudo`, que valida a sintaxe antes de salvar — editar esse arquivo direto no vi e salvar errado pode te trancar fora do sistema).

O motivo de isso importar para auditoria: cada uso de `sudo` é registrado com usuário, comando exato e timestamp — é uma trilha de auditoria nativa. Um ambiente bem configurado dá a cada pessoa só os comandos `sudo` específicos que ela precisa (ex: só reiniciar um serviço), não acesso root irrestrito — é *least privilege* aplicado na prática.

## Logs

Tudo fica, por convenção, em `/var/log`. O arquivo mais importante para investigação de acesso é o de autenticação — mas o nome varia por distribuição: `/var/log/auth.log` em Debian/Ubuntu, `/var/log/secure` em RHEL/CentOS/Fedora. É nele que aparecem tentativas de login (sucesso e falha), uso de `sudo`, e conexões SSH.

Nas distribuições mais recentes, o **systemd** centralizou boa parte do log em um formato binário consultado com `journalctl` — por exemplo, `journalctl -u ssh` mostra só os logs do serviço SSH, e `journalctl --since "1 hour ago"` filtra por tempo. Isso substituiu parte do que antes só existia em arquivo texto solto.

Ponto crítico de segurança: se o log só existe localmente na máquina, um atacante com acesso root pode simplesmente apagá-lo para cobrir rastro. Por isso ambientes maduros enviam log para um destino externo centralizado (um SIEM, que é exatamente o próximo assunto que você vai me perguntar) — log que não sai da máquina comprometida não é uma evidência confiável.

## Processos

`ps aux` ou `ps -ef` listam todos os processos rodando no sistema, com usuário dono, PID, uso de CPU/memória e o comando completo que iniciou o processo. `top` ou `htop` dão a versão interativa, atualizada em tempo real.

O que um analista procura aqui: processo rodando a partir de um diretório incomum (`/tmp`, `/dev/shm`), processo com nome parecido mas sutilmente diferente de um processo legítimo (ex: `sshdd` em vez de `sshd`), processo órfão (sem processo pai esperado), ou uso de CPU/memória fora do padrão para aquele sistema.

## Rede

`ss` é o comando atual (substituiu o `netstat`, que ainda existe mas está formalmente obsoleto na maioria das distros). `ss -tulnp` mostra portas TCP/UDP escutando, com o processo dono de cada uma. Isso é, na prática, a superfície de ataque exposta daquela máquina em forma de comando — toda porta escutando desnecessária é um risco que não precisava existir.

## Persistência

Mecanismo pelo qual um atacante garante que o acesso sobrevive a um reboot ou ao fechamento da sessão inicial:
- **cron**: tarefas agendadas, visíveis com `crontab -l` (do usuário atual) ou nos arquivos em `/etc/cron.d/` e `/etc/crontab` (nível de sistema, mais frequentemente abusado porque passa despercebido).
- **Serviços systemd**: um atacante pode criar uma unit file disfarçada de serviço legítimo que executa algo malicioso na inicialização.
- **SSH authorized_keys**: inserir uma chave pública própria em `~/.ssh/authorized_keys` de um usuário comprometido garante acesso futuro sem precisar de senha — é um dos métodos de persistência mais simples e mais comuns.

---

# Linux Server — O que muda em relação ao desktop

A diferença central: um servidor não existe para alguém usar interativamente, existe para expor um ou mais **serviços** pela rede (web, banco de dados, SSH). Isso muda completamente onde o risco se concentra — não é mais sobre o que um usuário clica, é sobre o que está escutando na rede e quem consegue alcançar isso.

## SSH — a porta de entrada administrativa

Como ninguém senta na frente de um servidor fisicamente, SSH é praticamente o único jeito de administrar a máquina, e por isso é o alvo número um de ataque de força bruta na internet inteira. O arquivo de configuração é `/etc/ssh/sshd_config`, e três ajustes ali separam um servidor básico de um servidor minimamente hardenizado: `PermitRootLogin no` (ninguém loga como root direto via SSH, tem que entrar como usuário comum e usar sudo depois — isso cria rastro de auditoria), `PasswordAuthentication no` (só autenticação por chave pública, que é praticamente imune a força bruta comparado a senha) e, opcionalmente, mudar a porta padrão 22 (não impede um atacante determinado, mas corta um volume enorme de varredura automatizada burra). O **Fail2ban** é a ferramenta padrão de mercado para banir automaticamente IPs com muitas tentativas de login falha.

## Firewall

`iptables`/`nftables` são o motor por baixo; `ufw` (Ubuntu) e `firewalld` (RHEL/Fedora) são interfaces mais simples por cima. O princípio que importa é o mesmo *least privilege* de sempre, aplicado a porta de rede: a regra padrão deveria ser negar tudo e abrir só a porta estritamente necessária pro serviço daquele servidor — um servidor de banco de dados não deveria ter porta 3306 (MySQL) acessível da internet aberta, só da rede interna ou de IPs específicos da aplicação.

## Gestão de patches e vulnerabilidades

Aqui é onde Linux server encosta direto em ITGC: `apt`/`yum`/`dnf` gerenciam pacotes, mas a pergunta de auditoria real não é "o comando existe", é "existe um processo formal e documentado de quando e como patches são aplicados, e alguém testa antes de produção?" Servidor que nunca é atualizado por medo de quebrar algo é um dos achados mais comuns em auditoria de infraestrutura — risco inerente alto, controle compensatório ausente.

## Hardening além da permissão básica

**SELinux** (RHEL/CentOS) e **AppArmor** (Ubuntu/Debian) são sistemas de controle de acesso obrigatório (MAC) que vão além do modelo dono/grupo/outros — mesmo que um processo rode como root, essas ferramentas podem restringir o que ele fisicamente consegue tocar no sistema, baseado em uma política explícita. É comum encontrar esses mecanismos desativados em servidor real "porque dava erro e alguém desligou" — isso também é achado de auditoria clássico.

Os **CIS Benchmarks** (Center for Internet Security) são o checklist padrão de mercado para hardening de Linux Server — é literalmente a referência que um auditor usa para comparar "como o servidor está configurado" contra "como deveria estar configurado". Vale conhecer que existe, mesmo sem decorar item por item.

## Serviços rodando com usuário próprio

Um servidor web bem configurado roda o processo do Nginx/Apache como um usuário de serviço dedicado (`www-data`, por exemplo), nunca como root — se o serviço for comprometido via uma falha na aplicação, o atacante herda só o privilégio daquele usuário limitado, não controle total da máquina. Isso é *least privilege* aplicado a processo, não só a pessoa.

## Logging centralizado

Em ambiente com múltiplos servidores, log que fica só localmente em cada máquina não escala e não é confiável (lembra: atacante com root apaga log local). `rsyslog` ou `syslog-ng` encaminham log de todos os servidores para um coletor central — que normalmente alimenta um SIEM, assunto da próxima conversa.

---

# Windows — Fundamentos para Análise de Segurança

## Active Directory

A maioria dos ambientes corporativos que você vai auditar ou defender roda em domínio Windows, então este é o conceito mais importante de toda a seção. O **Domain Controller** (DC) é o servidor que centraliza autenticação e política para toda a rede. As contas e máquinas são organizadas em **OUs** (Organizational Units) — uma estrutura hierárquica de pastas que agrupa usuários/computadores por critério (departamento, localização, função). As **GPOs** (Group Policy Objects) são o mecanismo pelo qual uma regra é aplicada em massa a tudo dentro de uma OU — desde "senha precisa ter 12 caracteres" até "bloquear execução de determinado programa" em todas as máquinas daquele grupo de uma vez.

Isso é, essencialmente, o RBAC que vimos no IAM, só que aplicado à máquina física e ao domínio inteiro, não a um serviço cloud isolado.

## Permissões (NTFS / ACL)

O sistema de arquivos NTFS usa **ACL** (Access Control List) — uma lista explícita de entradas Allow/Deny por usuário ou grupo, bem mais granular que o modelo dono/grupo/outros do Linux. Permissões podem ser herdadas de pastas pai ou definidas explicitamente em um arquivo específico. O comando `icacls` mostra e edita essas listas via linha de comando.

## Logs de Evento (Event Viewer)

O Windows registra eventos em categorias separadas — Application, Security, System, Setup, Forwarded Events — sendo **Security** a que mais interessa para um analista. Cada tipo de evento tem um número (Event ID) fixo. Os que valem decorar:

| Event ID | Significado |
|---|---|
| 4624 | Login bem-sucedido |
| 4625 | Login que falhou (procurado em padrão de força bruta) |
| 4648 | Login usando credencial explícita (alguém rodou algo "como outro usuário") |
| 4672 | Login com privilégio administrativo |
| 4688 | Novo processo criado — essencial para rastrear o que foi executado e quando |
| 4720 | Conta de usuário criada |
| 1102 | **Log de auditoria foi limpo** — isso quase sempre é sinal de alguém tentando apagar rastro, porque um usuário legítimo raramente tem motivo pra limpar o log de segurança |

## PowerShell

É a ferramenta de administração mais poderosa do Windows moderno — e por isso também a mais usada por atacante depois de já ter acesso inicial (é "living off the land": usar ferramenta nativa do sistema em vez de trazer malware externo, o que dificulta detecção por antivírus tradicional). Existem três camadas de logging que um ambiente bem configurado deveria ter ativadas: **Module Logging** (quais módulos foram carregados), **Script Block Logging** (o conteúdo real do script executado, registrado como Event ID 4104) e **Transcription** (grava a sessão inteira em um arquivo texto). Sem isso ativado, um atacante pode rodar comandos PowerShell inteiros sem deixar rastro nenhum no Event Viewer padrão.

## Persistência

Equivalente windows do cron: **Task Scheduler** (tarefas agendadas, visíveis em `taskschd.msc`) e as chaves de registro conhecidas como **Run keys** (`HKCU\...\Run` e `HKLM\...\Run`) — qualquer programa listado ali executa automaticamente no login. Mais avançado, mas vale saber que existe: **WMI event subscriptions**, um mecanismo de automação do Windows que pode ser abusado para disparar código malicioso em resposta a um evento do sistema, sem deixar entrada óbvia em Task Scheduler.

## Sysinternals (Microsoft, gratuito)

É o kit prático que todo analista júnior de Windows usa de verdade, muito mais do que a interface padrão do sistema:
- **Process Explorer**: Task Manager avançado — mostra árvore de processo pai/filho, hash do executável, e sinaliza se um processo não está assinado digitalmente.
- **Autoruns**: lista *todo* mecanismo de inicialização automática do sistema de uma vez (Run keys, serviços, tarefas agendadas, extensões de navegador) — é a ferramenta número um para caçar persistência.
- **TCPView**: versão gráfica e em tempo real do `netstat`.
- **Sysmon**: não é uma ferramenta que você abre, é um serviço que fica instalado permanentemente registrando criação de processo, conexão de rede e modificação de arquivo em um nível de detalhe muito maior que o Event Viewer padrão — é praticamente o padrão de mercado para quem monta visibilidade de segurança em ambiente Windows.

---

# Windows Server — O que muda em relação ao desktop

A diferença central aqui é ainda mais marcante que no Linux: Windows Server não roda aplicativo de usuário, ele roda **roles** (papéis de infraestrutura) — Active Directory Domain Services, DNS, DHCP, File Server, IIS (web), Hyper-V (virtualização). O servidor que hospeda o Active Directory é, na prática, a joia da coroa de qualquer rede corporativa: quem compromete o Domain Controller compromete a autenticação de tudo.

## Autenticação Kerberos e os ataques clássicos de AD

Active Directory usa **Kerberos** como protocolo de autenticação (não é só login e senha simples — envolve tickets criptografados com tempo de validade). Dois ataques merecem menção porque aparecem o tempo todo em relatório de pentest e em caso real: **Kerberoasting** (um atacante pede um ticket de serviço e tenta quebrar o hash offline pra descobrir a senha de uma conta de serviço, que costuma ter privilégio alto e senha nunca trocada) e **Golden Ticket** (depois de comprometer o Domain Controller, o atacante forja um ticket Kerberos que concede acesso praticamente ilimitado e quase indetectável). Você não precisa saber executar isso — precisa saber que esses nomes existem e por que uma conta de serviço com senha fraca e nunca rotacionada é um risco crítico, não cosmético.

## Modelo de administração em camadas (Tiered Model)

A Microsoft recomenda formalmente separar a administração em **Tier 0** (Domain Controllers e tudo que controla o domínio), **Tier 1** (servidores membros) e **Tier 2** (estações de trabalho de usuário comum) — com a regra de que uma credencial de um tier mais privilegiado nunca deveria logar em uma máquina de tier menos privilegiado. Isso existe porque o ataque mais comum em rede Windows é lateral: comprometer uma estação fraca de usuário comum e ir "subindo" até achar uma sessão de admin de domínio esquecida logada em algum lugar errado. **PAW** (Privileged Access Workstation) é a prática de ter uma máquina fisicamente separada, blindada, só para administrar o domínio — nunca misturar "máquina que eu uso pra navegar na internet" com "máquina que eu uso pra administrar o AD".

## LAPS — senha de admin local

Por padrão, muita empresa configura a mesma senha de Administrator local em todas as máquinas via imagem — o que significa que comprometer uma estação dá a senha de admin local de toda a frota. O **LAPS** (Local Administrator Password Solution, hoje nativo no Windows Server mais recente) randomiza e rotaciona essa senha automaticamente por máquina, guardando o valor atual de forma segura no próprio Active Directory.

## RDP e WinRM — as portas administrativas remotas

Assim como SSH é a porta de entrada no Linux, **RDP** (porta 3389) é a principal via de administração remota do Windows Server — e também um dos vetores de ransomware mais explorados da história recente, quando exposto direto à internet com senha fraca. **NLA** (Network Level Authentication) exige autenticação antes mesmo de estabelecer a sessão gráfica completa, reduzindo superfície de ataque. **WinRM** é o equivalente para administração via PowerShell remoto (`Enter-PSSession`), e merece a mesma cautela de exposição.

## SMB e a lição do WannaCry

O protocolo SMB (compartilhamento de arquivo do Windows) teve uma versão antiga, **SMBv1**, com uma vulnerabilidade (EternalBlue) que permitiu o ransomware WannaCry se espalhar sozinho pela rede em 2017 sem nenhuma ação do usuário. Até hoje, desabilitar SMBv1 explicitamente é item de checklist de hardening — é o exemplo mais citado de por que manter protocolo legado ativo "porque sempre foi assim" é risco real, não teórico.

## Logging específico de servidor

Além do Security log que já vimos, o Domain Controller tem o log **Directory Service** (eventos específicos do AD) e o **DNS Server log**. Em ambiente maduro, os logs de todos os servidores são centralizados via **WEF** (Windows Event Forwarding) para um coletor único — o mesmo princípio do rsyslog no Linux, evitando depender de log que mora só na máquina que pode ter sido comprometida.

---

## Comparação rápida — camada de servidor

| Conceito | Linux Server | Windows Server |
|---|---|---|
| Acesso administrativo remoto | SSH (porta 22) | RDP (3389) / WinRM |
| Hardening do acesso remoto | Chave pública, sem root direto, Fail2ban | NLA, LAPS, modelo em camadas (Tiers) |
| Autenticação central | — (cada servidor isola, salvo LDAP próprio) | Kerberos via Active Directory |
| Controle de acesso obrigatório | SELinux / AppArmor | — (controlado via ACL + GPO) |
| Checklist de hardening padrão | CIS Benchmarks (Linux) | CIS Benchmarks (Windows Server) |
| Falha histórica notória | — | SMBv1 / EternalBlue (WannaCry) |
| Log centralizado | rsyslog / syslog-ng → SIEM | WEF → coletor central → SIEM |
