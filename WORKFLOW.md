# Workflow Breakdown

## Stage 1: Message Ingestion
- Telegram trigger receives raw message
- Normalize text (trim, lowercase where relevant)

## Stage 2: Intent Detection
- LLM classifies query: order_status, return, complaint, billing, etc.
- Confidence score included
- If <60% confidence, route directly to escalation

## Stage 3: Memory Context
- HTTP request to Redis / memory backend
- Load last 5-10 messages from this customer
- Include order history snippet if available

## Stage 4: Tool Decision
- Code node: should we call an API?
- If intent = "order_status" → yes
- If intent = "complaint" → check severity first
- If intent = "general question" → no API needed

## Stage 5: Order API Call
- Validate order ID from message
- Call your backend API
- Parse response (order status, tracking, etc.)

## Stage 6: Verification Layer
- Quality checks on retrieved data
- Does the response match the query intent?
- Is order data complete and valid?
- If validation fails → escalate

## Stage 7: Escalation Decision
- If query severity = high → escalate
- If API failed → escalate
- If confidence < 70% → escalate
- Otherwise → proceed to AI response generation

## Stage 8: Cost-Optimized Prompting
- Build a lean prompt including:
  - Customer name + history
  - Their order details
  - Specific intent
  - Context (previous messages)
- Minimize tokens sent to GPT-4o

## Stage 9: AI Response Generation
- Call GPT-4o with optimized prompt
- Use temperature 0.7 (balanced creativity + consistency)
- Include instruction: "Be concise, helpful, professional"

## Stage 10: Response Validation
- Check response length (not too short, not absurd)
- Check for safety violations
- Check grammar / coherence
- If invalid → fallback response

## Stage 11: Send Reply
- Format response for Telegram
- Send via Telegram API
- Log success/failure

## Stage 12: Persist Memory
- Save this conversation turn
- Update customer profile
- Log metrics for analytics

## Error Handling
- Global error handler catches any node failure
- Sends fallback message to customer
- Escalates internally for debugging
- Does NOT crash the workflow
