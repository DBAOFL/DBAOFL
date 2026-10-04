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
