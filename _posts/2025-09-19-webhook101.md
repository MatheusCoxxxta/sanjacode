---
title: Levando webhooks a sério
date: 2025-09-18 23:37:28
tags:
---
Webhooks, postback, notificação reversa entre sistemas, muitos nomes que já ouvi pra essas APIs que são destinadas a notificar ações para sistemas parceiros, ou receber notificações de terceiros. 

Toda empresa que passei, em algum momento lidei com webhook, muitas vezes no contexto de pagamento: receber notificação de venda por Pix e receber notificações sobre assinaturas gerenciadas por um Gateway são atividades muitos comuns de quase todo SaaS, e nem todo dev trata com o devido cuidado.

# Estrutura base

Talvez o maior ponto de um webhook seja sua resiliencia perante um cenário muito incerto: outra aplicação está disparando ações na nossa aplicação se baseando apenas em contrato previamento firmado. Os dados podem vir via URL ou Body da requisição HTTP, os dados devem ser validados antes de prosseguir para uma operação, e um status code de sucesso vai ser sinal verde para a outra aplicação parar de disparar aquela mensagem.

Requisição -> Validação de dados -> Processamento -> Retorno positivo ou negativo (falha)

# Problemas frequentes

Vamos trabalhar separando em dois tipos de problema, o conceitual e o critico. No conceitual vemos mudanças que podem melhorar nossos webhooks, torna-los mais simples para sistemas externos, e até mesmo definir algumas ideias. No critico vamos falar de pontos que irão gerar problemas financeiros e de conciliação pra sua operação.

## Conceitual

Acredito que o primeiro problema seja tratar como uma porta comum de sistema, uma simples API. Não é o caso, webhooks não precisam seguir RESTful como se fosse uma comunicação externa do sistema. 

Sistemas externos vão interpretar os status code do seu webhooks em três categorias:
- 200: sucesso, tudo conforme o esperado.
- 401: alguma chave de autenticação, token ou qualquer camada de segurança não está de acordo.
- 400: dados malformatados - dados que estão em formato inválido, operação inválida decorrente de algo fora do esperado.
- 500+: falhas do lado do nosso sistema

Quando não encontramos um usuário em tentiva de execução de uma API do nosso sistema, retornamos como boa pratica um status 404, aquela entidade não foi encontrada. No webhook, um cenário parecido pode ser interpretado de outra forma: se o usuário não foi encontrado e processamos apenas usuários válidos naquela API, podemos interpretar como uma requisição malformatada, já que os dados não atendem aos critérios de validação.

## Critico

Aqui vão estar as race conditions, consumo de recurso em execuções duplicadas, gargalos de escalabilidade e roubo de recurso de peças que não podem processar em background.

### Race conditions

Sempre usamos o exemplo de duas operações acontecendo ao mesmo tempo em uma conta bancaria, dois saques.

Vou usar um exemplo mais simples dessa vez: se você não alterou nada nas regras padrões de LOCK do seu banco de daods, e um webhook chegar triplicado na aplicação, o que ocorre?

Se seu webhook pega o dado e já processa, provavelmente sua validação de duplicidade de dados vai falhar, antes dos dados da primeira request ser requesição para validarmos se ele já foi processado, o segunda requesição já vai ter passado pela checagem de duplicidade. 

Pode acontecer exatamente o mesmo com a terceira requisição. Mesmo com a checagem de duplicidade no seu código-fonte, o dado chegou triplicado ao banco de dados, como o banco estava com as regras padrões de LOCK, ele nunca esteve travado durante as execuções. Provavelmente, nesse momento, você tem três vezes o mesmo registro no banco.