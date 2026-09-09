## Prompt

Atue como um especialista em N8N.

Crie uma automação para registrar leads recebidos por formulário.

Público:
Equipe comercial.

Ferramentas envolvidas:
Google Forms, Google Sheets e Gmail.

Fluxo:
Receber uma resposta, salvar os dados na planilha e enviar um e-mail de confirmação.

Regras:
Ignorar registros sem e-mail válido.

Explique quais nós do N8N devem ser utilizados e a lógica de funcionamento do workflow.

## Automação

Perfeito! Vou atuar como especialista em N8N para criar uma automação robusta e profissional para captura de leads via Google Forms, com registro em planilha e envio de e-mail de confirmação.

---

## 🧩 **Arquitetura da Automação**

### **Visão Geral do Workflow**

```
[Webhook] → [Google Sheets] → [IF Node] → [Gmail] → [Respond to Webhook]
                    ↓ (sem e-mail válido)
                 [Stop & Error]
```

---

## 🔧 **Nós Necessários e Configuração**

### **1. Webhook Node (Trigger)**
**Função:** Receber os dados do Google Forms quando um novo lead é submetido.

**Configuração:**
- **Método:** POST
- **Path:** `/lead-form` (personalizável)
- **Response Mode:** "On Received" (para responder imediatamente ao Forms)
- **Options:** Ativar "Include Raw Body" para garantir captura completa dos dados

**Importante:** Copie a URL gerada (ex: `https://seu-n8n.com/webhook/lead-form`) para configurar no Google Forms como "URL de resposta" (usando integração via Apps Script ou extensões).

---

### **2. Google Sheets Node**
**Função:** Salvar os dados do lead na planilha principal.

**Configuração:**
- **Operation:** Append
- **Sheet:** Nome da aba (ex: "Leads")
- **Mapeamento de Colunas:** 
  - Coluna A: Timestamp ({{ $json.timestamp }})
  - Coluna B: Nome ({{ $json.nome }})
  - Coluna C: E-mail ({{ $json.email }})
  - Coluna D: Telefone ({{ $json.telefone }})
  - Coluna E: Empresa ({{ $json.empresa }})
  - Coluna F: Data de cadastro ({{ new Date().toISOString() }})
  - Coluna G: Status (padrão: "Novo Lead")

**Autenticação:** Usar OAuth2 com conta de serviço Google.

---

### **3. IF Node (Validação de E-mail)**
**Função:** Filtrar leads com e-mail inválido.

**Configuração:**
- **Conditions:** 
  - `{{ $json.email }}` exists
  - `{{ $json.email }}` matches regex: `/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/`
- **True Branch:** Continua para envio de e-mail
- **False Branch:** Envia para nó de log de erro ou interrompe

**Dica:** Use a função `includes('@')` e `.split('@')[1].includes('.')` para validação mais simples em versões antigas.

---

### **4. Gmail Node**
**Função:** Enviar e-mail de confirmação personalizado para o lead.

**Configuração:**
- **Resource:** Send Email
- **To:** `{{ $json.email }}`
- **Subject:** "Recebemos seu contato - [Sua Empresa]"
- **Body (HTML):**
```html
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"></head>
<body>
  <h2>Olá {{ $json.nome }},</h2>
  <p>Agradecemos seu interesse em nossos serviços!</p>
  <p><strong>Seus dados:</strong></p>
  <ul>
    <li>📧 E-mail: {{ $json.email }}</li>
    <li>📞 Telefone: {{ $json.telefone }}</li>
    <li>🏢 Empresa: {{ $json.empresa || 'Não informado' }}</li>
  </ul>
  <p>Nossa equipe comercial entrará em contato em até 24h úteis.</p>
  <p>Atenciosamente,<br><strong>Time Comercial</strong></p>
  <hr>
  <small>Esta é uma mensagem automática, não responda este e-mail.</small>
</body>
</html>
```
- **Options:** Ativar "HTML Body" e "Text Body" (fallback).

---

### **5. Respond to Webhook Node**
**Função:** Retornar status de sucesso ao Google Forms.

**Configuração:**
- **Response Code:** 200
- **Response Body:** `{ "status": "success", "message": "Lead cadastrado com sucesso!" }`
- **Options:** Ativar "Send Headers" com Content-Type: application/json

---

### **6. (Opcional) Error Trigger & Slack/Email Node**
**Função:** Notificar a equipe de TI se o fluxo falhar.

**Configuração:**
- Conectar o "Error Output" do Webhook a um nó **Gmail** ou **Slack** para alerta.
- Mensagem: `"❌ Falha no cadastro de lead: {{ $error.message }}"`

---

## 🧠 **Lógica de Funcionamento (Passo a Passo)**

| Etapa | Nó | Ação | Detalhes Técnicos |
|-------|-----|------|-------------------|
| 1 | Webhook | Recebe payload do Forms | Converte dados para JSON, identifica campos via `$json.fieldName` |
| 2 | Google Sheets | Insere nova linha | Usa "Append" para adicionar no final da planilha. Retorna ID da linha para logging |
| 3 | IF Node | Valida e-mail | Regex testa formato válido. Se falso, envia para "Stop" ou "Error" |
| 4 | Gmail | Dispara e-mail | Autenticação via OAuth2 com conta de serviço. Rate limit: ~500 e-mails/dia |
| 5 | Respond to Webhook | Confirma recebimento | Resposta HTTP 200 para o Forms evitar timeout (Forms espera resposta em até 30s) |

---

## 🛡️ **Boas Práticas e Ajustes Finos**

### **Campos do Forms (Mapeamento)**
- Use **"Custom Fields"** no Google Forms para garantir nomes consistentes:
  - `entry.123456789` → Mapeie para `nome`, `email`, etc. no nó "Set" (se necessário).

### **Tratamento de Erros**
- **Timeout:** Configurar Webhook com "Response Mode: Last Node" para responder apenas após salvar e enviar e-mail.
- **E-mail inválido:** Adicionar nó **"Stop and Error"** com mensagem clara.
- **Planilha cheia:** Configurar "Continue on Fail" para não travar o fluxo.

### **Segurança**
- Use **Webhook com autenticação** (Basic Auth ou Header Token) para evitar spam.
- Armazene credenciais no **"Credentials"** do N8N (nunca hardcoded).

### **Performance**
- Para muitos leads (100+/dia), ative **"Execute Once"** ou use **"Queue"** com Redis.
- Adicione um nó **"Wait"** (1-2 segundos) entre Google Sheets e Gmail para evitar rate limit da API.

---

## 📋 **Exemplo de Payload Recebido**

```json
{
  "timestamp": "2026-09-09T10:00:00Z",
  "nome": "João Silva",
  "email": "joao@empresa.com",
  "telefone": "(11) 99999-9999",
  "empresa": "Empresa XYZ",
  "mensagem": "Gostaria de saber mais sobre os planos."
}
```

---

## 🚀 **Fluxo Final no N8N (Visual)**

```
[Webhook] 
    ↓
[Set Node] (opcional - renomear campos)
    ↓
[Google Sheets] (salvar lead)
    ↓
[IF Node] (email válido?)
    ├── ✅ True → [Gmail] → [Respond to Webhook] → ✅ Sucesso
    └── ❌ False → [Stop Node] → ❌ Erro: "E-mail inválido"
```

---

## ⚠️ **Possíveis Problemas e Soluções**

| Problema | Solução |
|----------|---------|
| Forms não envia dados | Verificar URL do webhook e método POST |
| E-mail cai no spam | Usar SPF/DKIM configurado, incluir texto e versão em texto plano |
| Planilha com colunas erradas | Usar "Google Sheets Update" com Row Number para mapear colunas |
| Timeout do Forms | Reduzir processamento com "Execute Once" ou assíncrono via "Webhook Response" |

---

## 📈 **Melhorias Futuras (Opcional)**

- Adicionar **"Webhook>IF>Google Sheets"** para leads duplicados (validar e-mail já existe).
- Integrar com **CRM (HubSpot, Pipedrive)** via API.
- Enviar notificação para o time comercial via **Slack** ou **WhatsApp**.
- Adicionar **"Email Marketing"** via SendGrid/Mailchimp para nutrição automática.

---

Se precisar de ajuda com a configuração detalhada de algum nó específico ou com a integração via Google Apps Script para conectar o Forms ao Webhook, é só perguntar! 🚀

