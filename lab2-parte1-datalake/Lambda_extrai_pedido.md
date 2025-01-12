# Documentação e Código AWS Lambda

## Descrição:
Este AWS Lambda é responsável por receber mensagens de uma fila SQS padrão com informações de eventos de S3 ao serem criados novos arquivos. O Lambda lê o evento do S3, faz o download do arquivo, processa os pedidos e registra as informações no DynamoDB. Também envia os pedidos para outra fila SQS FIFO, garantindo que cada mensagem tenha um identificador de deduplicação exclusivo.

---

## Estrutura do DynamoDB:
- **Tabela:** `<DYNAMO_TABLE_NAME>`
- **PK:** `ORDER#<order_id>`
- **SK:** `STATUS#<order_status>`
- **Outros atributos:**
  - `OrderStatus`: Status atual do pedido.
  - `ArquivoOrigem`: Nome do arquivo de origem.
  - `ProcessedAt`: Data e hora do processamento.
  - `CompanyName`: Nome da empresa.

---

## Código Python do AWS Lambda:

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

---

## Fluxo de Trabalho:
1. O Lambda é acionado quando a fila SQS padrão recebe uma notificação de evento S3.
2. O Lambda lê o arquivo do S3 e processa os pedidos.
3. Cada pedido é enviado para a fila SQS FIFO de destino.
4. O Lambda registra os pedidos no DynamoDB com detalhes básicos e evita duplicações.

---

## Considerações:
- **Deduplicação:** O `MessageDeduplicationId` é gerado com base no `order_id`, `order_status` e um timestamp para garantir unicidade.
- **Logs:** O Lambda possui logs de erro e informações detalhadas para facilitar o monitoramento.

Se precisar de mais informações ou ajustes, estou à disposição!
