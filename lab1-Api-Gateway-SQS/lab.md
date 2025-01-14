# Passo a Passo da Atividade Prática

Nesta atividade, você irá:

- Criar uma DLQ (Dead Letter Queue) antes de criar a fila principal no Amazon SQS.
- Criar a fila principal do Amazon SQS e associá-la à DLQ.
- Criar uma função AWS Lambda para pré-validação dos pedidos JSON.
- Configurar a função Lambda com as permissões IAM adequadas e variáveis de ambiente.
- Criar uma API REST no Amazon API Gateway para enviar pedidos.
- Configurar e testar a integração completa.

---

### **Pré-requisitos**

- **Conta AWS ativa** com permissões para criar recursos (API Gateway, SQS, Lambda e IAM).
- **Acesso à console da AWS** via navegador web.

---

## **Etapa 1: Criar a Dead Letter Queue (DLQ)**

1. **Acessar o Amazon SQS:**
   - No menu de serviços, selecione **SQS** (ou utilize a barra de pesquisa).

2. **Criar uma nova fila:**
   - Clique em **Create queue**.
   - Escolha **FIFO Queue**.

3. **Configurações da fila:**
   - **Queue Name:** `order-processing-dlq.fifo`
   - **Enable Content-Based Deduplication:** marque esta opção.

4. **Finalizar a criação:**
   - Clique em **Create Queue**.

---

## **Etapa 2: Criar a Fila Principal e Associar à DLQ**

1. **Criar uma nova fila principal:**
   - No painel do Amazon SQS, clique em **Create queue**.
   - Escolha **FIFO Queue**.

2. **Configurações da fila principal:**
   - **Queue Name:** `order-processing.fifo`
   - **Enable Content-Based Deduplication:** marque esta opção.

3. **Configurar a Dead Letter Queue:**
   - Na seção **Dead-letter queue**, selecione **Enabled**.
   - **Dead-letter queue:** selecione `order-processing-dlq.fifo`.
   - **Maximum Receives:** defina 3 (ou outro valor conforme sua necessidade).

4. **Finalizar a criação:**
   - Clique em **Create Queue**.

---

## **Etapa 3: Criar a Função Lambda de Pré-Validação**

### **3.1. Criar a Função Lambda**

1. No menu de serviços, selecione **Lambda**.
2. Clique em **Create function**.
3. Escolha **Author from scratch**.
4. Preencha os campos:
   - **Function name:** `prevalidation-lambda`
   - **Runtime:** `Python 3.13`.
5. Clique em **Create function**.

---

### **3.2. Adicionar o Código à Função Lambda**

1. No painel da função, role até a seção **Code source**.
2. Apague o código padrão e cole o seguinte código:

```python
import json
import boto3
import os

def lambda_handler(event, context):
    sqs_client = boto3.client('sqs')
    queue_url = os.getenv('SQS_FIFO_QUEUE_URL')

    try:
        # Extrair o corpo do evento
        body = json.loads(event["body"])
        
        # Verificar campos essenciais
        if "order_id" not in body or not body.get("items"):
            return {
                "statusCode": 400,
                "body": json.dumps({"message": "O campo 'order_id' ou os 'items' estão faltando."})
            }
        
        # Enviar mensagem para SQS FIFO
        response = sqs_client.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(body),
            MessageGroupId="orders-group",
            MessageDeduplicationId=body["order_id"]
        )
        
        return {
            "statusCode": 200,
            "body": json.dumps({"message": "Pedido enviado com sucesso para a fila SQS FIFO.", "sqs_response": response})
        }

    except Exception as e:
        return {
            "statusCode": 500,
            "body": json.dumps({"message": f"Erro interno: {str(e)}"})
        }
```

---

### **3.3. Adicionar Variáveis de Ambiente**

1. No painel da função Lambda, clique em **Configuration** > **Environment variables**.
2. Clique em **Edit**.
3. Clique em **Add environment variable**
4. Adicione a seguinte variável de ambiente:
   - **Key:** `SQS_FIFO_QUEUE_URL`
   - **Value:** cole a **Queue URL** obtida na fila principal `order-processing.fifo` (disponível na aba **Details** da fila).
5. Clique em **Save**.

---


### **3.4. Configurar Permissões IAM para a Função Lambda**

1. No painel da função Lambda, clique em **Configuration** > **Permissions**.
2. Clique no nome da **role** atribuída à função.
3. Clique em **Add permissions** > **Create inline policy**.
4. Selecione a aba **JSON** e cole a seguinte política com mínimos privilégios:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:<YOUR_REGION>:<YOUR_ACCOUNT_ID>:order-processing.fifo"
    }
  ]
}
```

5. Substitua `<YOUR_REGION>` pela sua região AWS (por exemplo, `us-east-1`).
6. Substitua `<YOUR_ACCOUNT_ID>` pelo ID da sua conta AWS (disponível no canto superior direito do console).
7. Clique em **Next**.
8. Em **Policy name** digite o nome da política `SendMessagePolicy`.
9. Clique em **Create policy**.

---


---

## **Etapa 4: Criar o API Gateway**

### **4.1. Criar API REST**

1. No menu de serviços, selecione **API Gateway**.
2. Clique em **Create API**.
3. Escolha **REST API** e clique em **Build**.
4. Preencha os seguintes campos:
   - **API details:** selecione `New API` 
   - **API Name:** digite `order-api`
   - **Description:** digite `API para receber pedidos JSON`
   - **Endpoint Type:** `Regional`
5. Clique em **Create API**.

---

### **4.2. Criar o Recurso e Método**

1. No painel do **API Gateway**, clique em **Create Resource**.
2. Preencha:
   - **Resource Name:** `orders`
   - **Resource Path:** `/`
3. Clique em **Create Resource**.
4. Selecione o recurso `/orders`.
5. Clique em **Create Method**.
6. Em **Method type** Escolha **POST**.
7. Configure:
   - **Integration Type:** Lambda Function.
   - **Lambda Function:** `prevalidation-lambda`
8. Clique em **Create method**.

---

### **4.3. Configurar CORS (Cross-Origin Resource Sharing)**

1. Selecione o recurso **/orders**.
2. Clique em **Enable CORS**.
3. Selecione:
   - Default 4XX
   - Default 5XX
   - Post
4. Clique em **Save**.

---

## **Etapa 5: Deploy da API**

1. Clique em **Deploy API**.
2. Selecione **[New Stage]**.
3. Preencha os seguintes campos:
   - **Stage Name:** `prod`
   - **Deployment description:** `Versão de produção`
4. Clique em **Deploy**.

---

## **Etapa 6: Testar a API**

### **6.1. Realizar um Teste direto no API GATEWAY**

1. Na console do AWS API Gateway clique **POST**.
2. Clique na ABA **Test**
3. Em **Test method** preencha os campos:
   - **Query strings:** deixe vazio, pois não estamos utilizando parâmetros na URL.
   - **Headers:** `application/json`
   - **Request Body:**

	```json
	{
	  "order_id": "TS-00123",
	  "company": {
	    "name": "Tech Solutions Ltda",
	    "cnpj": "12.345.678/0001-99"
	  },
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

### **6.2. Verificação da Fila SQS**

1. Acesse a fila `order-processing.fifo`.
2. Verifique se a mensagem foi recebida na aba **Messages available**.
3. Acesse a fila `order-processing-dlq.fifo` e verifique se há mensagens com erro.

---

## **Etapa 7: Monitoramento e Logs**

1. Acesse o console **CloudWatch**.
2. Verifique os logs gerados pela função Lambda.
3. Analise se há falhas ou exceções.

---

## **Conclusão**

Você concluiu a atividade prática, criando uma solução **serverless** com **API Gateway**, **Lambda**, **SQS FIFO** e **DLQ** para validação e processamento de pedidos JSON. Essa arquitetura permite um fluxo robusto com monitoramento de erros e tratamento de falhas.
