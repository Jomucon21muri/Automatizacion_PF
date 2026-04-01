# Sesión 12: IA en Automatización Financiera

## Objetivos

- Integrar modelos de IA en workflows
- Implementar procesamiento de lenguaje natural (NLP)
- Automatizar decisiones con Machine Learning
- Usar GPT para tareas financieras

## OpenAI GPT en workflows financieros

### Configuración Básica

```javascript
// En n8n/Make/Zapier con HTTP Request
const openai_api_key = process.env.OPENAI_API_KEY;

const response = await fetch('https://api.openai.com/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${openai_api_key}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    model: 'gpt-4',
    messages: [
      {
        role: 'system',
        content: 'You are a financial analyst assistant.'
      },
      {
        role: 'user',
        content: 'Analyze this transaction: ...'
      }
    ],
    temperature: 0.3  // Lower for consistent financial analysis
  })
});

const data = await response.json();
const analysis = data.choices[0].message.content;
```

### Caso 1: Extracción de datos de facturas (OCR + GPT)

```javascript
// Workflow: PDF Invoice → Text → GPT → Structured Data

// Step 1: Convert PDF to text (usando OCR)
const pdfText = await convertPDFToText(invoice_pdf);

// Step 2: Extract structured data with GPT
const prompt = `
Extract the following information from this invoice text and return ONLY a valid JSON object:

Invoice Text:
${pdfText}

Required JSON fields:
{
  "invoice_number": "",
  "date": "YYYY-MM-DD",
  "vendor_name": "",
  "vendor_address": "",
  "total_amount": 0.00,
  "currency": "",
  "line_items": [
    {
      "description": "",
      "quantity": 0,
      "unit_price": 0.00,
      "total": 0.00
    }
  ],
  "tax_amount": 0.00,
  "payment_terms": ""
}

Return only the JSON, no markdown formatting or explanation.
`;

const gptResponse = await callGPT(prompt);
const invoiceData = JSON.parse(gptResponse);

// Step 3: Validate and create in accounting system
if (validateInvoiceData(invoiceData)) {
  await createQuickBooksInvoice(invoiceData);
}
```

### Caso 2: Clasificación inteligente de transacciones

```javascript
// Clasificar transacciones automáticamente con contexto

async function classifyTransaction(transaction) {
  const prompt = `
Classify this transaction into the most appropriate category for accounting purposes.

Transaction:
- Description: ${transaction.description}
- Amount: $${transaction.amount}
- Merchant: ${transaction.merchant}
- Date: ${transaction.date}

Available categories:
- Office Supplies
- Software & Subscriptions
- Marketing & Advertising
- Travel & Entertainment
- Utilities
- Salaries & Benefits
- Professional Services
- Equipment & Hardware
- Rent & Facilities
- Other

Return ONLY the category name, nothing else.
`;

  const category = await callGPT(prompt);
  
  return {
    ...transaction,
    category: category.trim(),
    auto_classified: true
  };
}

// Aplicar a batch de transacciones
const transactions = await getUncategorizedTransactions();
const classified = await Promise.all(
  transactions.map(tx => classifyTransaction(tx))
);

await bulkUpdateTransactions(classified);
```

### Caso 3: Generación de resúmenes ejecutivos

```javascript
async function generateExecutiveSummary(financialData) {
  const prompt = `
You are a CFO assistant. Generate a concise executive summary of this financial data.

Financial Data:
${JSON.stringify(financialData, null, 2)}

The summary should include:
1. Key highlights (3-5 bullet points)
2. Areas of concern (if any)
3. Recommendations (2-3 action items)

Keep it under 200 words, professional tone.
`;

  const summary = await callGPT(prompt);
  
  return summary;
}

// Uso en workflow
const monthlyData = await getMonthlyFinancials();
const summary = await generateExecutiveSummary(monthlyData);

await sendSlackMessage('#executive-team', summary);
await emailCEO(summary);
```

### Caso 4: Análisis de contratos

```javascript
async function analyzeContract(contractText) {
  const prompt = `
Analyze this financial contract and extract key terms:

Contract:
${contractText}

Extract:
1. Contract value and payment terms
2. Duration and renewal terms
3. Key obligations and deliverables
4. Termination clauses
5. Risk factors or unusual terms

Format as structured JSON.
`;

  const analysis = await callGPT(prompt);
  const parsed = JSON.parse(analysis);
  
  // Store in CRM
  await updateCRMWithContractTerms(parsed);
  
  // Alert if risky terms detected
  if (parsed.risk_factors && parsed.risk_factors.length > 0) {
    await alertLegalTeam({
      contract_id: contractId,
      risks: parsed.risk_factors
    });
  }
  
  return parsed;
}
```

## NLP para análisis de sentimiento financiero

### Análisis de earnings calls

```python
from transformers import pipeline

# Load FinBERT
finbert = pipeline("sentiment-analysis", model="ProsusAI/finbert")

def analyze_earnings_call(transcript):
    # Split into sentences
    sentences = transcript.split('.')
    
    sentiments = []
    for sentence in sentences:
        if len(sentence.strip()) > 10:
            result = finbert(sentence)[0]
            sentiments.append({
                'text': sentence,
                'sentiment': result['label'],
                'score': result['score']
            })
    
    # Calculate overall sentiment
    positive = sum(1 for s in sentiments if s['sentiment'] == 'positive')
    negative = sum(1 for s in sentiments if s['sentiment'] == 'negative')
    neutral = len(sentiments) - positive - negative
    
    overall_score = (positive - negative) / len(sentiments)
    
    return {
        'overall_sentiment': 'POSITIVE' if overall_score > 0.1 else 'NEGATIVE' if overall_score < -0.1 else 'NEUTRAL',
        'confidence': overall_score,
        'breakdown': {
            'positive': positive,
            'neutral': neutral,
            'negative': negative
        },
        'key_negative_sentences': [s['text'] for s in sentiments if s['sentiment'] == 'negative'][:5]
    }

# Workflow integration
earnings_transcript = fetch_earnings_call_transcript('AAPL', 'Q4-2024')
analysis = analyze_earnings_call(earnings_transcript)

if analysis['overall_sentiment'] == 'NEGATIVE':
    alert_trading_team({
        'stock': 'AAPL',
        'sentiment': 'negative',
        'concerns': analysis['key_negative_sentences']
    })
```

## ML para detección de fraude

### Random forest classifier

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# Prepare training data
X = df[[
    'amount',
    'hour_of_day',
    'day_of_week',
    'merchant_category',
    'distance_from_home',
    'transaction_velocity',  # txns in last hour
    'amount_deviation'       # vs customer average
]].values

y = df['is_fraud'].values

# Train model
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X_train, y_train)

# Predict fraud probability
def predict_fraud(transaction):
    features = extract_features(transaction)
    fraud_probability = clf.predict_proba([features])[0][1]
    
    return {
        'fraud_probability': fraud_probability,
        'fraud_predicted': fraud_probability > 0.7,
        'risk_level': 'HIGH' if fraud_probability > 0.7 else 'MEDIUM' if fraud_probability > 0.4 else 'LOW'
    }

# Real-time scoring in workflow
for transaction in new_transactions:
    fraud_score = predict_fraud(transaction)
    
    if fraud_score['fraud_predicted']:
        block_transaction(transaction)
        alert_fraud_team(transaction, fraud_score)
    elif fraud_score['risk_level'] == 'MEDIUM':
        flag_for_review(transaction, fraud_score)
```

## Previsión con redes neuronales

### LSTM para predicción de revenue

```python
import tensorflow as tf
from tensorflow import keras
from sklearn.preprocessing import MinMaxScaler

# Prepare time series data
scaler = MinMaxScaler()
revenue_scaled = scaler.fit_transform(monthly_revenue.values.reshape(-1, 1))

# Create sequences
def create_sequences(data, seq_length=12):
    X, y = [], []
    for i in range(len(data) - seq_length):
        X.append(data[i:i+seq_length])
        y.append(data[i+seq_length])
    return np.array(X), np.array(y)

X, y = create_sequences(revenue_scaled)

# Build LSTM model
model = keras.Sequential([
    keras.layers.LSTM(50, activation='relu', input_shape=(12, 1)),
    keras.layers.Dense(25),
    keras.layers.Dense(1)
])

model.compile(optimizer='adam', loss='mse')
model.fit(X, y, epochs=100, verbose=0)

# Predict next 6 months
last_sequence = revenue_scaled[-12:]
predictions = []

for _ in range(6):
    pred = model.predict(last_sequence.reshape(1, 12, 1), verbose=0)
    predictions.append(pred[0, 0])
    last_sequence = np.append(last_sequence[1:], pred)

# Inverse transform
predicted_revenue = scaler.inverse_transform(np.array(predictions).reshape(-1, 1))

# Alert if below target
target_revenue = 500000  # $500k/month
if predicted_revenue.mean() < target_revenue:
    send_alert(f"Forecasted revenue ({predicted_revenue.mean():.0f}) below target ({target_revenue})")
```

## Chatbot financiero con LangChain

```python
from langchain import OpenAI, SQLDatabase, SQLDatabaseChain

# Connect to financial database
db = SQLDatabase.from_uri("postgresql://user:password@localhost/finance_db")

# Create LLM chain
llm = OpenAI(temperature=0)
db_chain = SQLDatabaseChain.from_llm(llm, db)

# Query in natural language
def ask_financial_question(question):
    try:
        result = db_chain.run(question)
        return result
    except Exception as e:
        return f"Error: {str(e)}"

# Examples
questions = [
    "What was our total revenue last month?",
    "Which customer spent the most?",
    "Show me all transactions over $10,000 in March",
    "What's our current cash balance?"
]

for q in questions:
    answer = ask_financial_question(q)
    print(f"Q: {q}\nA: {answer}\n")

# Integrate with Slack
@app.event("message")
def handle_message(event):
    question = event['text']
    if question.startswith("@finbot"):
        answer = ask_financial_question(question.replace("@finbot", ""))
        say(answer)
```

## Caso completo: assistant financiero inteligente

```mermaid
graph TD
    A[User Query: Slack/Email] --> B[NLP: Classify Intent]
    B --> C{Intent Type}
    C -->|Data Query| D[LangChain SQL]
    C -->|Analysis Request| E[GPT-4 Analysis]
    C -->|Report Request| F[Generate Report]
    C -->|Alert Query| G[Check Alert History]
    D --> H[Format Response]
    E --> H
    F --> H
    G --> H
    H --> I[Send to User]
    I --> J[Log Interaction]
```

### Implementación

```python
class FinancialAssistant:
    def __init__(self):
        self.llm = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
        self.db_chain = SQLDatabaseChain.from_llm(self.llm, db)
        
    def classify_intent(self, query):
        prompt = f"""
        Classify this financial query into one of these categories:
        - data_query (asking for specific numbers/data)
        - analysis (asking for insights/interpretation)
        - report (requesting a formatted report)
        - alert (asking about alerts/notifications)
        
        Query: {query}
        
        Response with ONLY the category name.
        """
        
        return self.llm(prompt).strip()
    
    def handle_query(self, query):
        intent = self.classify_intent(query)
        
        if intent == 'data_query':
            return self.db_chain.run(query)
        
        elif intent == 'analysis':
            data = self.fetch_relevant_data(query)
            analysis_prompt = f"""
            As a financial analyst, analyze this data and provide insights:
            
            Data: {data}
            Question: {query}
            
            Provide a concise analysis with key findings and recommendations.
            """
            return self.llm(analysis_prompt)
        
        elif intent == 'report':
            return self.generate_report(query)
        
        else:
            return "I'm not sure how to help with that. Try asking about data, analysis, or reports."
    
    def generate_report(self, query):
        # Extract timeframe and type from query
        report_data = self.fetch_report_data(query)
        pdf = generate_pdf_report(report_data)
        
        return {
            'message': 'Report generated',
            'file': pdf,
            'summary': summarize_report(report_data)
        }

# Usage
assistant = FinancialAssistant()

# Slack integration
@app.message()
def respond_to_query(message, say):
    query = message['text']
    response = assistant.handle_query(query)
    say(response)
```

## Ejercicio práctico

**Construir**: Asistente financiero que:

1. Recibe preguntas en lenguaje natural (Slack)
2. Consulta base de datos SQL
3. Genera análisis con GPT-4
4. Detecta anomalías con ML
5. Envía reportes formateados

## Recursos

- [OpenAI API Docs](https://platform.openai.com/docs)
- [LangChain](https://python.langchain.com/)
- [FinBERT](https://huggingface.co/ProsusAI/finbert)
- [TensorFlow](https://www.tensorflow.org/)

## Resumen

✅ GPT para extracción y análisis  
✅ NLP para sentiment financiero  
✅ ML para detección de fraude  
✅ LSTM para forecasting  
✅ Chatbot financiero con LangChain  

**Próxima sesión**: Proyecto Integrador Final
