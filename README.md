# BenedictPDF

![Benedict Corp](https://raw.githubusercontent.com/Benedict-Corp/benedict-pdf/main/assets/logo.svg)

**Open source PDF processing for Microsoft Power Platform — watermark, password protect, and collect signatures on your own Azure infrastructure.**

[![License: MIT + Commons Clause](https://img.shields.io/badge/License-MIT%20%2B%20Commons%20Clause-blue.svg)](LICENSE)
[![Open Source](https://img.shields.io/badge/Open%20Source-Yes-brightgreen.svg)](https://github.com/Benedict-Corp/benedict-pdf)
[![Power Platform](https://img.shields.io/badge/Microsoft-Power%20Platform-742774.svg)](https://www.microsoft.com/en-us/power-platform)

[Website](https://www.benedictcorp.com/) · [Support the Project](https://pay.benedictcorp.store/b/00w3cx79a1N7cT7c1qcAo07) · [Report an Issue](https://github.com/Benedict-Corp/benedict-pdf/issues)

---

## Overview

BenedictPDF is a free, open source PDF processing toolkit built for Microsoft Power Platform. It runs as an Azure Function inside your own Azure subscription — your documents never leave your Microsoft environment.

**What it does:**
- Add text watermarks to PDFs with full control over position, color, size, and padding
- Password protect PDFs with AES-256 encryption
- Remove passwords from PDFs
- Send documents for e-signature via DocuSign (DocuSign version)

**Two versions available:**

| | BenedictPDF | BenedictPDF + DocuSign |
|---|---|---|
| Watermark | ✅ | ✅ |
| Password protection | ✅ | ✅ |
| Remove password | ✅ | ✅ |
| E-signature | ❌ | ✅ |
| DocuSign account required | ❌ | ✅ |

---

## How It Works — HTTP Actions vs Custom Connector

**The solution ships with HTTP actions by default.** This means Power Automate flows call your Azure Function directly via an HTTP request with your function URL and key. This approach works for most users and requires no additional setup beyond the Azure Function itself.

**Advanced users may prefer to create a Custom Connector.** A custom connector wraps the HTTP call into a named action that appears in Power Automate and Power Apps like any other connector — with named fields and no need to write JSON manually. Custom connectors also work in **Azure Logic Apps**, making BenedictPDF usable in enterprise automation scenarios beyond Power Platform.

To create a custom connector, refer to the [Custom Connector Setup](#part-4--custom-connector-setup-optional) section. Note that custom connectors require a **Microsoft Power Automate Premium license**.

> If you build a custom connector and use it in your flows, you will need to replace the existing HTTP actions with your connector actions and remap all parameters. Keep this in mind before deciding which approach to use.

---

## Prerequisites

Before you begin, make sure you have:

- **Microsoft 365** with Power Platform access
- **Azure subscription**
- **Microsoft Power Automate Premium or Power Apps Premium license** — required for HTTP actions and custom connectors
- **DocuSign account** - please note that current setup is using "Docusign Demo" actions. Replace to production DocuSign connector after testing

---

## Part 1 — Azure Function Setup

The Azure Function is the engine that processes your PDFs. It runs entirely in your own Azure subscription.

### Step 1 — Create a Function App

1. Go to [portal.azure.com](https://portal.azure.com)
2. Click **Create a resource** → search for **Function App** → click **Create**
3. Fill in the form:

| Field | Value |
|---|---|
| Subscription | Your subscription |
| Resource Group | Create new or use existing |
| Function App name | Choose a unique name e.g. `mycompany-pdf` |
| Runtime stack | Python |
| Version | 3.13 |
| Region | Choose closest to your location |
| Hosting plan | **Flex Consumption** |
| Instance size | 512 MB |
| Zone redundancy | Disabled |

4. Click **Review + create** → **Create**
5. Wait for deployment to complete → click **Go to resource**

> **Note on cost:** Flex Consumption includes 250,000 free executions and 100,000 GB-seconds per month.

### Step 2 — Deploy the Function Code

1. In the Azure portal, click the **Cloud Shell** icon (`>_`) in the top toolbar
2. Select **Bash** when prompted
3. Run the following commands one by one — copy and paste each block exactly as written:

**Create the folder structure:**
```bash
mkdir -p ~/BenedictPDF/pdf_processor
```

**Create host.json:**
```bash
cat > ~/BenedictPDF/host.json << 'EOF'
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true
      }
    }
  }
}
EOF
```

**Create requirements.txt:**
```bash
cat > ~/BenedictPDF/requirements.txt << 'EOF'
azure-functions
PyMuPDF
EOF
```

**Create function.json:**
```bash
cat > ~/BenedictPDF/pdf_processor/function.json << 'EOF'
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
EOF
```

**Create the main function:**
```bash
cat > ~/BenedictPDF/pdf_processor/__init__.py << 'EOF'
import azure.functions as func
import fitz
import base64
import json
import logging

def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info("PDF processor triggered.")
    doc = None

    try:
        body = req.get_json()

        pdf_base64 = body.get("pdf")
        if not pdf_base64:
            return func.HttpResponse(
                json.dumps({"error": "Missing pdf field"}),
                mimetype="application/json",
                status_code=400
            )

        if "," in pdf_base64:
            pdf_base64 = pdf_base64.split(",")[1]

        action = body.get("action", "process")
        if action not in ["process", "remove_password"]:
            return func.HttpResponse(
                json.dumps({"error": "Invalid action. Use 'process' or 'remove_password'."}),
                mimetype="application/json",
                status_code=400
            )

        watermark_text = body.get("watermark", "")
        password = body.get("password", "")
        current_password = body.get("current_password", "")
        fontsize = int(body.get("fontsize", 7))
        color_hex = body.get("color", "4C5270").lstrip("#")
        position = body.get("position", "top-right")
        first_page_only = body.get("first_page_only", False)
        padding_top = int(body.get("padding_top", 5))
        padding_right = int(body.get("padding_right", 0))

        pdf_bytes = base64.b64decode(pdf_base64)

        doc = fitz.open(stream=pdf_bytes, filetype="pdf")

        if doc.is_encrypted:
            if not current_password:
                return func.HttpResponse(
                    json.dumps({"error": "PDF is encrypted. Provide current_password to unlock."}),
                    mimetype="application/json",
                    status_code=400
                )

            if not doc.authenticate(current_password):
                return func.HttpResponse(
                    json.dumps({"error": "Incorrect current_password"}),
                    mimetype="application/json",
                    status_code=401
                )

        if action == "remove_password":
            output = doc.tobytes(
                encryption=fitz.PDF_ENCRYPT_NONE
            )

        else:
            if watermark_text:
                if len(color_hex) != 6:
                    return func.HttpResponse(
                        json.dumps({"error": "Invalid color format. Use HEX format like '4C5270'."}),
                        mimetype="application/json",
                        status_code=400
                    )

                r = int(color_hex[0:2], 16) / 255
                g = int(color_hex[2:4], 16) / 255
                b = int(color_hex[4:6], 16) / 255

                pages = [doc[0]] if first_page_only else doc

                for page in pages:
                    rect = page.rect
                    text_width = fitz.get_text_length(watermark_text, fontsize=fontsize)

                    if position == "top-left":
                        x, y = padding_right, padding_top + fontsize
                    elif position == "top-right":
                        x, y = rect.width - text_width - padding_right, padding_top + fontsize
                    elif position == "bottom-left":
                        x, y = padding_right, rect.height - padding_top
                    elif position == "bottom-right":
                        x, y = rect.width - text_width - padding_right, rect.height - padding_top
                    else:
                        x, y = rect.width / 4, rect.height / 2

                    page.insert_text(
                        (x, y),
                        watermark_text,
                        fontsize=fontsize,
                        color=(r, g, b)
                    )

            output = doc.tobytes(
                encryption=fitz.PDF_ENCRYPT_AES_256 if password else fitz.PDF_ENCRYPT_NONE,
                user_pw=password if password else None,
                owner_pw=password if password else None
            )

        return func.HttpResponse(
            json.dumps({
                "fileContent": base64.b64encode(output).decode("utf-8")
            }),
            mimetype="application/json",
            status_code=200
        )

    except Exception as e:
        logging.error(f"PDF processing error: {str(e)}")
        return func.HttpResponse(
            json.dumps({"error": "PDF processing failed"}),
            mimetype="application/json",
            status_code=500
        )

    finally:
        if doc:
            doc.close()
EOF
```

**Deploy:**
```bash
cd ~/BenedictPDF && zip -r ../BenedictPDF.zip . && az functionapp deployment source config-zip --resource-group <your-resource-group> --name <your-function-app-name> --src ../BenedictPDF.zip
```

Replace `<your-resource-group>` and `<your-function-app-name>` with your actual values. Wait for "Deployment was successful." before proceeding.

### Step 3 — Get Your Function Key

```bash
az functionapp function keys list --resource-group <your-resource-group> --name <your-function-app-name> --function-name pdf_processor
```

Save the key value — you will need it when configuring the Power Automate flows.

Your full function URL will be:
```
https://<your-function-app-name>.azurewebsites.net/api/pdf_processor?code=YOUR_KEY
```

> **Security note:** Treat your function key like a password/API Key/most valuable secret. Do not share it publicly or commit it to source control. Rotate it periodically in the Azure portal under **Function App → App keys**. And make sure to update it in Environment Variable in the Benedict PDF Solution in Power Apps.

---

## Part 2 — SharePoint Setup (DocuSign Version Only)

The DocuSign version uses SharePoint to track signature requests and store signed documents. Skip this section if you are using the standard version.

### Create a SharePoint Site

Create a dedicated SharePoint site for document processing, or use an existing one.

### Create a List — "Signature Requests"

Create a SharePoint list named exactly **Signature Requests** with the following columns:

| Column Name | Type | Notes |
|---|---|---|
| Title | Single line of text | Built-in — used for document name |
| DocusignEnvelopeID | Single line of text | Stores the DocuSign envelope ID |
| RequestorName | Single line of text | Name of the person requesting signature |
| RequestorEmail | Single line of text | Email of the requestor |
| DocumentPassword | Single line of text | Password applied to the final document |
| Status | Choice | Options: **Sent**, **Signed** |
| SignedDocumentID | Lookup | Lookup to the Signed Documents library |

### Create a Document Library — "Signed Documents"

Create a SharePoint Document Library named **Signed Documents** with the following column:

| Column Name | Type | Notes |
|---|---|---|
| RequestID | Lookup | Lookup to the **Signature Requests** list |

This library stores the final signed and password-protected PDFs, linked back to their signature request record.

---

## Part 3 — Power Platform Setup

### Step 1 — Import the Solution

1. Go to [make.powerautomate.com](https://make.powerautomate.com)
2. Click **Solutions** in the left menu
3. Click **Import solution**
4. Upload the solution zip file from this repository
5. Follow the import wizard — you will be prompted to set up connections
5.1. One of the Environment Variables you will see called "URL Azure Function". To get it - Go to your Azure Function App Overview and Click on "pdf_processor" -> search for "Get function URL" on top of the code -> Copy "default (Function key)" -> paste into the Environment Variable value.

### Step 2 — Configure the HTTP Action

The flows use HTTP actions to call your Azure Function. After importing the solution, open each flow and update the HTTP action:

- **URI:** Replace with your full function URL including the key
- **Method:** POST
- **Headers:** `Content-Type: application/json`

### Step 3 — Configure Connections

After importing, open each flow and confirm all connections are active. If any connection shows as invalid, click on it and sign in with your Microsoft 365 account.

Required connections:
- SharePoint
- OneDrive for Business
- Office 365 Outlook (for password email notification)
- DocuSign (DocuSign version only)

---

## Part 4 — Custom Connector Setup (Optional)

> This section is for advanced users who want a cleaner experience in Power Automate and Power Apps. Custom connectors also work in **Azure Logic Apps**. Requires Premium license.

### Create the Connector

1. Go to **Power Automate → Data → Custom connectors → + New custom connector → Create from blank**
2. Name it `BenedictPDF`
3. **General tab:**
   - Scheme: HTTPS
   - Host: `<your-function-app-name>.azurewebsites.net` (no https://, no trailing slash. To get the host go to your Azure Function App and copy Default domain)
4. **Security tab:**
   - Authentication type: API Key
   - Parameter label: API Key
   - Parameter name: `code`
   - Parameter location: Query
5. **Definition tab:** Click **+ New action** → fill in Summary, Description, Operation ID → click **+ Add request** and fill in:
   - Verb: POST
   - URL: `/api/pdf_processor`
   - Headers: `Content-Type: application/json`
   - Body: paste the example JSON from the API Reference section
   - Click **Import**
6. **Response:** Add a default response with body `{"fileContent": "sample"}` and set the `fileContent` property:
   - Type: `string`
   - Format: leave **blank** — do **not** set to `byte`

   > Setting format to `byte` causes Power Automate to automatically convert the base64 string to binary, which breaks the output in most scenarios.

7. Click **Update connector** → create a connection using your function key

### To upload document to Custom Connector

1. If it is a document from Power Apps, use `string(triggerBody()?['file']?['contentBytes'])`

2. If it is a document from SharePoint/DocuSign, use `body('<CONNECTOR_NAME>')?['$content']`

### Known Issue — fileContent Output

The connector returns `fileContent` as a base64 string. Depending on how Power Automate handles the response, you may encounter issues passing this value to a **Create file** action. In most cases `outputs('Process_PDF')?['body/fileContent']` should work fine, but there may be issues.

**If `fileContent` passes through correctly** — use `outputs('Process_PDF')?['body/fileContent']` in the File Content field.

**If the output appears as raw binary** (you see `%PDF-1.x` in the value instead of base64) — Power Automate is auto-converting the base64 somewhere in the chain. Fix this by:

1. Adding a **Compose** action after the BenedictPDF action with input:
   ```
   body('Process_PDF')?['fileContent']
   ```
2. Using the Compose output in the File Content field with expression:
   ```
   base64ToBinary(outputs('Compose'))
   ```

**If using the HTTP action** — use this expression for File Content:
```
base64ToBinary(body('HTTP')?['fileContent'])
```

Or if the HTTP action returns JSON directly:
```
base64ToBinary(body('HTTP')?['fileContent'])
```

> These nuances are caused by how Power Automate internally handles binary content and response types. The behaviour can vary depending on your environment, license, and connector configuration. If you encounter an issue not covered here, please [open an issue](https://github.com/Benedict-Corp/benedict-pdf/issues) and we will help.

---

## Usage

### Power Apps

#### Secure Documents

Open the **BenedictPDF** app from your Power Apps environment:

1. Upload a PDF document (PDF files only — other formats are not supported)
2. Enter watermark text (optional) or click **Generate** for an auto-generated unique identifier
3. Enter a password (optional) or click **Generate** for an auto-generated unique password
5. Click **Create**
6. Once processed, file will open automatically (make sure to enable pop-up windows in the browser). If, for some reason, it did not open, you will receive a file to your email.
7. The password is sent to your mailbox automatically as a separate email. Please note that, as part of security, password email does not contain any identification of the document itself; therefore, if you would secure several documents at once, you may receive different passwords.

#### Remove Password

Open the **BenedictPDF** app from your Power Apps environment and choose "Remove password from PDF":

1. Upload a PDF document secured with password
2. Enter password and click "Remove Password"
3. Document will open automatically and will be sent to you via email.

> You will recevie an error if password is incorrect.

#### Send Document for Signing in DocuSign

Open the **BenedictPDF** app from your Power Apps environment and follow steps from **Secure Documents** step above:

1. Click on toggle "Send for signing"
2. Fill out signer(s) details
3. Click "Send"
4. Once document will be signed, you will receive an email with the document and password as separate email.


### Power Automate

Use the **HTTP action** (or custom connector action) in any flow with the following parameters:

| Parameter | Description | Default |
|---|---|---|
| `pdf` | File content in base64 | Required |
| `action` | `process` or `remove_password` | `process` |
| `watermark` | Watermark text | None |
| `password` | Password to apply | None |
| `current_password` | Current password (required when removing) | None |
| `fontsize` | Watermark font size | `7` |
| `color` | Watermark color in HEX without `#` | `4C5270` |
| `position` | `top-right`, `top-left`, `bottom-right`, `bottom-left`, `center` | `top-right` |
| `first_page_only` | Apply watermark to first page only | `false` |
| `padding_top` | Top padding in points | `5` |
| `padding_right` | Right padding in points | `5` |

**Getting file content from SharePoint:**
```
body('Get_file_content')?['$content']
```

**Saving the processed file to SharePoint or OneDrive:**
```
base64ToBinary(body('HTTP')?['fileContent'])
```

> **Important:** Passwords use AES-256 encryption. There is no password recovery. A forgotten password means the document is permanently inaccessible. Always store passwords securely and send them to the recipient immediately after processing.

---

## API Reference

For users who want to call the function directly from their own applications or flows.

**Endpoint:**
```
POST https://<your-function-app>.azurewebsites.net/api/pdf_processor?code=YOUR_KEY (Go to your Azure Function App Overview and Click on "pdf_processor" -> search for "Get function URL" on top of the code -> Copy "default (Function key)" -> paste to the URI)
```

**Headers:**
```
Content-Type: application/json
```

**Example — watermark and password:**
```json
{
  "pdf": "base64encodedstring",
  "action": "process",
  "watermark": "CONFIDENTIAL",
  "password": "securepassword",
  "fontsize": 7,
  "color": "4C5270",
  "position": "top-right",
  "first_page_only": true,
  "padding_top": 5,
  "padding_right": 0
}
```

**Example — remove password:**
```json
{
  "pdf": "base64encodedstring",
  "action": "remove_password",
  "current_password": "existingpassword"
}
```

**Response:**
```json
{
  "fileContent": "base64encodedstring"
}
```

---

## Known Issues & Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `PDF processing failed` | Invalid base64 input | Ensure the pdf field contains clean base64. From SharePoint use `body('Get_file_content')?['$content']` |
| `Incorrect padding` | Data URI prefix included | The function strips this automatically. If persisting, check how file content is being passed |
| `File Content shows raw binary` | Power Automate auto-converting base64 | See fileContent troubleshooting in Custom Connector Setup |
| `HTTP 401` | Invalid or missing function key | Check your function URL includes `?code=YOUR_KEY` |
| `HTTP 403` | Network or VNet restriction | Your Azure environment may restrict external HTTP calls — contact your IT administrator |
| `HTTP 404` | Wrong function URL | Verify the function app name and region in the URL |
| `bad rotate value` | Older version of the code | Redeploy using the latest code from this repository |
| `string argument should contain only ASCII characters` | Non-base64 content passed to function | Wrap the file content in a Compose action using `string(triggerBody()?['file']?['contentBytes'])` before passing to the HTTP action |

If you encounter an issue not listed here, please [open an issue on GitHub](https://github.com/Benedict-Corp/benedict-pdf/issues) with details of your environment and the error message. We will do our best to help.

---

## License

BenedictPDF is released under the MIT License with Commons Clause.

You are free to use, modify, and integrate BenedictPDF into your own products and workflows. You may not resell BenedictPDF or a minimally modified version of it as a standalone product or service.

See [LICENSE](https://github.com/Benedict-Corp/benedict-pdf/blob/main/LICENSE) for full details.

---

## Support

BenedictPDF is free and will stay free.

If it saves you time or money, consider supporting Benedict Corp so we can keep building tools like this.

[**Support via Stripe →**](https://pay.benedictcorp.store/b/00w3cx79a1N7cT7c1qcAo07)

For questions or issues: [github.com/Benedict-Corp/benedict-pdf/issues](https://github.com/Benedict-Corp/benedict-pdf/issues)

For custom Power Platform solutions built for your organization: [benedictcorp.com](https://www.benedictcorp.com/) or email us at info@benedictcorp.com

---

*Benedict Corp. is not affiliated with or endorsed by Microsoft Corporation. Azure, Power Automate, Power Apps, and Logic Apps are trademarks of Microsoft Corporation. DocuSign is a trademark of DocuSign Inc. Stripe is a trademark of Stripe, Inc.*
