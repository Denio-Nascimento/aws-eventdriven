# **Documentação – Single Table Design no DynamoDB**

Esta documentação descreve como armazenar, em uma única tabela do DynamoDB, as informações de pedido, empresa (fornecedor), cliente, itens e informações adicionais (pagamento, envio, etc.). Também explica como o GSI (Global Secondary Index) é utilizado para agilizar a área de picking e identificar itens sem estoque.

---

## **Estrutura de Chaves (PK e SK)**

### **Partition Key (PK):**

**Formato:** `COMPANY#<cnpj>#ORDER#`\<order\_id>

### **Motivações:**

- Garante que cada pedido seja único e isolado por empresa/fornecedor.
- Evita colisão de `order_id` entre empresas diferentes.
- Permite a consulta de todos os sub-itens (Status, Itens, Cliente, etc.) relacionados a esse pedido.

### **Sort Key (SK):**

A `SK` varia de acordo com o “tipo de registro” ou “domínio” que estamos mapeando dentro do mesmo pedido.

### **Principais SKs (exemplos):**

| **SK**                  | **Uso/Significado**                                                                                   |
|-------------------------|------------------------------------------------------------------------------------------------------|
| `META#<cnpj>`           | Cabeçalho do pedido (nome da empresa, data, etc.).                                                    |
| `STATUS#<order_status>` | Indica o status atual do pedido (Pendente, AguardandoEstoque, etc.).                                  |
| `ITEM#<product_id>`     | Representa cada produto presente no pedido.                                                           |
| `CUSTOMER#<cpf>`        | Vínculo ao cliente responsável pelo pedido.                                                           |
| `SHIPPING#<city>#<date>`| Informações de envio (cidade e data de entrega prevista).                                              |
| `PAYMENT#<payment_method>` | Informações de pagamento (método, valor, transação).                                               |

---

## **Documentação dos Campos de Exemplo**

### **1. Cabeçalho do Pedido (META#\*\*\*\*)**

- **PK:** `COMPANY#12.345.678/0001-99#ORDER#ORD-20231001-0001`
- **SK:** `META#12.345.678/0001-99`

**Atributos Exemplo:**

```json
{
  "order_id": "ORD-20231001-0001",
  "company_cnpj": "12.345.678/0001-99",
  "company_name": "Tech Solutions Ltda",
  "order_date": "2023-10-01T10:30:00Z",
  "notes": "Entregar no período da tarde."
}
```

### **2. Status do Pedido (STATUS#\<order\_status>)**

- **PK:** `COMPANY#<cnpj>#ORDER#<order_id>`
- **SK:** `STATUS#Pendente`

**Atributos Exemplo:**

```json
{
  "order_id": "ORD-20231001-0001",
  "order_status": "Pendente"
}
```

**Uso:** Saber em que fase o pedido está (Pendente, AguardandoSeparacao, etc.).

### **3. Itens do Pedido (ITEM#\<product\_id>)**

- **PK:** `COMPANY#<cnpj>#ORDER#<order_id>`
- **SK:** `ITEM#<product_id>`

**Atributos Típicos:** `product_id`, `quantity`, `unit_price`, `description`, `item_status`, etc.

**Configuração de GSI2:**

- **GSI2-PK:** `ITEMSTATUS#<item_status>` (por exemplo, `ITEMSTATUS#AguardandoEstoque`)
- **GSI2-SK:** `COMPANY#<cnpj>#ORDER#<order_id>#ITEM#<product_id>`

**Objetivo:**

- Localizar rapidamente itens que estão sem estoque (`item_status = "SemEstoque"`).
- Agilizar a área de picking para montar pedidos (`item_status = "AguardandoSeparacao"`).

**Exemplo:**

```json
{
  "PK": "COMPANY#12.345.678/0001-99#ORDER#ORD-20231001-0001",
  "SK": "ITEM#PRD-1001",
  "GSI2-PK": "ITEMSTATUS#AguardandoEstoque",
  "GSI2-SK": "COMPANY#12.345.678/0001-99#ORDER#ORD-20231001-0001#ITEM#PRD-1001",
  "product_id": "PRD-1001",
  "description": "Smartphone XYZ",
  "quantity": 1,
  "unit_price": 2500.00
}
```

### **4. Cliente (CUSTOMER#\*\*\*\*)**

- **PK:** `COMPANY#<cnpj>#ORDER#<order_id>`
- **SK:** `CUSTOMER#<cpf>`

**Atributos Exemplo:**

```json
{
  "first_name": "Mariana",
  "last_name": "Alves",
  "cpf": "123.456.789-00",
  "email": "mariana.alves@example.com",
  "phone": "+55 21 91234-5678",
  "address": {
    "street": "Avenida das Américas",
    "number": "5000",
    "complement": "Bloco 2, Apt 301",
    "neighborhood": "Barra da Tijuca",
    "city": "Rio de Janeiro",
    "state": "RJ",
    "zip_code": "22640-102"
  }
}
```

### **5. Informações de Envio (SHIPPING#********#********)**

- **PK:** `COMPANY#<cnpj>#ORDER#<order_id>`
- **SK:** `SHIPPING#Rio de Janeiro#2023-10-03`

**Atributos Exemplo:**

```json
{
  "shipping_method": "Express",
  "shipping_cost": 50.00,
  "city": "Rio de Janeiro",
  "expected_delivery_date": "2023-10-03"
}
```

### **6. Pagamento (PAYMENT#\<payment\_method>)**

- **PK:** `COMPANY#<cnpj>#ORDER#<order_id>`
- **SK:** `PAYMENT#CartaoDeCredito`

**Atributos Exemplo:**

```json
{
  "payment_method": "CartaoDeCredito",
  "transaction_id": "TXN-9876543210",
  "amount": 2800.00
}
```

---

## **GSI de Inversão (Opcional)**

### **Configuração:**

- **GSI1-PK:** `SK`
- **GSI1-SK:** `PK`

### **Exemplos de Consultas:**

1. **Listar pedidos em “STATUS#AguardandoSeparacao”:**
   - **Consulta:** `GSI1-PK = "STATUS#AguardandoSeparacao"`
2. **Listar pedidos de um cliente específico:**
   - **Consulta:** `GSI1-PK = "CUSTOMER#123.456.789-00"`

---

## **Exemplo Completo de Itens (Um Pedido)**

Para o pedido:

- `order_id = ORD-20231001-0001`
- `cnpj = 12.345.678/0001-99`
- `order_status = "Pendente"`
- `item_status inicial = "AguardandoEstoque"`

### **Entradas na Tabela:**

1. **Cabeçalho:**
   - **PK:** `COMPANY#12.345.678/0001-99#ORDER#ORD-20231001-0001`
   - **SK:** `META#12.345.678/0001-99`
2. **Status:**
   - **SK:** `STATUS#Pendente`
3. **Itens:**
   - **Item 1:**
     - **SK:** `ITEM#PRD-1001`
     - **GSI2-PK:** `ITEMSTATUS#AguardandoEstoque`
     - **GSI2-SK:** `COMPANY#12.345.678/0001-99#ORDER#ORD-20231001-0001#ITEM#PRD-1001`
   - **Item 2:**
     - **SK:** `ITEM#PRD-2002`
     - **GSI2-PK:** `ITEMSTATUS#AguardandoEstoque`
     - **GSI2-SK:** `COMPANY#12.345.678/0001-99#ORDER#ORD-20231001-0001#ITEM#PRD-2002`
4. **Cliente:**
   - **SK:** `CUSTOMER#123.456.789-00`
5. **Envio:**
   - **SK:** `SHIPPING#Rio de Janeiro#2023-10-03`
6. **Pagamento:**
   - **SK:** `PAYMENT#CartaoDeCredito`

---

## **Conclusão**

- **PK:** `COMPANY#<cnpj>#ORDER#<order_id>` isola o pedido por empresa.
- **SK com prefixos semânticos** (META, STATUS, ITEM, CUSTOMER, SHIPPING, PAYMENT) organiza os registros.
- **GSI2 (ITEMSTATUS)** ajuda a controlar:
  - **GSI2-PK:** `ITEMSTATUS#<item_status>`
  - **GSI2-SK:** `COMPANY#<cnpj>#ORDER#<order_id>#ITEM#<product_id>`

### **Principais Consultas Disponíveis:**

- Listar itens pendentes ou sem estoque via GSI2.
- Listar pedidos por status, cliente, cidade, método de pagamento via GSI de inversão.
- Evitar colisões de `order_id` entre empresas.
- Otimizar o picking e separação.

Este modelo Single Table Design atende às regras de negócio, segmenta por empresa, facilita a busca por status, picking de itens, e permite evoluir com novos SKs se necessário.

