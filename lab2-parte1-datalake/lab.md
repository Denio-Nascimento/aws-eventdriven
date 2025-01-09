# Introdução ao Laboratório de Processamento de Pedidos com AWS

Este laboratório tem como objetivo ensinar a criação de uma arquitetura **serverless** robusta na AWS para processar pedidos de venda a partir de arquivos JSON armazenados no Amazon S3. Durante esta atividade prática, você aprenderá a implementar uma solução com validação de pedidos, envio de eventos ao EventBridge, registro de dados no DynamoDB e tratamento de erros com SQS.

A arquitetura proposta é altamente escalável e permite o uso de **Lambda Layers** para reaproveitamento de código, seguindo as melhores práticas de modularidade e manutenção. Ao final do laboratório, você terá uma aplicação que valida os pedidos, identifica erros e envia notificações para monitoramento.

---

## **O que Você Irá Aprender**

Nesta atividade, você irá:

- Criar uma fila de erros no Amazon SQS para receber pedidos inválidos.
- Criar uma tabela no Amazon DynamoDB para registrar arquivos e pedidos processados.
- Criar um barramento de eventos no Amazon EventBridge.
- Criar uma **AWS Lambda Layer** para validação dos pedidos.
- Criar uma função AWS Lambda para pré-validação dos pedidos JSON utilizando a layer.
- Configurar a função Lambda com as permissões IAM adequadas e variáveis de ambiente.
- Configurar o S3 para invocar diretamente a função Lambda.
- Realizar testes de validação e envio ao EventBridge.

---

### **Pré-requisitos**

- **Conta AWS ativa** com permissões para criar recursos (S3, SQS, Lambda, EventBridge e IAM).
- **Acesso ao console da AWS** via navegador web.

---

## **Etapa 1: Criar a Fila SQS de Erros**

1. **Acessar o Amazon SQS:**
   - No menu de serviços, selecione **SQS** (ou utilize a barra de pesquisa).

2. **Criar uma nova fila:**
   - Clique em **Create queue**.
   - Escolha **Standard Queue**.

3. **Configurações da fila:**
   - **Queue Name:** `sqs-erros-pedidos`.

4. **Finalizar a criação:**
   - Clique em **Create Queue**.

---

## **Etapa 2: Criar Tabela no DynamoDB**

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

## **Etapa 3: Criar o Barramento de Eventos no EventBridge**

1. **Acessar o Amazon EventBridge:**
   - No menu de serviços, selecione **EventBridge**.

2. **Criar um novo Event Bus:**
   - Clique em **Create Event Bus**.
   - **Name:** `event-bus-pedidos`.

3. **Finalizar a criação:**
   - Clique em **Create**.

---

## **Etapa 4: Criar a Layer de Validação**

### **4.1. Criar Código da Layer**
Crie um arquivo chamado `validation.py` com o seguinte conteúdo:

```python
def validate_order(order):
    required_fields = ["order_id", "customer", "items", "payment", "company", "order_status"]
    for field in required_fields:
        if field not in order:
            return False
    return order["order_status"] in ["Novo Pedido", "Alterar Pedido", "Cancela Pedido"]
```

### **4.2. Empacotar a Layer**
1. Crie uma pasta `python` e coloque o arquivo `validation.py` dentro dela.
2. Comprime a pasta em um arquivo ZIP:
   ```bash
   zip -r validation_layer.zip python/
   ```

### **4.3. Fazer Upload da Layer**
1. No console da AWS, vá para **Lambda** > **Layers**.
2. Clique em **Create layer**.
3. Preencha os campos:
   - **Name:** `order-validation-layer`
   - **Upload a .zip file:** faça upload do `validation_layer.zip`.
   - **Runtime:** selecione `Python 3.9`.
4. Clique em **Create**.

---

## **Etapa 5: Criar a Função Lambda de Pré-Validação**

### **5.1. Criar a Função Lambda**

1. No menu de serviços, selecione **Lambda**.
2. Clique em **Create function**.
3. Escolha **Author from scratch**.
4. Preencha os campos:
   - **Function name:** `prevalidation-lambda`
   - **Runtime:** `Python 3.9`.
5. Clique em **Create function**.

### **5.2. Vincular a Layer de Validação**
1. No painel da função Lambda, clique em **Layers**.
2. Clique em **Add a layer**.
3. Selecione **Custom layers**.
4. Escolha `order-validation-layer`.
5. Clique em **Add**.

---

### **5.3. Adicionar o Código à Função Lambda**

1. No painel da função, role até a seção **Code source**.
2. Apague o código padrão e cole o seguinte código:

```python
import boto3
import json
import os
import logging
from datetime import datetime
from validation import validate_order

# Configuração de logs
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Clientes AWS
s3_client = boto3.client('s3')
dynamodb_client = boto3.resource('dynamodb')
eventbridge_client = boto3.client('events')
sqs_client = boto3.client('sqs')
sns_client = boto3.client('sns')

# Variáveis de ambiente
DYNAMO_TABLE_NAME = os.getenv('DYNAMO_TABLE_NAME')
SNS_TOPIC_ARN = os.getenv('SNS_TOPIC_ARN')
SQS_ERROR_QUEUE_URL = os.getenv('SQS_ERROR_QUEUE_URL')
EVENT_BUS_NAME = os.getenv('EVENT_BUS_NAME')

def lambda_handler(event, context):
    """Função principal para processar pedidos do S3 e enviar para o EventBridge."""
    logger.info("Iniciando o processamento de pedidos.")
    for record in event['Records']:
        bucket_name = record['s3']['bucket']['name']
        file_name = record['s3']['object']['key']

        try:
            # Baixar arquivo JSON do S3
            response = s3_client.get_object(Bucket=bucket_name, Key=file_name)
            file_content = response['Body'].read().decode('utf-8')
            orders = json.loads(file_content)
        except Exception as e:
            logger.error(f"Erro ao ler o arquivo {file_name}: {str(e)}")
            return {
                "statusCode": 500,
                "body": f"Erro ao ler o arquivo {file_name}."
            }

        if check_file_processed(file_name):
            logger.info(f"Arquivo {file_name} já foi processado.")
            return {
                "statusCode": 200,
                "body": "Arquivo duplicado ignorado."
            }

        register_file_in_dynamodb(file_name, orders)

        for order in orders:
            order_id = order.get("order_id")
            company_cnpj = order["company"]["cnpj"]

            # Validar pedido utilizando a layer
            if not validate_order(order):
                send_to_error_queue(order, "Pedido inválido ou com status desconhecido.")
                send_sns_notification(order_id, f"Pedido {order_id} inválido ou com status inválido.")
                continue

            if check_order_processed(order_id, company_cnpj):
                logger.info(f"Pedido {order_id} da empresa {company_cnpj} já foi processado.")
                continue

            send_to_eventbridge(order)
            register_order_in_dynamodb(order_id, company_cnpj, file_name)

        return {
            "statusCode": 200,
            "body": f"Arquivo {file_name} processado com sucesso."
        }


def check_file_processed(file_name):
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    response = table.get_item(Key={"PK": f"FILE#{file_name}", "SK": "SUMMARY"})
    return 'Item' in response

def register_file_in_dynamodb(file_name, orders):
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    processing_date = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")

    table.put_item(
        Item={
            "PK": f"FILE#{file_name}",
            "SK": "SUMMARY",
            "Orders": json.dumps(orders),
            "Status": "Processed",
            "ProcessingDate": processing_date
        }
    )

def check_order_processed(order_id, company_cnpj):
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    response = table.get_item(Key={"PK": f"ORDER#{order_id}", "SK": f"COMPANY#{company_cnpj}"})
    return 'Item' in response

def register_order_in_dynamodb(order_id, company_cnpj, file_name):
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    processing_date = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")

    table.put_item(
        Item={
            "PK": f"ORDER#{order_id}",
            "SK": f"COMPANY#{company_cnpj}",
            "FileName": file_name,
            "Status": "Processed",
            "ProcessingDate": processing_date
        }
    )

def send_to_eventbridge(order):
    """Envia pedido válido para o EventBridge."""
    try:
        eventbridge_client.put_events(
            Entries=[
                {
                    "Source": "app.pedidos",
                    "DetailType": "PedidoProcessado",
                    "Detail": json.dumps(order),
                    "EventBusName": EVENT_BUS_NAME
                }
            ]
        )
        logger.info(f"Pedido {order['order_id']} enviado para EventBridge.")
    except Exception as e:
        logger.error(f"Erro ao enviar pedido {order['order_id']} para EventBridge: {str(e)}")

def send_to_error_queue(order, reason):
    """Envia mensagens inválidas para a fila de erros."""
    try:
        sqs_client.send_message(
            QueueUrl=SQS_ERROR_QUEUE_URL,
            MessageBody=json.dumps({"order": order, "reason": reason})
        )
        logger.info(f"Pedido {order['order_id']} enviado para a fila de erros.")
    except Exception as e:
        logger.error(f"Erro ao enviar pedido {order['order_id']} para fila de erros: {str(e)}")

def send_sns_notification(order_id, message):
    try:
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"Erro no pedido {order_id}",
            Message=message
        )
        logger.info(f"Notificação enviada para pedido {order_id}: {message}")
    except Exception as e:
        logger.error(f"Erro ao enviar notificação SNS para pedido {order_id}: {str(e)}")
```

---

### **5.4. Adicionar Variáveis de Ambiente**

1. No painel da função Lambda, clique em **Configuration** > **Environment variables**.
2. Clique em **Edit**.
3. Clique em **Add environment variable**.
4. Adicione as seguintes variáveis:
   - **`DYNAMO_TABLE_NAME`**: `OrdersTable`
   - **`SNS_TOPIC_ARN`**: ARN do tópico SNS.
   - **`SQS_ERROR_QUEUE_URL`**: URL da fila SQS de erros.
   - **`EVENT_BUS_NAME`**: `event-bus-pedidos`.

---

## **Etapa 6: Testar o Fluxo Completo**

### **6.1. Realizar Upload de Arquivo no S3**
1. No bucket S3, clique em **Upload**.
2. Faça upload de um arquivo JSON de pedidos, como:

```json
{
  "order_id": "TS-00123",
  "company": {
    "name": "Tech Solutions Ltda",
    "cnpj": "12.345.678/0001-99"
  },
  "order_status": "Novo Pedido",
  "items": [
    {
      "product_id": "TS-001",
      "description": "Smartphone XYZ",
      "quantity": 1,
      "unit_price": 2500.00
    }
  ]
}
```

### **6.2. Verificar Logs e Mensagens**
1. Acesse o console **CloudWatch Logs**.
2. Verifique se há mensagens de sucesso ou erros na função Lambda.
3. Acesse o console do **EventBridge** e **SQS** para confirmar o roteamento e mensagens de erro.

---

## **Conclusão**
Você concluiu a atividade prática, criando uma solução **serverless** com **S3**, **Lambda**, **EventBridge**, **DynamoDB**, e **SQS**, utilizando uma **AWS Lambda Layer** para validação de pedidos. Esta arquitetura permite o reuso de código e melhora a modularidade e manutenibilidade da solução.

