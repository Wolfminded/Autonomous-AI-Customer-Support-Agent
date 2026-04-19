# Setup Instructions

## 1. Export Your Workflow

In n8n, click **Download** → save as `n8n-workflow.json`

## 2. Set Up n8n

```bash
# If self-hosting
docker run -it --rm \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

## 3. Configure Credentials

In n8n UI, go to **Credentials**:

- **Telegram Bot Token:** Get from BotFather on Telegram
- **OpenAI API Key:** Generate at platform.openai.com
- **Order API:** Your backend endpoint (with auth headers)
- **Memory Store:** Redis connection string OR HTTP endpoint

## 4. Import the Workflow

- Click **Workflows** → **Import from file**
- Select `n8n-workflow.json`
- Click each red node to fill in credentials

## 5. Test

Send a message to your Telegram bot:

What's the status of order #12345?

Should get back: order status + tracking info (if in system)

## 6. Deploy to Production

- **Option A:** Run n8n in cloud (n8n.cloud)
- **Option B:** Self-host on your server (AWS, DigitalOcean, etc.)
- **Option C:** Use n8n's built-in webhook → keep workflow active

---

See ARCHITECTURE.md for customization & scaling.


## Usage Example

### Customer asks for order status
User: "Hey, where's my order?"
Agent: [Detects intent = order_status]
Agent: [Calls API with customer ID]
API: {status: "in_transit", tracking: "..."}
Agent: "Your order is in transit! Track it here: [link]"

### Customer escalation
User: "I'm furious, this product broke on arrival"
Agent: [Detects sentiment = upset, intent = complaint]
Agent: [Severity check fails]
Agent: "I'm sorry to hear that. Let me connect you with our support team."
[Routes to human + creates ticket with full context]
