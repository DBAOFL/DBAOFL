# AAA Framework
O framework AAA tem como objetivo padronizar o processo de identificação, autenticação, autorização e registro dos logs de login de uma empresa.

## Autenticação
Um usuário final, sendo cliente ou funcionário, tenta acessar um serviço ou dado. Esse usuário precisará comprovar que possui acesso ao que deseja. O processo ocorreria da seguinte forma:
- Usuário pede acesso aos serviços e dados desejados por meio de um sistema de autenticação;
- A requisição é encaminhada ao firewall ou concentrador de VPN (VPNC);
- O firewall/VPNC faz uma requisição para onde as informações de login de usuário são guardadas: um servidor AAA;
- Caso as informações fornecidas pelo usuário forem legítimas e estiverem registradas no servidor AAA, esse irá retornar ao firewall/VPNC que o login pode ser permitido. Caso contrário, negado;
- Com a resposta do servidor AAA, o firewall/VPNC irá liberar ou negar o acesso aos serviços e/ou dados.

### Certificado assinado digitalmente nos dispositivos
Muitas vezes a empresa não irá guardar os dados de autenticação dos usuários em um servidor ou máquina. Uma alternativa para isso seria utilizando certificados digitais.
Uma vez que a empresa empresa possua uma Autoridade de Certificados (CA) confiável, o processo ocorre da seguinte forma:
- A empresa cria um certificado para um dispositivo;
- O certificado é assinado digitalmente com o próprio CA da empresa para validação;
- O CA possui o próprio certificado, que é comparado com o certificado do dispositivo final para prosseguir com autenticação.

## Autorização 
Com o usuário conseguindo acesso aos serviços, a próxima questão levantada é: esse usuário possui acesso a que?
Como não é escalável adicionar permissões individualmente a cada usuário de uma empresa, tendo em vista tanto a adição de um novo usuário ou serviço e manutenção desses, utilizasse um modelo de autorização:
- Utilizar um modelo de usuário, um cargo, que já possui todas as permissões necessárias para aquele setor-> (abstração);
- Um novo usuário é associado a um cargo com as permissões pre-definidas;
- Caso precise modificar algo em um setor, mudasse no cargo, e não em cada indivíduo do setor.

## Logs de Login
Para fins de registro e monitoramento, a empresa deve registrar dados como: data do login e logout, e dados e serviços acessados, enviados e recebidos;
