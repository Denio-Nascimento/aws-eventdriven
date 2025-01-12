# **Laboratório Serverless AWS: Validação e Roteamento de Pedidos** 

Este laboratório é a **segunda parte** do fluxo de processamento de pedidos. Supondo que já exista uma Lambda anterior que insere pedidos em uma fila SQS FIFO (chamada `sqs-pedidos-validos.fifo`), agora vamos:

1. **Criar a Lambda de Validação** que:
   - Lê pedidos da fila `sqs-pedidos-validos.fifo`.
   - Valida dados obrigatórios.
   - Se válidos, envia evento ao **Amazon EventBridge**.
   - Se inválidos, envia alerta via **SNS**.

2. **Criar e Configurar o Amazon SNS** para notificar sobre pedidos inválidos.

3. **Criar um Event Bus** no **Amazon EventBridge** e **3 Regras** de roteamento, cada uma para um status de pedido:
   - `Pendente`  
   - `Alterar Pedido`  
   - `Cancela Pedido`  

4. **Criar Filas SQS** (FIFO) de destino para cada status, cada uma com sua **DLQ** configurada.

5. **Testar** o fluxo tanto diretamente via console (evento de teste), quanto colocando mensagens na fila SQS ou enviando pedidos pela Lambda anterior (opcional).

---

## **Etapa 1: Criar a Lambda de Validação e Envio**

### 1.1. Acessar o AWS Lambda
1. No menu de serviços da AWS, procure por **Lambda**.
2. Clique em **Create function**.

### 1.2. Criar a Função
1. **Function name:** `validation-and-send-lambda`
2. **Runtime:** escolha **Python 3.13** (ou versão mais recente disponível).
3. Clique em **Create function**.

### 1.3. Inserir o Código-Fonte
Na página da função, role até **Code source**:

1. Apague o conteúdo padrão do editor.
2. Cole o seguinte código:

```python
import boto3
import json
import os
import logging

# Importa a função de validação do Layer (ajuste para o nome correto do seu arquivo/módulo)
from order_validation_layer import validate_order 

logger = logging.getLogger()
logger.setLevel(logging.INFO)

eventbridge_client = boto3.client('events')
sns_client = boto3.client('sns')

EVENT_BUS_NAME = os.getenv('EVENT_BUS_NAME')  # ex: 'event-bus-pedidos'
SNS_TOPIC_ARN = os.getenv('SNS_TOPIC_ARN')   # ex: 'arn:aws:sns:us-east-1:123456789012:sns-notificacoes-erros'

def lambda_handler(event, context):
    """
    Lê pedidos de uma fila SQS (evento disparado pelo SQS),
    valida cada pedido e envia para EventBridge ou SNS (caso inválido).
    """
    for record in event['Records']:
        try:
            body = json.loads(record['body'])
            logger.info(f"Validando pedido: {body.get('order_id', 'sem-id')}")

            if validate_order(body):
                logger.info(f"Pedido válido: {body['order_id']} com status {body['order_status']}")
                send_to_eventbridge(body)
            else:
                logger.warning(f"Pedido inválido: {body['order_id']}. Enviando alerta para SNS.")
                send_to_sns_alert(body, "Pedido inválido")

        except Exception as e:
            logger.error(f"Erro ao processar pedido {body.get('order_id', 'desconhecido')}: {str(e)}", exc_info=True)

def send_to_eventbridge(order):
    """
    Envia o pedido validado ao EventBridge, que irá rotear conforme o order_status.
    """
    try:
        event = {
            "Source": "socket-entregas.orders",
            "DetailType": "OrderEvent",
            "Detail": json.dumps(order),
            "EventBusName": EVENT_BUS_NAME
        }
        eventbridge_client.put_events(Entries=[event])
        logger.info(f"Pedido {order['order_id']} enviado ao EventBridge com status {order['order_status']}")
    except Exception as e:
        logger.error(f"Erro ao enviar pedido {order['order_id']} para EventBridge: {str(e)}")

def send_to_sns_alert(order, message):
    """
    Envia um alerta para o SNS quando o pedido é inválido.
    """
    try:
        alert_message = {
            "order_id": order.get('order_id'),
            "status": order.get('order_status'),
            "message": message
        }
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=json.dumps(alert_message),
            Subject="Alerta de Pedido com Problema"
        )
        logger.info(f"Alerta enviado para SNS para o pedido {order['order_id']}")
    except Exception as e:
        logger.error(f"Erro ao enviar alerta SNS: {str(e)}")
```

3. Clique em **Deploy** para salvar as alterações.

### 1.4. Adicionar Variáveis de Ambiente
1. Vá em **Configuration** > **Environment variables**.
2. Clique em **Edit** e depois em **Add environment variable**:
   - **Key**: `EVENT_BUS_NAME`
   - **Value**: `event-bus-pedidos` (ou o nome do seu Event Bus)
3. Repita para:
   - **Key**: `SNS_TOPIC_ARN`
   - **Value**: ARN do seu tópico SNS (por exemplo, `arn:aws:sns:us-east-1:123456789012:sns-notificacoes-erros`)
4. Clique em **Save**.

### 1.5. Ajustar Timeout (Opcional)
1. Ainda em **Configuration** > **General configuration**.
2. Clique em **Edit** e ajuste **Timeout** (ex: 30 s).
3. Clique em **Save**.

---

## **Etapa 2: Permissões IAM da Lambda**

### 2.1. Acessar a Role
1. Na página da função Lambda, vá em **Configuration** > **Permissions**.
2. Clique no nome da role (algo como `validation-and-send-lambda-role-xyz`), abrindo o console do IAM.

### 2.2. Adicionar Política Inline para SQS (invocada)
Para que a Lambda possa receber mensagens da fila, ela normalmente só precisa de permissão pass-through do SQS. Se for **SQS como trigger**, a permissão principal é dada pelo serviço. Porém, caso seja necessário explicitamente (por exemplo, a Lambda possa deletar mensagens, etc.), você adiciona algo como:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:sqs-pedidos-validos.fifo"
    }
  ]
}
```
*(Ajuste Região e ARN da fila conforme sua conta.)*

### 2.3. Adicionar Política para EventBridge
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:123456789012:event-bus/event-bus-pedidos"
    }
  ]
}
```
*(Novamente, ajuste Região, Conta e nome do Event Bus.)*

### 2.4. Adicionar Política para SNS
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:sns-notificacoes-erros"
    }
  ]
}
```
Clique em **Review**, dê um nome como `LambdaSendEventBridgeAndSNSPolicy`, e crie.

---

## **Etapa 3: Criar o SNS para Notificações**

### 3.1. Criar o Tópico SNS
1. Vá em **Amazon SNS** > **Topics** > **Create topic**.
2. **Type:** Standard (ou FIFO, dependendo da sua necessidade).
3. **Name:** `sns-notificacoes-erros`
4. Clique em **Create topic**.

### 3.2. Criar uma Assinatura (Opcional)
1. Clique em **Create subscription**.
2. Em **Protocol**, escolha **Email**.
3. Em **Endpoint**, digite seu email.
4. Clique em **Create subscription**.
5. Confirme o email que receber.

---

## **Etapa 4: Criar o EventBridge Bus e Regras**

### 4.1. Criar um Event Bus
1. No console da AWS, vá em **Amazon EventBridge**.
2. Em **Event buses** > **Create event bus**.
3. **Name:** `event-bus-pedidos`.
4. Clique em **Create**.

### 4.2. Criar 3 Regras de Roteamento
Precisamos rotear de acordo com `order_status`, conforme o código do Lambda.

#### 4.2.1. Regra Pendente
1. **Name:** `regra-pedido-pendente`
2. **Event Bus:** `event-bus-pedidos`
3. **Event pattern (JSON)**:
   ```json
   {
     "source": ["socket-entregas.orders"],
     "detail-type": ["OrderEvent"],
     "detail": {
       "order_status": ["Pendente"]
     }
   }
   ```
4. **Target:** SQS — Selecione a fila `sqs-pedido-pendente.fifo`
5. **MessageGroupId:** `orders-group` (obrigatório para FIFO)
6. **Create rule**.

#### 4.2.2. Regra Alterar Pedido
- **Name:** `regra-pedido-alterar`
- **Event pattern (JSON)**:
  ```json
  {
    "source": ["socket-entregas.orders"],
    "detail-type": ["OrderEvent"],
    "detail": {
      "order_status": ["Alterar Pedido"]
    }
  }
  ```
- **Target:** `sqs-pedido-alterado.fifo`
- **MessageGroupId:** `orders-group`

#### 4.2.3. Regra Cancela Pedido
- **Name:** `regra-pedido-cancela`
- **Event pattern (JSON)**:
  ```json
  {
    "source": ["socket-entregas.orders"],
    "detail-type": ["OrderEvent"],
    "detail": {
      "order_status": ["Cancela Pedido"]
    }
  }
  ```
- **Target:** `sqs-pedido-cancelado.fifo`
- **MessageGroupId:** `orders-group`

---

## **Etapa 5: Criar Filas SQS de Destino e DLQs**

Cada status precisa de sua **fila FIFO**. Se quisermos alta confiabilidade, também criamos **DLQs** para cada fila. Abaixo, um exemplo para a fila Pendente:

1. Vá em **Amazon SQS** > **Create queue**.
2. **FIFO Queue**.
3. **Queue name:** `sqs-pedido-pendente.fifo`
4. Marque **Enable high throughput FIFO** se desejar.
5. **Content-based deduplication:** Ativado.
6. **Dead-letter queue:** selecione `sqs-pedido-pendente-dlq.fifo` (também FIFO).
7. **MaxReceiveCount:** 3.
8. Repita o processo para:
   - `sqs-pedido-alterado.fifo` / `sqs-pedido-alterado-dlq.fifo`
   - `sqs-pedido-cancelado.fifo` / `sqs-pedido-cancelado-dlq.fifo`

---

## **Etapa 6: Teste Manual no Console (Lambda)**

Para testar sua **Lambda** de forma isolada:

1. Volte para a função `validation-and-send-lambda`.
2. Clique em **Test** no topo.
3. Clique em **Configure test event** e insira um JSON que simule uma mensagem SQS:
   ```json
   {
     "Records": [
       {
         "messageId": "1",
         "receiptHandle": "abc",
         "body": "{\"order_id\":\"SM-19244\",\"order_status\":\"Pendente\",\"company\":{\"name\":\"Acme Corp\"},\"customer\":{}}",
         "attributes": {},
         "messageAttributes": {},
         "md5OfBody": "",
         "eventSource": "aws:sqs",
         "eventSourceARN": "arn:aws:sqs:us-east-1:123456789012:sqs-pedidos-validos.fifo",
         "awsRegion": "us-east-1"
       }
     ]
   }
   ```
4. Clique em **Save** e depois em **Test**.
5. Verifique os **Logs** em **CloudWatch** para confirmar se o pedido foi validado e roteado corretamente.

### Verificando o Roteamento
- Se `order_status = "Pendente"`, verifique se a mensagem chegou na fila `sqs-pedido-pendente.fifo`.
- Se for inválido, verifique no **CloudWatch** se houve envio de alerta ao **SNS**.

---

## **Etapa 7: Validar o Fluxo Completo**

Para um **teste de ponta a ponta** (opcional), use a **Primeira Lambda** (que lê do S3 e envia para `sqs-pedidos-validos.fifo`). Assim que a mensagem chegar nessa fila:
1. Ela aciona automaticamente a `validation-and-send-lambda`.
2. Se válido, o **EventBridge** roteia conforme o status.
3. Aparecerá na fila `sqs-pedido-pendente.fifo`, `sqs-pedido-alterado.fifo` ou `sqs-pedido-cancelado.fifo`.

---

## **Conclusão**

Neste documento, configuramos:

- **Lambda** que valida pedidos e envia para **EventBridge** ou **SNS** (em caso de erro).
- **SNS** para alertas.
- **EventBridge** com 3 regras de roteamento, cada qual ligada a uma fila FIFO.
- Filas SQS com suas DLQs, para isolar e rastrear falhas.

Essa arquitetura **Event Driven** é escalável, desacoplada e facilita o controle de diferentes fluxos de pedidos (novos, alterados, cancelados) em paralelo.
