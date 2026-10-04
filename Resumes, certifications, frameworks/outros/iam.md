# IAM
O termo IAM é sigla para "Identity and Access Management", IAM foi criado para resolver um problema em específico que todas as empresas já se depararam ou ainda irão se deparar: a identidade certa precisa ter o acesso certo, ao recurso certo, pelo motivo certo, e nada mais.

## Auth
Quando alguém fala sobre autenticação, precisasse definir se a pessoa está se referindo a autenticação de identidade (**AuthN**) ou à autorização (**AuthZ**). AuthN responde "Você é quem você diz ser?" e AuthZ responde "O que você é permitido fazer?".

A partir da autenticação, é criado os modelos de acesso. Os mais comuns são **RBAC** (Role-Based Access Control), que configura as autorizações a partir de que "cargo" ou "papel" você desempenha, e **ABAC** (Attribut-Based Access Control), que configura as autorizações a partir de regras predefinidas que se relacionam.

Para os modelos de acesso, há dois princípios que todos utilizam: **least privilege** (menor privilégio) e **segregation of duties** (segregação de funções). 
- _Least privilege_: define que ao se definir um usuário final de algum sistema, começasse sem nenhum acesso e então irá adicionando estritamente o necessário, que é diferente de começar com todos as permissões e então ir removendo as desnecessárias.  O segundo caso é mais propenso a gerar mais erros e problemas futuramente.
- _Segragation of duties_: Nenhuma pessoa sozinha deve ser capaz de realizar uma operação macro ou micro de importância sozinha. Por exemplo, adicionar um fornecedor ao banco de dados E aprovar o pagamento para o mesmo.

Para toda identidade, há um **ciclo de vida**. Provisionamento -> revisão periódica -> desprovisionamento, também conhecido como **ITGC**.

Como é necessário que alguém realize as configurações críticas de um ambiente desse, e de outras áreas não somente IAM, existe as contas com acesso privilegiado, o conhecido administrador, root ou qualquer coisa que consiga realizar essas configurações. Esse tipo de conta requer uma atenção mais que redobrada, pois podem destruir todo um ambiente caso usada de forma errônea. MFA, uso restrito, monitoramento, tudo para realizar o controle de acesso privilegiado (**PAM**).

## Provedores de Nuvem
Provedores como a Amazon Web Services (**AWS**) e a Microsoft Azure (**Azure**) são nada mais que containers onde concentra todas essas configurações e logística. Uma das alternativas para se evitar criar usuários IAM soltos no sistema, para evitar muitas chaves de acesso que não expiram. É utilizado o método "papel/cargo IAM" (IAM Role). O usuário assume uma identidade temporária por um curto período de tempo e depois a perde, evitando que as senhas sejam permanentemente armazenadas no código. 

A permissão em si — o que cada entidade pode fazer — é definida em políticas, documentações em formato JSON com quatro pilares centrais: **Effect** (permitir ou negar algo), **Action** (qual operação), **Resource** (Para qual recurso específico) e **Condition** (Filtro extra). Essas políticas podem ser tanto anexadas na identidade (**identity-based**) ou anexadas ao recurso (**resource-based**), sendo recomendado que faça de forma que **ambos** estejam funcionando.

Há outros termos e ferramentas como IAM Identity Center — uma identidade dando acesso à várias contas sem multiplicar usuário solto por conta — e o CloudTail — o que grava registro por registo de eventos, quem fez, quando, de onde — que irão variar de provedor para provedor mas todos tem o objetivo de tornar melhor o acesso e controle.
