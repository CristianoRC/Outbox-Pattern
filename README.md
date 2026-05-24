# Outbox Pattern

Documentação com exemplos na prática de como funciona o Outbox Pattern em microsserviço. 

## O que é o Outbox Pattern


### Qual problema ele resolve

Também conhecido como Transactional Outbox Pattern, é usado em situações onde você precisa garantir a transação de uma operação, salvar no banco e enviar os eventos. Imagina que uma transação chegou na sua conta digital, você já subtraiu o saldo da pessoa no banco de dados, mas acaba tendo problemas ao enviar a mensagem para o seu serviço de mensageria, informando que o esse valor deveria ser enviado para outra conta externa, por conta de instabilidade, ele estar fora do ar... . Como você lida com isso? Quais eventos já foram enviados, quais precisaremos reenviar?

<img src="./images/error.png" width="800"/>

É impossível fazer uma transação entre um banco de dados e um message broker, para resolver esse problema a gente poderia fazer de duas formas:

- Se der erro ao enviar a mensagem para o tópico, a gente desfaz o passo anterior, só que isso tem alguns impactos, muitas vezes o processo de desfazer é mais demorado, e precisaria ser feito de forma assíncrona(mas não temos mensageria para isso kkk), o outro é que por motivos 100% técnico tu podes perder de fazer milhares de vendas ou transações bancárias, isso não é legal.

- Outra forma de garantir isso é com o Outbox Pattern, que vamos entrar em detalhes a seguir, e vamos entender o porquê é uma solução melhor do que a anterior.


### Como ele funciona

A grande maioria dos bancos de dados conseguem dar garantia de uma transação, que basicamente é executar mais de uma ação no banco de dados como se fosse uma só, aí todos funcionam ou nenhuma funciona, e já que não dá para fazer esse fluxo de transação entre um banco e um message broker como expliquei anteriormente, o Outbox Pattern tem como ideia principal de em uma única transação para o banco de dados você salvar a ação, e o evento relacionado a ação, e depois vem um outro serviço que roda em background que pega a mensagem de evento do banco e coloca no message broker, dessa forma você nunca vai perder um evento, e vai ter a garantia se ele já foi enviado ou não.


<img src="./images/outbox.png" width="800"/>

#### Estrutura de uma tabela de eventos

Existem duas abordagens, uma de criar uma tabela genérica, com os dados que tem que ter no evento, muitas das vezes uma coluna JSON se tiverem usando um banco relacional, e outra é ter uma tabela específica para cada tipo de evento, mas todas elas tem alguns pontos em comum.

- Id
- Chave de Idempotência
- Já foi processado
- Data do processamento
- Mensagem que deve ser enviada no evento

Em tabelas genéricas você pode ter coisas como

- Qual tópico/fila esse evento deve ser enviado
- Qual o tipo desse evento(transação, saque...)

Já foi processado e Data do processamento podem se tornar uma coisa só, essa é a coluna que garante que a mensagem já foi enviada ou não, e quando ela foi enviada.

<img src="./images/table.png" width="600"/>


#### Possíveis problemas

Mas um dos possíveis questionamentos é em relação a esse serviço que adiciona mensagem no tópico, ele pode dar problema novamente! Se você salvar no banco que foi enviado e deu problema para enviar teria que dar um rollback, sim, aí seria um problema parecido do inicial, mas, mais simples de resolver. Mas, na prática, o ideal é enviar a mensagem e depois salvar no banco de dados que foi feito com sucesso. Você pode até argumentar que o banco de dados pode estar fora do ar depois que a mensagem foi enviada, e ela seria enviada duas vezes, sim, esse é um problema, mas conseguimos resolver ele de uma forma simples, que é criar consumidores idempotentes, da uma olhada nesse vídeo que eu falo um pouco mais sobre o assunto.
[[Arquitetura] O que são chaves de idempotência?](https://youtu.be/U0DyJx68oCY)

### CDC - Change Data Capture

São conceitos muito parecidos (e muitas vezes usados em conjunto). A grande diferença é que o CDC funciona lendo o log de transações do próprio banco (como o WAL no Postgres ou o binlog no MySQL), capturando as mudanças direto da fonte, sem precisar de uma tabela extra de eventos como no Outbox. Por outro lado, o CDC normalmente exige uma infra adicional pra rodar, como Debezium e Kafka Connect, enquanto o Outbox vive dentro do banco da própria aplicação, o que costuma ser mais simples operacionalmente, mesmo tendo que lidar com a tabela extra de eventos. O conceito de CDC é algo que nunca utilizei, então também indico dar uma olhada, principalmente no Debezium, uma das ferramentas mais usadas no mercado para isso.

## Inbox Pattern

Ele tem uma ideia bem parecida com o outbox, mas eles podem ser usados de formas totalmente diferentes.

A sua ideia principal é **garantir o processamento idempotente dos eventos recebidos**, evitando duplicação. Como a maioria dos message brokers entrega mensagens com semântica *at-least-once* (pelo menos uma vez), é comum que o mesmo evento chegue mais de uma vez, seja por retry, falha no ACK ou rebalanceamento de consumidores.

Para resolver isso, a gente salva o evento (ou pelo menos sua chave de idempotência) no banco assim que ele chega, e só processa se ainda não tiver sido processado.

Como efeito colateral, você também ganha durabilidade, ou seja, não perde eventos mesmo se o serviço cair antes de processar, e ganha também a possibilidade de reprocessar quando precisar.

Além da idempotência, o padrão também ajuda em outros cenários: processos longos com muitos passos, onde você quer desacoplar a recepção do processamento, dando ACK rápido pro broker; necessidade de auditoria e registro dos eventos recebidos; ou quando o evento precisa ser executado em uma ordem específica que uma priority queue não resolve.

<img src="./images/inbox.png" width="800"/>



Obs.: O exemplo usado não é 100% real, foram retirados alguns conceitos para facilitar a explicação*