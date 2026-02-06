# Invoice Processing Automation Workflow

## Overview
This automation workflow processes invoices from Google Drive, extracts key information using AI, logs data to Google Sheets, and sends automated email notifications. The system uses n8n (workflow automation tool) to orchestrate the entire process.

## Workflow Architecture
```
Google Drive → Download PDF → Extract Data → AI Analysis → 
Sheet Logging → Email Composition → Email Sending
```
![Automation Flow](./images/automation-flow.png)

## Components Breakdown

### 1. Google Drive Trigger
**Node Type:** Google Drive Trigger  
**Configuration:** Monitors a specific folder for new invoice PDF files

**Setup:**
- Connect your Google Drive account
- Select the folder to monitor (e.g., "Invoices/Incoming")
- Set trigger on "File Updated" or "File Created"
- Filter for PDF files only

**Example Configuration:**
```json
{
  "folderId": "your-folder-id",
  "event": "fileCreated",
  "fileType": "application/pdf"
}
```

### 2. Download File
**Node Type:** Google Drive - Download File  
**Purpose:** Downloads the PDF invoice from Google Drive to n8n for processing

**Configuration:**
- Input: File ID from trigger
- Output: Binary file data
- File name preservation: Enabled
- Binary Data Path: `data`

### 3. Extract from File (PDF Parser)
**Node Type:** Extract from File  
**Purpose:** Converts PDF binary data to text for AI processing

**Configuration:**
- Input Binary Field: `data` (from Download File node)
- Output Format: Text
- Extraction Method: PDF text extraction

**Output:**
- Plain text content of the invoice
- Metadata (page count, file info)

**Important Notes:**
- Handles multi-page PDFs
- Preserves text structure
- Extracts tables and formatted data

### 4. Information Extractor (AI/LLM Node)
**Node Type:** AI Agent / LLM Integration (Google Gemini Chat Model or Ollama)

#### Option A: Using Ollama (Local/Self-hosted)

**Model Configuration:**
- Model: `llama3` or `mistral`
- Temperature: 0.1 (for consistent extraction)
- Max Tokens: 2000

**Prompt Template:**
```
Extract the following information from this invoice:

Invoice Text:
{{$json.text}}

Return ONLY a valid JSON object with these exact fields:
{
  "invoice_number": "",
  "invoice_date": "",
  "due_date": "",
  "client_name": "",
  "client_email": "",
  "client_phone": "",
  "subtotal": "",
  "tax": "",
  "grand_total": "",
  "service_provider": "",
  "services": []
}

Extract all amounts as numbers without currency symbols.
```

**Ollama Setup:**
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model
ollama pull llama3

# Run Ollama server
ollama serve
```

**n8n HTTP Request Configuration:**
```json
{
  "url": "http://localhost:11434/api/generate",
  "method": "POST",
  "body": {
    "model": "llama3",
    "prompt": "{{$node.prompt}}",
    "stream": false
  }
}
```

#### Option B: Using Google Gemini

**Configuration:**
- Model: `gemini-pro`
- API Key: Set in n8n credentials
- System Prompt: Same extraction prompt as above

**Expected Output Structure:**
```json
{
  "invoice_number": "INV-2026-0402",
  "invoice_date": "2026-02-04",
  "due_date": "2026-02-19",
  "client_name": "Global Enterprise Solutions Inc.",
  "client_email": "sarvesh@gmai.com",
  "client_phone": "9321555257",
  "subtotal": "49000.00",
  "tax": "8820.00",
  "grand_total": "57820.00",
  "service_provider": "XYZ Pvt. Limited",
  "services": [
    "Enterprise Web Architecture",
    "Custom Software Development",
    "Cloud Infrastructure & Security",
    "Maintenance & Technical Support"
  ]
}
```

### 5. Append Row in Sheet
**Node Type:** Google Sheets - Append  
**Purpose:** Logs extracted invoice data to Google Sheets for record-keeping

**Google Sheet Structure:**

| Column | Field Name | Data Type | Example |
|--------|------------|-----------|---------|
| A | Invoice Number | Text | INV-2026-0402 |
| B | Invoice Date | Date | 2026-02-04 |
| C | Due Date | Date | 2026-02-19 |
| D | Client Name | Text | Global Enterprise Solutions Inc. |
| E | Client Email | Email | sarvesh@gmai.com |
| F | Client Phone | Text | 9321555257 |
| G | Subtotal | Currency | 49000.00 |
| H | Tax | Currency | 8820.00 |
| I | Grand Total | Currency | 57820.00 |
| J | Service Provider | Text | XYZ Pvt. Limited |
| K | Services | Text | Enterprise Web Architecture, Custom Software Development... |
| L | Processing Date | Timestamp | 2026-02-06 10:41:00 |
| M | Status | Text | Processed |

**Field Mapping:**
```javascript
{
  "Invoice Number": "={{$json.invoice_number}}",
  "Invoice Date": "={{$json.invoice_date}}",
  "Due Date": "={{$json.due_date}}",
  "Client Name": "={{$json.client_name}}",
  "Client Email": "={{$json.client_email}}",
  "Client Phone": "={{$json.client_phone}}",
  "Subtotal": "={{$json.subtotal}}",
  "Tax": "={{$json.tax}}",
  "Grand Total": "={{$json.grand_total}}",
  "Service Provider": "={{$json.service_provider}}",
  "Services": "={{$json.services.join(', ')}}",
  "Processing Date": "={{$now}}",
  "Status": "Processed"
}
```

### 6. Message a Model (Email Draft Composition)
**Node Type:** AI Agent / LLM  
**Purpose:** Generates a professional email notification about the invoice

**Prompt Template:**
```
Compose a professional email notification for a new invoice received.

Invoice Details:
- Invoice Number: {{$json.invoice_number}}
- Client: {{$json.client_name}}
- Amount: ${{$json.grand_total}}
- Due Date: {{$json.due_date}}
- Service Provider: {{$json.service_provider}}

Create an email with:
1. Subject line (concise and action-oriented)
2. Email body (professional, includes all key details, call-to-action)

Format the response as JSON:
{
  "subject": "subject line here",
  "body": "email body here"
}

The email should be formal but friendly, and remind the recipient to review and process the invoice by the due date.
```

**Expected LLM Output:**
```json
{
  "subject": "[Action Required] New Invoice Received - INV-2026-0402 - XYZ Pvt. Limited",
  "body": "Dear Team,\n\nWe have received a new invoice that requires your attention.\n\nExecutive Summary:\n\nAmount: $57,820.00\nClient: XYZ Pvt. Limited\nDue Date: February 19, 2026\n\nFull Details:\n\nInvoice Number: INV-2026-0402\nInvoice Date: February 4, 2026\nService Provider: XYZ Pvt. Limited\n\nPlease review the invoice details and ensure payment is processed by the due date to avoid any late fees.\n\nIf you have any questions or need additional information, please don't hesitate to reach out.\n\nBest regards,\nAutomated Invoice Processing System"
}
```

### 7. Code in JavaScript (JSON Formatting)
**Node Type:** Code Node (JavaScript)  
**Purpose:** Parses the LLM response and formats it properly for the email node

**JavaScript Code:**
```javascript
// Get the LLM response
const llmResponse = items[0].json.response || items[0].json.text;

// Parse the JSON from LLM (handle if it's wrapped in markdown code blocks)
let emailData;
try {
  // Remove markdown code blocks if present
  const cleanedResponse = llmResponse.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
  emailData = JSON.parse(cleanedResponse);
} catch (error) {
  // Fallback parsing if JSON is malformed
  const subjectMatch = llmResponse.match(/"subject"\s*:\s*"([^"]*)"/);
  const bodyMatch = llmResponse.match(/"body"\s*:\s*"([^"]*)"/s);
  
  emailData = {
    subject: subjectMatch ? subjectMatch[1] : 'New Invoice Received',
    body: bodyMatch ? bodyMatch[1].replace(/\\n/g, '\n') : llmResponse
  };
}

// Get invoice data from previous node
const invoiceData = items[0].json;

// Return formatted data for email node
return [
  {
    json: {
      subject: emailData.subject,
      body: emailData.body,
      to: invoiceData.client_email || 'accounts@company.com',
      invoiceNumber: invoiceData.invoice_number,
      grandTotal: invoiceData.grand_total,
      dueDate: invoiceData.due_date
    }
  }
];
```

**Output Format:**
```json
{
  "subject": "[Action Required] New Invoice Received - INV-2026-0402 - XYZ Pvt. Limited",
  "body": "Dear Team,\n\nWe have received a new invoice...",
  "to": "sarvesh@gmai.com",
  "invoiceNumber": "INV-2026-0402",
  "grandTotal": "57820.00",
  "dueDate": "2026-02-19"
}
```

### 8. Send a Message (Email Notification)
**Node Type:** Gmail / Send Email  
**Purpose:** Sends the formatted email notification to stakeholders

**Configuration:**
- To: `={{$json.to}}`
- Subject: `={{$json.subject}}`
- Email Type: Text or HTML
- Body: `={{$json.body}}`

**Optional Enhancements:**
- CC: Finance team email
- BCC: Archive email
- Attachments: Original PDF invoice (from binary data)
- Reply-To: Accounts payable email

**HTML Email Template (Optional):**
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; }
    .header { background: #4285f4; color: white; padding: 20px; }
    .content { padding: 20px; }
    .invoice-details { background: #f5f5f5; padding: 15px; margin: 20px 0; }
    .footer { font-size: 12px; color: #666; margin-top: 30px; }
  </style>
</head>
<body>
  <div class="header">
    <h2>New Invoice Notification</h2>
  </div>
  <div class="content">
    <p>Dear Team,</p>
    <p>A new invoice has been received and processed automatically.</p>
    
    <div class="invoice-details">
      <h3>Executive Summary:</h3>
      <p><strong>Amount:</strong> ${{$json.grandTotal}}</p>
      <p><strong>Invoice Number:</strong> {{$json.invoiceNumber}}</p>
      <p><strong>Due Date:</strong> {{$json.dueDate}}</p>
    </div>
    
    <p>Please review and process the invoice by the due date.</p>
    
    <div class="footer">
      <p>This is an automated message from the Invoice Processing System.</p>
    </div>
  </div>
</body>
</html>
```

## Installation & Setup

1. **Install n8n:**
```bash
   npm install n8n -g
```

2. **Configure Google Drive:**
   - Create OAuth2 credentials in Google Cloud Console
   - Add credentials to n8n

3. **Configure Google Sheets:**
   - Create a new spreadsheet with the structure above
   - Add Google Sheets credentials to n8n

4. **Configure AI Model:**
   - For Ollama: Install and run locally
   - For Google Gemini: Add API key to n8n credentials

5. **Configure Email:**
   - Add Gmail credentials to n8n
   - Enable "Less secure app access" or use App Passwords

![Automation Flow](./automated-email.png)

## Usage

1. Upload invoice PDFs to the monitored Google Drive folder
2. The workflow automatically triggers and processes the invoice
3. Data is extracted and logged to Google Sheets
4. Email notification is sent to stakeholders

## Troubleshooting

- **PDF extraction fails:** Ensure PDF is text-based, not scanned images
- **AI extraction errors:** Adjust prompt template or increase token limit
- **Email sending fails:** Check Gmail credentials and security settings

## Future Enhancements

- Add support for scanned PDFs using OCR
- Implement invoice approval workflow
- Add Slack notifications
- Create dashboard for invoice analytics
- Support multiple currencies

## License

MIT