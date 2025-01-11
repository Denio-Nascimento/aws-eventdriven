# **Laboratório Serverless AWS: Entrada de Pedido pelo Data Lake

Este laboratório tem como objetivo ensinar a criação de uma arquitetura **Event Driven - Event-driven Architecture (EDA)**  na AWS para processar pedidos de venda a partir de arquivos JSON armazenados no Amazon S3. Durante esta atividade prática, você aprenderá a implementar uma solução com validação de pedidos, envio de eventos ao EventBridge, registro de dados no DynamoDB e tratamento de erros com SQS.

A arquitetura proposta é altamente escalável e permite o uso de **Lambda Layers** para reaproveitamento de código, seguindo as melhores práticas de modularidade e manutenção. Ao final do laboratório, você terá uma aplicação que valida os pedidos, identifica erros e envia notificações para monitoramento.

---

## **O que Você Irá Aprender**

Nesta atividade, você irá:

- Criar uma fila no Amazon SQS para receber pedidos.
- Criar uma tabela no Amazon DynamoDB para registrar arquivos e pedidos processados.
- Criar um barramento de eventos no Amazon EventBridge.
- Criar Regras do EventBridge
- Criar uma **AWS Lambda Layer** para validação dos pedidos.
- Criar uma função AWS Lambda para pré-validação dos pedidos JSON utilizando a layer.
- Configurar a função Lambda com as permissões IAM adequadas e variáveis de ambiente.
- Configurar o S3 Notification para  enviar eventos para o Amazon SQS.
- Realizar testes de validação e envio ao EventBridge e SQS.

---

### **Pré-requisitos**

- **Conta AWS ativa** com permissões para criar recursos (S3, SQS, SNS, Lambda, EventBridge e IAM).
- **Acesso ao console da AWS** via navegador web.


### **Componentes da Arquitetura:**

1. **Lambda de Extração e Controle de Duplicação:**
   - Responsável por ler o arquivo JSON enviado ao S3.
   - Faz o registro de arquivos processados no DynamoDB para evitar duplicações.
   - Envia pedidos individuais para a fila SQS FIFO para processamento.

2. **Lambda de Validação:**
   - Lê os pedidos da SQS FIFO, valida os dados obrigatórios e verifica o status do pedido.
   - Encaminha os pedidos válidos ao EventBridge, onde são roteados para filas SQS específicas de acordo com o status do pedido.
   - Em caso de erro, envia os pedidos inválidos para uma fila DLQ FIFO e notifica via SNS.

---

## **Etapa 1: Criar o Bucket S3 e o Prefixo para os Pedidos**

1. **Acessar o Amazon S3:**

   - Faça login na console da AWS.
   - No menu de serviços, selecione **S3** (pode usar a barra de pesquisa).
   - **AWS Region:** Certifique-se de estar na região em que pretende trabalhar. (por exemplo, **us-east-1**).

2. **Criar um Bucket:**

   - Clique em **Create bucket**.
   - **Bucket name:** `translogistica-pedidos` (os nomes de bucket devem ser exclusivos globalmente; se esse nome não estiver disponível, escolha outro nome exclusivo, como `translogistica-pedidos-seu-nome`).
   - Mantenha as demais configurações padrão.
   - Clique em **Create bucket**.

3. **Criar o Prefixo (Pasta) "novos-pedidos/":**

   - Clique no nome do bucket criado para acessá-lo.
   - Clique em **Create folder**.
   - **Folder name:** `novos-pedidos`
   - Clique em **Create folder**.

---

## **Etapa 2: Criar as Filas SQS**

### **2.1. Fila Standard: `sqs-pedidos-json`**
Esta fila é utilizada para armazenar a notificação de novos arquivos JSON carregados no bucket S3.

1. Acesse o console da **AWS** e selecione **SQS**.
2. Clique em **Create queue**.
3. Escolha **Standard Queue**.
4. **Queue Name:** `sqs-pedidos-json`.
5. Clique em **Create Queue**.

### **2.2. Fila DLQ FIFO: `sqs-pedidos-dlq.fifo`**
Esta fila FIFO armazena pedidos que não puderam ser processados após o número máximo de tentativas.

1. Crie uma fila **FIFO** com o nome `sqs-pedidos-dlq.fifo`.
2. Habilite **Content-based deduplication**.

### **2.3. Fila FIFO: `sqs-pedidos-validos.fifo`**
Esta fila FIFO recebe pedidos individuais extraídos do arquivo JSON.

1. Clique em **Create queue**.
2. Escolha **FIFO Queue**.
3. **Queue Name:** `sqs-pedidos-validos.fifo`.
4. Marque **Content-based deduplication** para evitar envios duplicados.
5. Configure uma DLQ FIFO para tratar mensagens que falham:
   - Em **Dead-letter queue:** Clica em **Enabled**
   - **Choose queue:** `sqs-pedidos-dlq.fifo`.
   - **MaxReceiveCount:** 3 (número máximo de tentativas).
6. Clique em **Create Queue**.

### **2.3. Fila DLQ FIFO: `sqs-pedidos-dlq.fifo`**
Esta fila FIFO armazena pedidos que não puderam ser processados após o número máximo de tentativas.

### **2.4. Filas por Status de Pedido e suas DLQs**
Para garantir o encaminhamento correto dos pedidos de acordo com o status, crie uma fila FIFO e sua respectiva DLQ FIFO para cada tipo de status de pedido:

  Repita os procedimento dos tópicos 2.2 e 2.3

- **`sqs-pedido-novo.fifo`**: recebe pedidos com status "Novo".
  - **DLQ:** `sqs-pedido-novo-dlq.fifo`.
- **`sqs-pedido-alterado.fifo`**: recebe pedidos com status "Alterado".
  - **DLQ:** `sqs-pedido-alterado-dlq.fifo`.
- **`sqs-pedido-cancelado.fifo`**: recebe pedidos com status "Cancelado".
  - **DLQ:** `sqs-pedido-cancelado-dlq.fifo`.

Essas filas permitem separar o fluxo de processamento e podem ser usadas para diferentes ações ou serviços específicos para cada status.

---

## **Etapa 3: Criar Tabela no DynamoDB**

1. **Acessar o Amazon DynamoDB:**
   - No menu de serviços, selecione **DynamoDB**.

2. **Criar uma nova tabela:**
   - Clique em **Create Table**.

3. **Configurar a tabela:**
   - **Table Name:** `OrdersTable`.
   - **Partition Key:** `PK` (Tipo: `String`).
   - **Sort Key:** `SK` (Tipo: `String`).

4. **Finalizar a criação:**
   - Clique em **Create Table**.

---

## **Etapa 4: Criar o Barramento de Eventos no EventBridge**

1. **Acessar o Amazon EventBridge:**
   - No menu de serviços, selecione **EventBridge**.

2. **Criar um novo Event Bus:**
   - No menu a esqueda clique em **Event buses**
   - Clique em **Create Event Bus**.
   - **Name:** `event-bus-pedidos`.

4. **Finalizar a criação:**
   - Clique em **Create**.

---

## **Etapa 4: Criar as Regras do EventBridge**
As regras do EventBridge definem o roteamento dos eventos de pedidos conforme o status.

### **Passos Detalhados para Criar uma Regra no EventBridge**

#### **1. Regra `regra-pedido-novo`**
**Objetivo:** Roteia os pedidos com status "Novo" para a fila SQS `sqs-pedido-novo.fifo`.

**Passos detalhados:**
1. Acesse o console **EventBridge**.
2. No painel lateral esquerdo, clique em **Rules** (Regras).
3. Clique em **Create rule** (Criar regra).
4. **Name:** `regra-pedido-novo`.
5. **Description:** "Roteia pedidos novos para a fila `sqs-pedido-novo.fifo`".
6. Em **Event bus**, selecione `event-bus-pedidos`.
7. **Rule type:** escolha **Event pattern**.
8. Clique em **Next** (Próximo).

**Definir o Padrão de Evento:**
1. Em **Event Source**, selecione **Custom event bus**.
2. Em **Event pattern**, insira o seguinte JSON:
   ```json
   {
     "detail-type": ["PedidoNovo"]
   }
   ```

**Configurar Destino:**
1. Em **Select target**, selecione **SQS queue**.
2. Em **Queue**, selecione `sqs-pedido-novo.fifo`.
3. No campo **Message group ID*** (obrigatório), insira um valor, por exemplo: `orders-group`.
4. Clique em **Next** (Próximo).

**Revisão e Criação:**
1. Revise as informações inseridas.
2. Clique em **Create rule** (Criar regra).

---

#### **2. Regra `regra-pedido-alterado`**
**Objetivo:** Roteia os pedidos com status "Alterado" para a fila SQS `sqs-pedido-alterado.fifo`.

**Passos detalhados:**
1. **Name:** `regra-pedido-alterado`.
2. **Description:** "Roteia pedidos alterados para a fila `sqs-pedido-alterado.fifo`".
3. **Event pattern:**
   ```json
   {
     "detail-type": ["PedidoAlterado"]
   }
   ```
4. **Destino:** selecione `sqs-pedido-alterado.fifo`.

---

#### **3. Regra `regra-pedido-cancelado`**
**Objetivo:** Roteia os pedidos com status "Cancelado" para a fila SQS `sqs-pedido-cancelado.fifo`.

**Passos detalhados:**
1. **Name:** `regra-pedido-cancelado`.
2. **Description:** "Roteia pedidos cancelados para a fila `sqs-pedido-cancelado.fifo`".
3. **Event pattern:**
   ```json
   {
     "detail-type": ["PedidoCancelado"]
   }
   ```
4. **Destino:** selecione `sqs-pedido-cancelado.fifo`.

---

---

## **Etapa 6: Criar a Layer de Validação**

### **6.1. Código da Layer (`validation.py`)**
Esta layer contém a função responsável por validar os campos obrigatórios do pedido.

~~~python
def validate_order(order):
    required_fields = ["order_id", "customer", "items", "payment", "company", "order_status"]
    for field in required_fields:
        if field not in order:
            return False
    return order["order_status"] in ["PedidoNovo", "PedidoAlterado", "PedidoCancelado"]
~~~

### **6.2. Compactação e Upload**
1. Crie a pasta `python` e mova `validation.py` para dentro.
2. No terminal:
~~~bash
zip -r validation_layer.zip python/
~~~
3. Acesse **AWS Lambda > Layers** > **Create layer**.
4. Faça upload do arquivo ZIP, nomeie como `order-validation-layer` e selecione `Python 3.x`.

---

## **Etapa 7: Configurar SNS**
O SNS é utilizado para enviar notificações por e-mail quando um pedido falha na validação.

1. Acesse **SNS** > **Create topic** > **FIFO**.
2. Nomeie como `sns-notificacoes-erros`.
3. Configure uma assinatura de e-mail.
4. Confirme o recebimento.


---


## **Etapa 8: Criar as Funções Lambda**

### **8.1. Lambda de Extração e Controle de Duplicidade**
Esta função lê o arquivo JSON do S3, registra os pedidos no DynamoDB para evitar duplicação e envia os pedidos para a fila FIFO.

#### **Passos:**
1. Acesse **AWS Lambda** > **Create function**.
2. **Function name:** `extract-and-send-lambda`.
3. **Runtime:** `Python 3.x`.
4. **Timeout:** 2 minutos.
5. **Memory:** 256 MB.
6. Vincule a layer `order-validation-layer`.
7. Adicione variáveis de ambiente:
   - **`DYNAMO_TABLE_NAME`**: `OrdersTable`
   - **`SQS_FIFO_URL`**: URL da fila FIFO.

#### **Código:**
~~~python
import boto3, json, os, logging
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)

dynamo_client = boto3.resource('dynamodb')
sqs_client = boto3.client('sqs')
s3_client = boto3.client('s3')
DYNAMO_TABLE_NAME = os.getenv('DYNAMO_TABLE_NAME')
SQS_FIFO_URL = os.getenv('SQS_FIFO_URL')

def lambda_handler(event, context):
    for record in event['Records']:
        bucket, key = record['s3']['bucket']['name'], record['s3']['object']['key']
        response = s3_client.get_object(Bucket=bucket, Key=key)
        orders = json.loads(response['Body'].read().decode('utf-8'))

        if check_file_processed(key):
            logger.info(f"Arquivo {key} já processado.")
            return {"statusCode": 200, "body": "Arquivo duplicado ignorado"}

        register_file_in_dynamodb(key)
        for order in orders:
            send_to_fifo(order)

def check_file_processed(file_name):
    table = dynamo_client.Table(DYNAMO_TABLE_NAME)
    return 'Item' in table.get_item(Key={"PK": f"FILE#{file_name}", "SK": "SUMMARY"})

def register_file_in_dynamodb(file_name):
    table = dynamo_client.Table(DYNAMO_TABLE_NAME)
    table.put_item(Item={"PK": f"FILE#{file_name}", "SK": "SUMMARY", "ProcessedAt": datetime.utcnow().isoformat()})

def send_to_fifo(order):
    sqs_client.send_message(QueueUrl=SQS_FIFO_URL, MessageBody=json.dumps(order), MessageGroupId="orders-group")
~~~

#### **Política IAM (JSON)**
Adicione a seguinte política ao role da Lambda de Extração:

1. No painel da função Lambda, clique em **Configuration** > **Permissions**.
2. Clique no nome da **role** atribuída à função.
3. Clique em **Add permissions** > **Create inline policy**.
4. Selecione a aba **JSON** e cole a seguinte política com mínimos privilégios:

~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::SEU_BUCKET/*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem", "dynamodb:GetItem"],
      "Resource": "arn:aws:dynamodb:REGIÃO:ID_DA_CONTA:table/OrdersTable"
    },
    {
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:REGIÃO:ID_DA_CONTA:sqs-pedidos-validos.fifo"
    }
  ]
}
~~~



### **8.2. Lambda de Validação**
Esta função valida o conteúdo do pedido e encaminha os eventos para o EventBridge com base no status.

#### **Passos:**
1. **Function name:** `validation-and-send-lambda`.
2. **Timeout:** 2 minutos.
3. **Memory:** 256 MB.
4. Adicione variáveis de ambiente:
   - **`EVENT_BUS_NAME`**: `event-bus-pedidos`.
   - **`SNS_TOPIC_ARN`**: ARN do tópico SNS.
   - **`DLQ_URL`**: URL da DLQ FIFO.

#### **Código:**
~~~python
import boto3, json, os, logging
from validation import validate_order

logger = logging.getLogger()
logger.setLevel(logging.INFO)

eventbridge_client = boto3.client('events')
sns_client = boto3.client('sns')
sqs_client = boto3.client('sqs')
EVENT_BUS_NAME = os.getenv('EVENT_BUS_NAME')
SNS_TOPIC_ARN = os.getenv('SNS_TOPIC_ARN')
DLQ_URL = os.getenv('DLQ_URL')

def lambda_handler(event, context):
    for record in event['Records']:
        order = json.loads(record['body'])
        if not validate_order(order):
            send_to_dlq_and_notify(order)
            continue
        send_to_eventbridge(order)

def send_to_dlq_and_notify(order):
    sqs_client.send_message(QueueUrl=DLQ_URL, MessageBody=json.dumps(order))
    sns_client.publish(TopicArn=SNS_TOPIC_ARN, Message=f"Pedido inválido: {order['order_id']}")

def send_to_eventbridge(order):
    eventbridge_client.put_events(Entries=[{"Source": "app.pedidos", "DetailType": order['order_status'], "Detail": json.dumps(order), "EventBusName": EVENT_BUS_NAME}])
~~~

#### **Política IAM (JSON)**

1. No painel da função Lambda, clique em **Configuration** > **Permissions**.
2. Clique no nome da **role** atribuída à função.
3. Clique em **Add permissions** > **Create inline policy**.
4. Selecione a aba **JSON** e cole a seguinte política com mínimos privilégios:
~~~json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sqs:ReceiveMessage",
      "Resource": "arn:aws:sqs:REGIÃO:ID_DA_CONTA:sqs-pedidos-validos.fifo"
    },
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:REGIÃO:ID_DA_CONTA:event-bus/event-bus-pedidos"
    },
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:REGIÃO:ID_DA_CONTA:sns-notificacoes-erros"
    },
    {
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:REGIÃO:ID_DA_CONTA:sqs-pedidos-dlq.fifo"
    }
  ]
}
~~~


---

## **Etapa 9: Teste do Fluxo Completo**

1. Envie um arquivo JSON para o bucket S3.
2. Verifique os logs no **CloudWatch**.
3. Confira as mensagens nas filas FIFO e DLQ.
4. Confirme os eventos no **EventBridge**.
5. Verifique as notificações via **SNS**.

---

## **Conclusão**
Essa atualização melhora a resiliência e modularidade do processamento de pedidos, com explicações detalhadas e um fluxo bem estruturado que separa cada responsabilidade de forma clara.

