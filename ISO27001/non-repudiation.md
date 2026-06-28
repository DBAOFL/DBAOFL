# Não-repúdio
A principal função é confirmar e garantir que foi alguém qualificado que enviou a informação, ou seja, confirmar que a assinatura no documento é legítima.

## Prova de Integridade
Qualquer dado recebido precisa ser comprovado que é o mesmo de quando enviado.
- Usasse hash na criptografia.

## Prova de Origem
Autenticação do usuário que enviou, garantindo que a assinatura não é falsa. 
Um método comum é fazer a assinatura com uma chave privada.

## Processo de Assinatura Digital
- Enviasse o arquivo desejado;
- O arquivo é associado a uma hash;
- A hash é criptografada com a chave privada do remetente, criando a assinatura digital;
- O destinatário recebe tanto o arquivo quanto a assinatura digital;
- O destinatário usa a chave pública do remetente para descriptografar a assinatura digital, obtendo a hash enviada;
- O destinatário "recria" a hash do arquivo recebido e compara com a descriptografada pela assinatura digital.
