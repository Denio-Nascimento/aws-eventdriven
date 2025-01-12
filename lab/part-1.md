# Laboratório Serverless AWS: Entrada de Pedido pelo Data Lake
Este laboratório tem como objetivo ensinar a criação de uma arquitetura Event Driven - Event-driven Architecture (EDA) na AWS para processar pedidos de venda a partir de arquivos JSON armazenados no Amazon S3 e enviados pelo Amazon API Gateway.

O laboratório está dividido em quatro partes, organizadas em quatro arquivos que devem ser seguidos na ordem, pois cada um complementa o anterior.

A primeira parte (que se trata desse arquivo) ensina a desenvolver uma função AWS Lambda que é responsável por receber mensagens de uma fila SQS padrão com informações de eventos de S3 ao serem criados novos arquivos. A função Lambda lê o evento do S3, faz o download do arquivo, processa os pedidos e registra as informações no DynamoDB sobre os arquivos e pedidos processados para evitar indempotência. Além disso, envia os pedidos para uma fila SQS FIFO `sqs-pedidos-validos.fifo`.

A segunda parte do fluxo de processamento de pedidos cria uma função Lambda de validação que lê mensagens da fila `sqs-pedidos-validos.fifo`, valida os dados e, dependendo do resultado, envia eventos para o Amazon EventBridge ou alertas para o Amazon SNS. Também serão configurados um Event Bus com três regras de roteamento para diferentes status de pedido (Pendente, Alterar Pedido e Cancela Pedido), filas SQS FIFO de destino para cada status com suas respectivas DLQs, e serão realizados testes do fluxo via console.

Na terceira parte do fluxo, será criado um API Gateway para receber pedidos como alternativa ao envio para o Data Lake (S3). Uma função Lambda, utilizando o mesmo Layer de validação da etapa anterior, processará os pedidos recebidos, validando os dados e encaminhando os pedidos válidos para o Event Bus, que foi configurado com as regras de roteamento na etapa anterior. Em caso de falha na validação, o usuário receberá uma mensagem de erro diretamente pelo API Gateway. O fluxo será testado para garantir o roteamento correto dos pedidos válidos e o retorno apropriado de erros para entradas inválidas.

Na quarta e última parte, será desenvolvido um backend com três funções Lambda responsáveis por operar sobre os pedidos armazenados nas filas SQS. Cada função Lambda será responsável por uma operação específica: armazenar, alterar ou cancelar o pedido no DynamoDB. As funções Lambda processarão os pedidos de acordo com o padrão de Single Table Design.



---

## **O que Você Irá Aprender nesta atividade**

- [**Etapa 1:** Criar o Bucket S3 e o Prefixo para os Pedidos](#etapa-1-criar-o-bucket-s3-e-o-prefixo-para-os-pedidos).
- [**Etapa 2:** Criar as Filas SQS](#etapa-2-criar-as-filas-sqs)
  - [**2.1** Criar fila SQS Standard para o S3](#21-fila-standard-sqs-pedidos-json)
  - [**2.2** Criar fila FIFO dlq para armazena pedidos que não puderam ser processados após o número máximo de tentativas.](#22-fila-dlq-fifo-sqs-pedidos-dlqfifo)
  - [**2.3** Criar fila FIFO para armazena pedidos de forma individual](#23-fila-fifo-sqs-pedidos-validosfifo)
- [**Etapa 3:** Criar Tabela no DynamoDB para controle de indempotência de pedidos e arquivos JSON.](#etapa-3-criar-tabela-no-dynamodb)
- [**Etapa 4:** Lambda de Extração de pedido e Controle de Duplicidade](#etapa-4-lambda-de-extração-de-pedido-e-controle-de-duplicidade)
- [**Etapa 5:** Criar o Triger para Lambda SQS](#dd)

---

### **Pré-requisitos**

- **Conta AWS ativa** com permissões para criar recursos (S3, SQS, SNS, Lambda, EventBridge e IAM).
- **Acesso ao console da AWS** via navegador web.


### Fluxo de Trabalho:
- O Lambda é acionado quando a fila SQS padrão recebe uma notificação de evento S3.
- O Lambda lê o arquivo do S3 e processa os pedidos.
- Cada pedido é enviado para a fila SQS FIFO de destino.
- O Lambda registra os pedidos no DynamoDB com detalhes básicos e evita duplicações.

---

## **Etapa 1**: Criar o Bucket S3 e o Prefixo para os Pedidos

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


## **Etapa 4: Lambda de Extração de pedido e Controle de Duplicidade**
Esta função lê o arquivo JSON do S3, registra os pedidos no DynamoDB para evitar duplicação e envia os pedidos para a fila FIFO.

#### **Passos:**
1. Acesse **AWS Lambda** > **Create function**.
2. **Function name:** `extract-and-send-lambda`.
3. **Runtime:** `Python 3.13`.
4. **Timeout:** 30 minutos.
5. **Memory:** 256 MB.
6. Vincule a layer `order-validation-layer`.
7. Adicione variáveis de ambiente:
   - **`DYNAMO_TABLE_NAME`**: `OrdersTable`
   - **`SQS_FIFO_URL`**: URL da fila FIFO.

#### **Código:**
~~~python
import boto3, json, os, logging
from datetime import datetime
from decimal import Decimal

def decimal_default(obj):
    if isinstance(obj, (float, Decimal)):
        return str(obj)  # Convertendo para string para evitar erro de Decimal no JSON
    raise TypeError(f"Tipo não suportado: {type(obj)} - Valor: {obj}")

logger = logging.getLogger()
logger.setLevel(logging.INFO)

dynamo_client = boto3.resource('dynamodb')
sqs_client = boto3.client('sqs')
s3_client = boto3.client('s3')

DYNAMO_TABLE_NAME = os.getenv('DYNAMO_TABLE_NAME')
SQS_FIFO_URL = os.getenv('SQS_FIFO_URL')

def lambda_handler(event, context):
    for record in event['Records']:
        try:
            # Extrair a mensagem da fila SQS com evento S3
            body = json.loads(record['body'])
            s3_event = body.get('Records', [])[0] if 'Records' in body else None
            if not s3_event or 's3' not in s3_event:
                logger.error("Mensagem SQS não contém informações S3 válidas: %s", body)
                continue

            bucket = s3_event['s3']['bucket']['name']
            key = s3_event['s3']['object']['key']

            logger.info(f"Iniciando processamento do arquivo: {key}")

            # Ler o arquivo do S3
            response = s3_client.get_object(Bucket=bucket, Key=key)
            orders = json.loads(response['Body'].read().decode('utf-8'), parse_float=Decimal)
            logger.debug(f"Conteúdo do arquivo: {orders}")

            if check_file_processed(key):
                logger.info(f"Arquivo {key} já processado.")
                continue

            # Processar os pedidos e enviar para a fila
            successful_orders = []
            for order in orders:
                logger.debug(f"Processando pedido: {order['order_id']}")
                if should_process_order(order):
                    logger.info(f"Pedido elegível para processamento: {order['order_id']} com status {order['order_status']}")
                    if send_to_fifo(order):
                        successful_orders.append(order)
                else:
                    logger.warning(f"Pedido {order['order_id']} ignorado devido ao status: {order['order_status']}")

            # Registrar o arquivo no DynamoDB apenas se houver pedidos enviados
            if successful_orders:
                register_file_in_dynamodb(key)
                register_orders_in_dynamodb(successful_orders, key)

        except KeyError as e:
            logger.error(f"Erro na estrutura da mensagem SQS: campo ausente {str(e)}", exc_info=True)
        except Exception as e:
            logger.error(f"Erro ao processar a mensagem da fila SQS: {str(e)}", exc_info=True)

    return {"statusCode": 200, "body": "Processamento concluído com sucesso"}

def check_file_processed(file_name):
    table = dynamo_client.Table(DYNAMO_TABLE_NAME)
    response = table.get_item(Key={"PK": f"FILE#{file_name}", "SK": "SUMMARY"})
    is_processed = 'Item' in response
    logger.debug(f"Arquivo {file_name} já registrado? {is_processed}")
    return is_processed

def register_file_in_dynamodb(file_name):
    logger.info(f"Registrando arquivo {file_name} no DynamoDB.")
    table = dynamo_client.Table(DYNAMO_TABLE_NAME)
    table.put_item(Item={"PK": f"FILE#{file_name}", "SK": "SUMMARY", "ProcessedAt": datetime.utcnow().isoformat()})

def register_orders_in_dynamodb(orders, file_name):
    table = dynamo_client.Table(DYNAMO_TABLE_NAME)
    with table.batch_writer() as batch:
        for order in orders:
            order_id = order['order_id']
            status = order['order_status']
            try:
                logger.debug(f"Registrando pedido {order_id} com status {status}")
                batch.put_item(Item={
                    "PK": f"ORDER#{order_id}",
                    "SK": f"STATUS#{status}",
                    "OrderStatus": status,
                    "ArquivoOrigem": file_name,
                    "ProcessedAt": datetime.utcnow().isoformat(),
                    "CompanyName": order['company']['name']
                })
            except Exception as e:
                logger.error(f"Erro ao registrar pedido {order_id} no DynamoDB: {str(e)}")

def send_to_fifo(order):
    try:
        logger.debug(f"Serializando pedido {order['order_id']} para envio.")
        order_message = json.dumps(order, default=decimal_default)

        params = {
            "QueueUrl": SQS_FIFO_URL,
            "MessageBody": order_message,
            "MessageGroupId": "orders-group"
        }

        # Adicionar MessageDeduplicationId sempre para garantir unicidade
        deduplication_id = str(hash(f"{order['order_id']}-{order['order_status']}-{datetime.utcnow().isoformat()}"))
        params["MessageDeduplicationId"] = deduplication_id
        logger.debug(f"Usando MessageDeduplicationId: {deduplication_id}")

        sqs_client.send_message(**params)
        logger.info(f"Pedido {order['order_id']} enviado com sucesso para a fila.")
        return True
    except Exception as e:
        logger.error(f"Erro ao enviar pedido {order['order_id']} para a fila: {str(e)}")
        return False

def should_process_order(order):
    status = order['order_status']
    logger.debug(f"Verificando status do pedido {order['order_id']}: {status}")
    if status in ["PedidoCancelado", "PedidoAlterado"]:
        return True
    elif status in ["Pendente", "PedidoNovo"]:
        table = dynamo_client.Table(DYNAMO_TABLE_NAME)
        response = table.get_item(Key={"PK": f"ORDER#{order['order_id']}", "SK": f"STATUS#{status}"})
        exists = 'Item' in response
        logger.debug(f"Pedido {order['order_id']} com status {status} já existe? {exists}")
        return not exists  # Só processa se não existir
    return False
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
3. Confira as mensagens nas fila FIFO.

---

## **Conclusão**
Essa atualização melhora a resiliência e modularidade do processamento de pedidos, com explicações detalhadas e um fluxo bem estruturado que separa cada responsabilidade de forma clara.
