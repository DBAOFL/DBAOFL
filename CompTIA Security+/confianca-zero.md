# Confiança Zero
Muitas redes são relativamente abertas por dentro, isso é, uma vez que entrou, há poucos controles de segurança.
Zero Trust é uma alternativa holística para a segurança das redes, tratando de todo dispositivo, processo e pessoa. Nada é confiado desde o início.

## Planos de operação
Divide a rede em partes funcionais, seja para os componentes fisicos, virtuais ou de núvem.

Os planos são:
- Dados: Processa, encaminha, codifica e realiza outras operações com os dados da rede.
- Controle: Administra as ações do plano de dados, define as políticas e regras e tabelas (Roteamento, seção, NAT, etc...).

## Controle de Confiança
### Identidade Adaptativa
Como uma forma mais moderna de identificar um indivíduo ou dado, leva em consideração:
- A origem e os recursos solicitados;
- Indicadores de risco: relação para a empresa, localização física, tipo de conexão, endereço IP, etc...;
- Também torna a autenticação mais robusta se necessário.

Reduzir o número de pontos de entrada e combinar a identidade adaptativa com regras predeterminadas reforça ainda mais o processo.

## Zonas de Segurança
Segurança é mais do que uma relação um-para-um, o contexto também é importante.
- Se o solicitante é confiado ou não;
- Está vindo de rede externa ou interna;
- De que VPN está comunicando;
- De que departamento;
- etc...
Uma zona pode já ser suficiente para negar acesso, mas nunca permitir explicitamente.

## Pontos de Política
### Ponto de Reforço de Política (PEP)
Todos usuários e sistemas devem passar pelo PEP antes de acessar o recurso solicitado. 
O PEP reúne todas as informações do solicitante e envia ao PDP para que seja analisado se o tráfego pode ser permitido ou não, e permite ou nega a depender da decisão tomada no PDP.

### Ponto de Decisão de Política (PDP)
É o responsável pelo processo de decidir sobre uma autenticação.
- O Mecanismo de Política é o responsável por avaliar cada decisão de acesso baseado em políticas e nas informações coletadas pelo PEP. Permite, nega ou retira autorização.
- O Administrador de Política é o que comunica com o PEP a decisão do Mecanismo de Política se deve permitir ou negar, e gera os tokens de acesso ou credenciais.
