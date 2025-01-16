# **Laboratório Serverless AWS: Operações Finais de Pedido (Quarta Parte)**

Bem-vindo(a) à **quarta e última parte** do laboratório sobre **Arquitetura Event Driven** para **processar pedidos** na AWS. Aqui, vamos **finalizar** o fluxo, criando **três funções Lambda** que vão **operar** (inserir, alterar ou cancelar) os pedidos no DynamoDB, consumindo mensagens de **três filas SQS FIFO** distintas, roteadas pelo EventBridge.

> **Objetivo**  
> - Consumir mensagens das filas SQS `sqs-pedido-pendente.fifo`, `sqs-pedido-alterado.fifo` e `sqs-pedido-cancelado.fifo`.  
> - Executar a **lógica** correspondente: inserir/criar, alterar e cancelar pedidos em uma **Single Table** DynamoDB.  
> - Fazer uso do **AWS Systems Manager Parameter Store** para armazenar o **nome da tabela**.  
> - Adotar o **princípio de menor privilégio** nas **IAM Policies**.

---

## **Estrutura da Tabela DynamoDB**

A tabela será estruturada da seguinte forma:

| **PK**                          | **SK**                  | **GSI2#PK**               | **GSI2#SK**                                         |
|----------------------------------|-------------------------|--------------------------|---------------------------------------------------|
| COMPANY#<CNPJ>#ORDER#<order_id>  | STATUS#<order_status>   |                          |                                                   |
|                                  | ITEM#<product_id>       | ITEMSTATUS#<item_status> | COMPANY#<cnpj>#ORDER#<order_id>#ITEM#<product_id> |
|                                  | CUSTOMER#<cpf>          |                          |                                                   |
|                                  | META#<cnpj>             |                          |                                                   |
|                                  | SHIPPING#<UF>#<date>  |                          |                                                   |
|                                  | PAYMENT#<payment_method>|                          |                                                   |

Além do índice secundário `GSI2`, será criado um **novo índice GSI1** que inverte a **PK** e a **SK**:
- **Partition Key:** `SK`
- **Sort Key:** `PK`

Esse índice permite realizar consultas rápidas utilizando os atributos secundários como principais.

---


## **Índice**

1. [**Visão Geral**](#visão-geral)  
2. [**O que Você Irá Aprender**](#o-que-você-irá-aprender)  
3. [**Pré-requisitos**](#pré-requisitos)  
4. [**Etapa 1: Criar/Confirmar Tabela DynamoDB e Parameter Store**](#etapa-1-criarconfirmar-tabela-dynamodb-e-parameter-store)  
5. [**Etapa 2: Lambda de Inclusão (Pendente)**](#etapa-2-lambda-de-inclusão-pendente)  
   - [2.1 Criar a Lambda](#21-criar-a-lambda)  
   - [2.2 Código-Fonte da Lambda](#22-código-fonte-da-lambda)  
   - [2.3 Permissões IAM](#23-permissões-iam)  
   - [2.4 Adicionar Trigger SQS](#24-adicionar-trigger-sqs)  
6. [**Etapa 3: Lambda de Alteração**](#etapa-3-lambda-de-alteração)  
   - [3.1 Criar a Lambda](#31-criar-a-lambda)  
   - [3.2 Código-Fonte**](#32-código-fonte)  
   - [3.3 Permissões IAM](#33-permissões-iam)  
   - [3.4 Adicionar Trigger SQS](#34-adicionar-trigger-sqs)  
7. [**Etapa 4: Lambda de Cancelamento**](#etapa-4-lambda-de-cancelamento)  
   - [4.1 Criar a Lambda](#41-criar-a-lambda)  
   - [4.2 Código-Fonte**](#42-código-fonte)  
   - [4.3 Permissões IAM](#43-permissões-iam)  
   - [4.4 Adicionar Trigger SQS](#44-adicionar-trigger-sqs)  
8. [**Etapa 5: Testes Manuais**](#etapa-5-testes-manuais)  
9. [**Conclusão**](#conclusão)  

---

## **Visão Geral**

Nas partes anteriores, construímos:

- Uma **Lambda** que carrega pedidos do S3 e envia para `sqs-pedidos-validos.fifo` (Parte 1).  
- Uma **Lambda** que valida e encaminha ao EventBridge, que roteia conforme `order_status` para as filas `sqs-pedido-pendente.fifo`, `sqs-pedido-alterado.fifo`, e `sqs-pedido-cancelado.fifo` (Parte 2).  
- Uma **opção** de API Gateway para inserir pedidos diretamente, também enviando para o EventBridge (Parte 3).

Agora, finalizamos o fluxo criando:

1. **Lambda de Inclusão** (pedido pendente).  
2. **Lambda de Alteração** (pedido alterado).  
3. **Lambda de Cancelamento** (pedido cancelado).

Cada Lambda consumirá sua **fila SQS** respectiva e atualizará o **DynamoDB** em **Single Table Design**.

---

## **O que Você Irá Aprender**

- Criar Lambdas que **lêem** parâmetros do **Parameter Store** (nome da tabela).  
- Inserir e atualizar registros em um **Single Table** no DynamoDB.  
- Criar **Policies IAM** seguindo o **princípio de menor privilégio**.  
- **Testar** o fluxo e checar resultados no **DynamoDB**.

---

## **Pré-requisitos**

- Ter as filas **FIFO**:
  - `sqs-pedido-pendente.fifo`
  - `sqs-pedido-alterado.fifo`
  - `sqs-pedido-cancelado.fifo`
- Ter o **Event Bus** (ex.: `event-bus-pedidos`) configurado com **regras** que mandem `order_status = "Pendente"` para `sqs-pedido-pendente.fifo`, etc.
- Ter uma **função** (da parte 2) que envia pedidos com esses `order_status` para o EventBridge.
- Ter a **tabela** `pedido` (ou outro nome) no DynamoDB, com **PK** (string) e **SK** (string).

---

## **Etapa 1: Criar/Confirmar Tabela DynamoDB e Parameter Store**

### 1.1. Criar Tabela DynamoDB (caso não tenha)

1. Acesse **DynamoDB** > **Tables** > **Create table**.  
2. **Table name:** `pedido`  
3. **Partition key:** `PK` (String)  
4. **Sort key:** `SK` (String)  
5. Deixe as outras configurações no padrão e clique em **Create table**.

### 1.2. Criar Parameter Store
Para evitar “hardcode” do nome da tabela, criaremos um **Parameter**:

1. Vá em **AWS Systems Manager** > **Parameter Store**.  
2. Clique em **Create parameter**.  
3. **Name**: `/lab/pedido/table_name`  
4. **Type**: String  
5. **Value**: `pedido` (o nome exato da tabela).  
6. Clique em **Create parameter**.

Com isso, cada Lambda poderá ler `ssm.get_parameter(Name="/lab/pedido/table_name")` para descobrir o nome da tabela.

---

## **Etapa 2: Lambda de Inclusão (Pendente)**

Quando chega um pedido com `order_status = "Pendente"`, o EventBridge envia para a fila `sqs-pedido-pendente.fifo`. Esta Lambda faz **inserção** (ou batch insert) no DynamoDB.

### 2.1. Criar a Lambda

1. Acesse **AWS Lambda** > **Create function**.  
2. **Function name:** `incluir-pedido-lambda` (por exemplo).  
3. **Runtime:** Python 3.13 (ou outra versão atual).  
4. Clique em **Create function**.

### 2.2. Código-Fonte da Lambda

Cole o **código** abaixo (que lê do Parameter Store, monta `PK` e `SK`, e faz `batch_write_item`):

```python
import json
import boto3

ssm = boto3.client('ssm')
dynamo = boto3.client('dynamodb')

def lambda_handler(event, context):
    try:
        # Ler nome da tabela do Parameter Store
        table_name = ssm.get_parameter(Name="/lab/pedido/table_name")["Parameter"]["Value"]

        for record in event["Records"]:
            body_str = record["body"]
            envelope = json.loads(body_str)
            detail = envelope.get("detail", {})

            # Monta PK = COMPANY#<cnpj>#ORDER#<order_id>
            company = detail.get("company", {})
            cnpj = company.get("cnpj", "NO_CNPJ")
            order_id = detail.get("order_id", "NO_ORDER_ID")
            order_status = detail.get("order_status", "N/A")
            notes = detail.get("notes", "")
            order_date = detail.get("order_date", "")

            shipping = detail.get("shipping", {})
            shipping_address = shipping.get("address", {})
            uf = shipping_address.get("state", "NO_STATE")
            expected_date = shipping.get("expected_delivery_date", "1970-01-01")

            customer = detail.get("customer", {})
            cpf = customer.get("cpf", "NO_CPF")

            items_array = detail.get("items", [])

            pk_base = f"COMPANY#{cnpj}#ORDER#{order_id}"

            records_to_write = []

            # 1) STATUS
            status_item = {
                "PutRequest": {
                    "Item": {
                        "PK": {"S": pk_base},
                        "SK": {"S": f"STATUS#{order_status}"},
                        "order_date": {"S": order_date},
                        "order_status": {"S": order_status},
                        "notes": {"S": notes}
                    }
                }
            }
            records_to_write.append(status_item)

            # 2) CUSTOMER
            customer_item = {
                "PutRequest": {
                    "Item": {
                        "PK": {"S": pk_base},
                        "SK": {"S": f"CUSTOMER#{cpf}"},
                        "first_name": {"S": customer.get("first_name", "")},
                        "last_name": {"S": customer.get("last_name", "")},
                        "cpf": {"S": cpf},
                        "email": {"S": customer.get("email", "")},
                        "phone": {"S": customer.get("phone", "")}
                    }
                }
            }
            records_to_write.append(customer_item)

            # 3) SHIPPING
            shipping_sk = f"SHIPPING#{uf}#{expected_date}"
            shipping_item = {
                "PutRequest": {
                    "Item": {
                        "PK": {"S": pk_base},
                        "SK": {"S": shipping_sk},
                        "shipping_method": {"S": shipping.get("method", "")},
                        "shipping_cost": {"N": str(shipping.get("cost", 0))},
                        "uf": {"S": uf},
                        "expected_delivery_date": {"S": expected_date}
                    }
                }
            }
            records_to_write.append(shipping_item)

            # 4) ITEMS
            for prod in items_array:
                product_id = prod.get("product_id", "no_id")
                quantity = prod.get("quantity", 0)
                unit_price = prod.get("unit_price", 0)
                description = prod.get("description", "")

                sk_item = f"ITEM#{product_id}"
                item_record = {
                    "PutRequest": {
                        "Item": {
                            "PK": {"S": pk_base},
                            "SK": {"S": sk_item},
                            "product_id": {"S": product_id},
                            "description": {"S": description},
                            "quantity": {"N": str(quantity)},
                            "unit_price": {"N": str(unit_price)}
                        }
                    }
                }
                records_to_write.append(item_record)

            # Batch write
            resp = dynamo.batch_write_item(RequestItems={table_name: records_to_write})
            unprocessed = resp.get("UnprocessedItems", {})
            if unprocessed.get(table_name):
                print("WARNING | Unprocessed items:", unprocessed[table_name])

        return {"statusCode": 200, "body": json.dumps("Pedidos inseridos com sucesso!")}

    except Exception as e:
        print("Erro ao inserir pedidos:", e)
        return {
            "statusCode": 500,
            "body": json.dumps({"message": "Erro ao inserir pedidos", "error": str(e)})
        }
```

### 2.3. Permissões IAM

Precisamos de:

- **SSM**: `ssm:GetParameter` (para `/lab/pedido/table_name`).  
- **DynamoDB**: `dynamodb:BatchWriteItem` (ou `PutItem`).  
- **SQS**: `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` (para `sqs-pedido-pendente.fifo`).  

#### **Exemplo de Política Inline**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowGetParameter",
      "Effect": "Allow",
      "Action": "ssm:GetParameter",
      "Resource": "arn:aws:ssm:REGION:ACCOUNT_ID:parameter/lab/pedido/table_name"
    },
    {
      "Sid": "AllowBatchWriteDynamo",
      "Effect": "Allow",
      "Action": [
        "dynamodb:BatchWriteItem"
      ],
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/pedido"
    },
    {
      "Sid": "AllowSQSTrigger",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:REGION:ACCOUNT_ID:sqs-pedido-pendente.fifo"
    }
  ]
}
```

- Para anexar: vá na Lambda > **Configuration** > **Permissions** > clique na **Role** > **Add permissions** > **Create inline policy** > cole em **JSON** > **Review policy** > dar um nome > **Create policy**.

### 2.4. Adicionar Trigger SQS

1. Na Lambda `incluir-pedido-lambda`, vá em **Configuration** > **Triggers**.  
2. Clique em **Add trigger**.  
3. Selecione **SQS**.  
4. **SQS queue**: `sqs-pedido-pendente.fifo`.  
5. Clique em **Add**.

---

## **Etapa 3: Lambda de Alteração**

Quando `order_status = "Alterar Pedido"`, o EventBridge envia para `sqs-pedido-alterado.fifo`. Esta Lambda atualiza itens (`ITEM#<product_id>`) e define `item_status = "ALTERADO"` (por exemplo).

### 3.1. Criar a Lambda

1. **Function name**: `alterar-pedido-lambda`.  
2. Runtime: Python 3.13.  
3. **Create function**.

### 3.2. Código-Fonte

Exemplo minimalista: se quisermos apenas definir `quantity` e `item_status = ALTERADO`.  

```python
import json
import boto3

ssm = boto3.client('ssm')
dynamo = boto3.client('dynamodb')

def lambda_handler(event, context):
    try:
        table_name = ssm.get_parameter(Name="/lab/pedido/table_name")["Parameter"]["Value"]

        for record in event["Records"]:
            pedido = json.loads(record["body"])
            detail = pedido.get("detail", {})

            cnpj = detail.get("company", {}).get("cnpj", "NO_CNPJ")
            order_id = detail.get("order_id", "NO_ORDER_ID")
            pk_base = f"COMPANY#{cnpj}#ORDER#{order_id}"

            items = detail.get("items", [])

            for it in items:
                product_id = it.get("product_id", "no_id")
                quantity = it.get("quantity", 0)

                # SK = ITEM#<product_id>
                sk_item = f"ITEM#{product_id}"

                # Altera item_status e quantity
                dynamo.update_item(
                    TableName=table_name,
                    Key={
                        "PK": {"S": pk_base},
                        "SK": {"S": sk_item}
                    },
                    UpdateExpression="SET item_status = :st, quantity = :q",
                    ExpressionAttributeValues={
                        ":st": {"S": "ALTERADO"},
                        ":q":  {"N": str(quantity)}
                    }
                )

        return {"statusCode": 200, "body": "Pedido alterado com sucesso!"}

    except Exception as e:
        print("Erro ao alterar pedido:", e)
        return {"statusCode": 500, "body": str(e)}
```

### 3.3. Permissões IAM

- **ssm:GetParameter**  
- **dynamodb:UpdateItem**  
- **sqs:ReceiveMessage**, **sqs:DeleteMessage**, **sqs:GetQueueAttributes** (fila `sqs-pedido-alterado.fifo`).

(É semelhante ao exemplo anterior; só troque `BatchWriteItem` por `UpdateItem` e o ARN da fila.)

### 3.4. Adicionar Trigger SQS

1. Na Lambda, vá em **Configuration** > **Triggers**.  
2. **Add trigger**.  
3. Selecione a fila `sqs-pedido-alterado.fifo`.  
4. **Add**.

---

## **Etapa 4: Lambda de Cancelamento**

Quando `order_status = "Cancela Pedido"`, o EventBridge roteia para `sqs-pedido-cancelado.fifo`. Vamos apagar o `STATUS#antigo`, criar `STATUS#Cancelado`, e definir `item_status="CANCELADO"`.

### 4.1. Criar a Lambda

- **Function name:** `cancelar-pedido-lambda`  
- **Runtime:** Python 3.13  
- **Create function**

### 4.2. Código-Fonte

Exemplo completo:

```python
import json
import boto3

ssm = boto3.client('ssm')
dynamo = boto3.client('dynamodb')

def lambda_handler(event, context):
    try:
        table_name = ssm.get_parameter(Name="/lab/pedido/table_name")["Parameter"]["Value"]

        for record in event["Records"]:
            body_str = record["body"]
            pedido = json.loads(body_str)

            detail = pedido.get("detail", {})
            cnpj = detail.get("company", {}).get("cnpj", "NO_CNPJ")
            order_id = detail.get("order_id", "NO_ORDER_ID")
            pk = f"COMPANY#{cnpj}#ORDER#{order_id}"

            # 1) Query para obter todos os SKs
            resp = dynamo.query(
                TableName=table_name,
                KeyConditionExpression="PK = :p",
                ExpressionAttributeValues={":p": {"S": pk}}
            )
            db_items = resp.get("Items", [])

            # 2) Apagar o status antigo e criar STATUS#Cancelado
            old_status_sk = None
            for db_item in db_items:
                sk_val = db_item["SK"]["S"]
                if sk_val.startswith("STATUS#"):
                    old_status_sk = sk_val
                    break

            if old_status_sk:
                dynamo.delete_item(
                    TableName=table_name,
                    Key={
                        "PK": {"S": pk},
                        "SK": {"S": old_status_sk}
                    }
                )

            # Novo status
            dynamo.put_item(
                TableName=table_name,
                Item={
                    "PK": {"S": pk},
                    "SK": {"S": "STATUS#Cancelado"},
                    "order_status": {"S": "Cancelado"}
                }
            )

            # 3) Para cada ITEM#..., marcar item_status = "CANCELADO"
            for db_item in db_items:
                sk_val = db_item["SK"]["S"]
                if sk_val.startswith("ITEM#"):
                    dynamo.update_item(
                        TableName=table_name,
                        Key={
                            "PK": {"S": pk},
                            "SK": {"S": sk_val}
                        },
                        UpdateExpression="SET item_status = :st",
                        ExpressionAttributeValues={
                            ":st": {"S": "CANCELADO"}
                        }
                    )

        return {"statusCode": 200, "body": "Pedido(s) cancelado(s)!"}

    except Exception as e:
        print("Erro ao cancelar pedido:", e)
        return {"statusCode": 500, "body": str(e)}
```

### 4.3. Permissões IAM

Precisa de:

- `ssm:GetParameter`  
- `dynamodb:Query`, `dynamodb:DeleteItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem`  
- `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` na fila `sqs-pedido-cancelado.fifo`.

### 4.4. Adicionar Trigger SQS

1. **Configuration** > **Triggers**  
2. **Add trigger**  
3. Selecione `sqs-pedido-cancelado.fifo`  
4. **Add**

---

## **Etapa 5: Testes Manuais**

Para cada Lambda, teste no console:

1. **Lambda de Inclusão** (`incluir-pedido-lambda`):  
   - Vá em **Test** > crie um evento SQS, com body contendo `"order_status":"Pendente"`.  
   - Verifique logs e o registro no DynamoDB.

2. **Lambda de Alteração** (`alterar-pedido-lambda`):  
   - Teste com `{"order_status":"Alterar Pedido"}` e alguns itens no `items[]`.  
   - Verifique se `quantity` e `item_status="ALTERADO"` apareceram no DynamoDB.

3. **Lambda de Cancelamento** (`cancelar-pedido-lambda`):  
   - Teste com `{"order_status":"Cancela Pedido"}` e verifique se a Lambda criou `STATUS#Cancelado` e setou `item_status="CANCELADO"` nos itens.

> **Dica**: Acesse o **CloudWatch Logs** para ver detalhes (impressões, erros etc.). Acesse o **DynamoDB** > **Tables** > `pedido` para confirmar as chaves `PK` e `SK`.

---

## **Conclusão**

Nesta quarta parte, criamos:

- **Três Lambdas** para **manipular** (inserir, alterar, cancelar) pedidos, cada uma consumindo **fila SQS** distinta.  
- Uso do **Parameter Store** para obter o **nome da tabela** (`/lab/pedido/table_name`), evitando acoplamento.  
- **Políticas IAM** específicas para cada Lambda, respeitando o princípio de **menor privilégio** (acesso apenas a **GetParameter**, a operações pontuais do DynamoDB, e a fila SQS correspondente).  
- **Testes** manuais para verificar o fluxo.

Assim, **finalizamos** todo o **pipeline** EDA (Event Driven Architecture) de pedidos, do **S3/API** até a **inserção/alteração/cancelamento** no DynamoDB. Parabéns por concluir todo o **Laboratório Serverless**!
