# AWS Lambda - Processamento de Pedidos com S3, SQS, SNS e DynamoDB

## **Visão Geral**
Este projeto exemplifica uma aplicação serverless em AWS que processa arquivos JSON contendo pedidos de compra armazenados em um bucket S3. O arquivo é lido, validado e cada pedido é enviado individualmente para uma SQS FIFO, enquanto os registros são mantidos no DynamoDB para garantir idempotência.

---

## **Arquitetura**

### **Componentes Utilizados:**
- **Amazon S3:** Repositório de arquivos JSON contendo os pedidos.
- **AWS Lambda:** Função responsável por processar os arquivos e enviar os pedidos.
- **Amazon SQS Standard:** Recebe notificações de novos arquivos.
- **Amazon SQS FIFO:** Recebe pedidos individuais após validação.
- **Amazon DynamoDB:** Registro de arquivos e pedidos para garantir idempotência.
- **Amazon SNS:** Notificações de erros de pedidos inválidos.

---

## **Detalhes de Implementação**

### **Lambda Handler (`lambda_handler.py`)**

```python
import boto3
import json
import os
from datetime import datetime

# Clientes AWS
s3_client = boto3.client('s3')
sqs_client = boto3.client('sqs')
dynamodb_client = boto3.resource('dynamodb')
sns_client = boto3.client('sns')

# Variáveis de ambiente
SQS_FIFO_QUEUE_URL = os.getenv('SQS_FIFO_QUEUE_URL')
DYNAMO_TABLE_NAME = os.getenv('DYNAMO_TABLE_NAME')
SNS_TOPIC_ARN = os.getenv('SNS_TOPIC_ARN')

def lambda_handler(event, context):
    """Função principal para processar pedidos do S3 e enviar para a SQS FIFO."""
    for record in event['Records']:
        bucket_name = record['s3']['bucket']['name']
        file_name = record['s3']['object']['key']

        try:
            # Baixar arquivo JSON do S3
            response = s3_client.get_object(Bucket=bucket_name, Key=file_name)
            file_content = response['Body'].read().decode('utf-8')
            orders = json.loads(file_content)
        except Exception as e:
            print(f"Erro ao ler o arquivo {file_name}: {str(e)}")
            return {
                "statusCode": 500,
                "body": f"Erro ao ler o arquivo {file_name}."
            }

        # Verificar se o arquivo já foi processado
        if check_file_processed(file_name):
            print(f"Arquivo {file_name} já foi processado.")
            return {
                "statusCode": 200,
                "body": "Arquivo duplicado ignorado."
            }

        # Registrar o arquivo com todos os pedidos em um único JSON
        register_file_in_dynamodb(file_name, orders)

        # Processar cada pedido individualmente
        for order in orders:
            order_id = order.get("order_id")
            company_cnpj = order["company"]["cnpj"]

            # Validar campos obrigatórios
            if not validate_order(order):
                send_sns_notification(order_id, f"Pedido {order_id} inválido. Campos obrigatórios faltando.")
                continue

            # Verificar se o pedido já foi processado
            if check_order_processed(order_id, company_cnpj):
                print(f"Pedido {order_id} da empresa {company_cnpj} já foi processado.")
                continue

            # Enviar pedido válido para SQS FIFO
            send_to_fifo_queue(order)

            # Registrar pedido individual no DynamoDB
            register_order_in_dynamodb(order_id, company_cnpj, file_name)

        return {
            "statusCode": 200,
            "body": f"Arquivo {file_name} processado com sucesso."
        }

def validate_order(order):
    """Valida os campos obrigatórios do pedido."""
    required_fields = ["order_id", "customer", "items", "payment", "company"]
    for field in required_fields:
        if field not in order:
            return False
    return True

def check_file_processed(file_name):
    """Verifica se o arquivo já foi processado."""
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    response = table.get_item(Key={"PK": f"FILE#{file_name}", "SK": "SUMMARY"})
    return 'Item' in response

def register_file_in_dynamodb(file_name, orders):
    """Registra o arquivo no DynamoDB com todos os pedidos em JSON."""
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    processing_date = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")

    table.put_item(
        Item={
            "PK": f"FILE#{file_name}",
            "SK": "SUMMARY",
            "Orders": json.dumps(orders),  # Armazena todos os pedidos em JSON
            "Status": "Processed",
            "ProcessingDate": processing_date
        }
    )
    print(f"Arquivo {file_name} registrado no DynamoDB com todos os pedidos.")

def check_order_processed(order_id, company_cnpj):
    """Verifica se o pedido já foi processado para uma empresa específica."""
    table = dynamodb_client.Table(DYNAMO_TABLE_NAME)
    response = table.get_item(Key={"PK": f"ORDER#{order_id}", "SK": f"COMPANY#{company_cnpj}"})
    return 'Item' in response

def register_order_in_dynamodb(order_id, company_cnpj, file_name):
    """Registra o pedido individual no DynamoDB."""
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
    print(f"Pedido {order_id} registrado no DynamoDB com sucesso.")

def send_to_fifo_queue(order):
    """Envia pedido válido para SQS FIFO."""
    try:
        sqs_client.send_message(
            QueueUrl=SQS_FIFO_QUEUE_URL,
            MessageBody=json.dumps(order),
            MessageGroupId="OrdersGroup"
        )
        print(f"Pedido {order['order_id']} enviado para SQS FIFO.")
    except Exception as e:
        print(f"Erro ao enviar pedido {order['order_id']} para SQS FIFO: {str(e)}")

def send_sns_notification(order_id, message):
    """Envia notificação SNS para pedidos inválidos."""
    try:
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"Erro no pedido {order_id}",
            Message=message
        )
        print(f"Notificação enviada para pedido {order_id}: {message}")
    except Exception as e:
        print(f"Erro ao enviar notificação SNS para pedido {order_id}: {str(e)}")
```

---

## **Descrição das Funções**

### `lambda_handler`
Função principal que:
- Lê o arquivo JSON do S3.
- Valida cada pedido.
- Verifica duplicidade do arquivo e do pedido no DynamoDB.
- Envia os pedidos válidos para a SQS FIFO.
- Registra os pedidos processados no DynamoDB.

### `validate_order`
Verifica se os campos obrigatórios (`order_id`, `customer`, `items`, `payment`, `company`) estão presentes.

### `check_file_processed`
Consulta no DynamoDB se o arquivo já foi processado usando `PK = FILE#<file_name>` e `SK = SUMMARY`.

### `register_file_in_dynamodb`
Registra o arquivo como um único item no DynamoDB com todos os pedidos em JSON.

### `check_order_processed`
Verifica se um pedido específico já foi processado para uma empresa (`PK = ORDER#<order_id>` e `SK = COMPANY#<cnpj>`).

### `register_order_in_dynamodb`
Registra o pedido individualmente no DynamoDB com informações como `FileName`, `Status` e `ProcessingDate`.

### `send_to_fifo_queue`
Envia os pedidos válidos para a SQS FIFO com `MessageGroupId` fixo para manter a ordenação.

### `send_sns_notification`
Publica uma notificação no SNS caso o pedido seja inválido.

---

## **Variáveis de Ambiente**

| Variável            | Descrição                       | Exemplo                               |
|---------------------|----------------------------------|---------------------------------------|
| `SQS_FIFO_QUEUE_URL` | URL da SQS FIFO                  | `https://sqs.sa-east-1.amazonaws.com/...` |
| `DYNAMO_TABLE_NAME`  | Nome da tabela DynamoDB          | `OrdersTable`                         |
| `SNS_TOPIC_ARN`      | ARN do tópico SNS                | `arn:aws:sns:sa-east-1:123456789012:InvalidOrders` |

---

## **Pré-requisitos**
- **DynamoDB Table:**
  - Com `PK` (`Partition Key`): `STRING`
  - Com `SK` (`Sort Key`): `STRING`

---

## **Exemplo de Registro no DynamoDB**

### Registro do Arquivo
```json
{
  "PK": "FILE#orders_batch_2025-01-07.json",
  "SK": "SUMMARY",
  "Orders": "[{\"order_id\": \"AL-56300\", \"company\": {\"cnpj\": \"47.569.380/0001-43\"}, ... }]",
  "Status": "Processed",
  "ProcessingDate": "2025-01-08 15:45:00"
}
```

### Registro de Pedido Individual
```json
{
  "PK": "ORDER#AL-56300",
  "SK": "COMPANY#47.569.380/0001-43",
  "FileName": "orders_batch_2025-01-07.json",
  "Status": "Processed",
  "ProcessingDate": "2025-01-08 15:45:00"
}
```

---

## **Melhorias Futuras**
- **Validação Avançada:** Utilizar bibliotecas como `jsonschema` para validações mais detalhadas.
- **Auditoria:** Implementar logs detalhados com AWS CloudWatch.
- **Dead Letter Queue (DLQ):** Configurar DLQ para mensagens que falharem na SQS.

---

Com esse projeto, é possível garantir o processamento de pedidos com controle de idempotência e alta confiabilidade. Sinta-se à vontade para adaptar ou expandir conforme as necessidades!

