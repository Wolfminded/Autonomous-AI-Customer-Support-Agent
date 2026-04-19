{
  "name": "AI Customer Support Agent (Telegram · Production)",
  "nodes": [
    {
      "parameters": {
        "updates": [
          "message"
        ],
        "additionalFields": {}
      },
      "id": "fbd42bac-fd37-41e0-8754-787782e79eec",
      "name": "① User Message — Telegram Trigger",
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1.1,
      "position": [
        -1792,
        176
      ],
      "webhookId": "YOUR_WEBHOOK_ID",
      "credentials": {
        "telegramApi": {
          "id": "YOUR_TELEGRAM_CREDENTIAL_ID",
          "name": "Your Telegram Bot"
        }
      },
      "notes": "DOC STEP 1 — User Message. Entry point. Every incoming Telegram message starts here."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 1 — Normalize User Message\n// Extracts all required fields from Telegram payload\n// ══════════════════════════════════════════════\nconst msg = $input.item.json;\n\nconst chatId    = msg?.message?.chat?.id ?? msg?.callback_query?.message?.chat?.id;\nconst userId    = msg?.message?.from?.id ?? msg?.callback_query?.from?.id;\nconst firstName = msg?.message?.from?.first_name ?? 'Customer';\nconst username  = msg?.message?.from?.username ?? 'unknown';\nconst text      = (msg?.message?.text ?? msg?.callback_query?.data ?? '').trim();\nconst messageId = msg?.message?.message_id ?? null;\nconst timestamp = new Date().toISOString();\n\nif (!chatId || !text) throw new Error('Invalid Telegram payload');\n\n// Extract order ID from message (e.g. 'My order 8472 has not arrived')\nconst orderIdMatch = text.match(/\\border\\s*[#:]?\\s*(\\d{3,10})/i);\nconst extractedOrderId = orderIdMatch ? orderIdMatch[1] : null;\n\nreturn [{\n  json: {\n    chatId, userId, firstName, username,\n    text, messageId, timestamp,\n    extractedOrderId,\n    sessionKey: `session_${userId}`,\n    raw: msg\n  }\n}];"
      },
      "id": "c92a0e0e-f8c1-4133-8d0a-bd729b01c488",
      "name": "Normalize User Message",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1568,
        176
      ],
      "notes": "DOC STEP 1 — Parses chatId, userId, text, and pre-extracts order ID from message."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 2 — Intent Detection\n// Classifies into: order_status | refund_request |\n// delivery_issue | general_question\n// Outputs structured data WITH confidence score\n// ══════════════════════════════════════════════\nconst text = $input.item.json.text.toLowerCase();\n\nconst intentRules = [\n  {\n    intent: 'order_status',\n    patterns: [/order.*(status|track|where|update|check)/i, /track.*order/i, /where.*my.*order/i, /order\\s*#?\\d+/i],\n    weight: 0\n  },\n  {\n    intent: 'refund_request',\n    patterns: [/refund/i, /money back/i, /charge.*wrong/i, /cancel.*order/i, /return.*item/i, /dispute/i],\n    weight: 0\n  },\n  {\n    intent: 'delivery_issue',\n    patterns: [/not.*arriv/i, /hasn.t.*arriv/i, /late/i, /delay/i, /missing.*package/i, /lost.*package/i, /wrong.*item/i, /damaged/i, /never.*receiv/i],\n    weight: 0\n  },\n  {\n    intent: 'general_question',\n    patterns: [/how.*do/i, /what.*is/i, /help/i, /support/i, /contact/i, /question/i, /info/i],\n    weight: 0\n  }\n];\n\n// Score each intent\nfor (const rule of intentRules) {\n  for (const pattern of rule.patterns) {\n    if (pattern.test(text)) rule.weight++;\n  }\n}\n\n// Sort by score descending\nintentRules.sort((a, b) => b.weight - a.weight);\n\nconst topIntent  = intentRules[0];\nconst totalHits  = intentRules.reduce((s, r) => s + r.weight, 0);\nconst confidence = totalHits > 0 ? parseFloat((topIntent.weight / totalHits).toFixed(2)) : 0.40;\nconst intent     = topIntent.weight > 0 ? topIntent.intent : 'general_question';\n\n// Confidence < 0.50 = uncertain → flag for verification layer\nconst isUncertain = confidence < 0.50;\n\nreturn [{\n  json: {\n    ...$input.item.json,\n    intent,\n    confidence,\n    isUncertain,\n    intentScores: Object.fromEntries(intentRules.map(r => [r.intent, r.weight]))\n  }\n}];"
      },
      "id": "2256b063-c313-4098-bd8f-ace53c9777ff",
      "name": "② Intent Detection",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1344,
        176
      ],
      "notes": "DOC STEP 2 — Classifies intent into 4 types with confidence score. Low confidence flagged for verification."
    },
    {
      "parameters": {
        "url": "=https://YOUR_MEMORY_SERVICE/history/{{$json.userId}}",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "limit",
              "value": "5"
            },
            {
              "name": "summarize",
              "value": "true"
            }
          ]
        },
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "X-API-Key",
              "value": "={{$env.MEMORY_API_KEY}}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "options": {
          "timeout": 3000
        }
      },
      "id": "747cca23-c7ae-476c-bb53-c5bda4d47e68",
      "name": "③ Retrieve Memory",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        -1120,
        176
      ],
      "continueOnFail": true,
      "notes": "DOC STEP 3 — Retrieve Memory. Fetches last 5 interactions (summarized) for context. continueOnFail=true so flow never breaks if memory is unavailable."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 3 — Build memory context\n// Summarizes history to minimize token usage\n// (Cost Optimization: send minimal context)\n// ══════════════════════════════════════════════\nconst input = $input.item.json;\n\n// Memory service response (may be null if service down)\nlet memoryHistory = [];\ntry {\n  const raw = input?.history ?? input?.data ?? [];\n  // Take last 5 turns max, summarize to key facts only (cost optimization)\n  memoryHistory = Array.isArray(raw) ? raw.slice(-5).map(h => ({\n    role: h.role ?? 'user',\n    summary: (h.content ?? h.text ?? '').slice(0, 120) // cap at 120 chars per turn\n  })) : [];\n} catch(e) {\n  memoryHistory = [];\n}\n\nconst memorySummary = memoryHistory.length > 0\n  ? memoryHistory.map(h => `${h.role}: ${h.summary}`).join(' | ')\n  : 'No prior history.';\n\nreturn [{\n  json: {\n    ...$input.item.json,\n    memoryHistory,\n    memorySummary,\n    hasMemory: memoryHistory.length > 0\n  }\n}];"
      },
      "id": "d08300ae-adeb-4d62-a812-41f81df1ca56",
      "name": "Build Memory Context",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -896,
        176
      ],
      "notes": "DOC STEP 3 + Cost Optimization — Summarizes history to 120 chars per turn, max 5 turns."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false
          },
          "conditions": [
            {
              "id": "needs-tool",
              "leftValue": "={{$json.intent}}",
              "rightValue": "general_question",
              "operator": {
                "type": "string",
                "operation": "notEquals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "650c52b8-3dee-477a-81d1-6071db7f2b5b",
      "name": "④ Tool Decision",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        -672,
        176
      ],
      "notes": "DOC STEP 4 — Tool Decision. order_status / refund_request / delivery_issue → call Order API. general_question → skip to AI generation directly."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 4 — Validate before Order API call\n// Test scenarios per document:\n// - valid order ID → call API\n// - missing order ID → ask user\n// - invalid format → reject gracefully\n// ══════════════════════════════════════════════\nconst input = $input.item.json;\nconst orderId = input.extractedOrderId;\n\nif (!orderId) {\n  return [{\n    json: {\n      ...input,\n      apiCallNeeded: false,\n      apiSkipReason: 'missing_order_id',\n      apiResult: null,\n      telegramFallback: `Hi ${input.firstName}! I need your order number to look that up. Could you share it? (Example: \"My order is 8472\")`\n    }\n  }];\n}\n\n// Validate order ID format: 3-10 digits only\nif (!/^\\d{3,10}$/.test(orderId)) {\n  return [{\n    json: {\n      ...input,\n      apiCallNeeded: false,\n      apiSkipReason: 'invalid_order_id',\n      apiResult: null,\n      telegramFallback: `That order number (${orderId}) doesn't look right. Order IDs are usually 4–7 digits. Could you double-check?`\n    }\n  }];\n}\n\nreturn [{\n  json: { ...input, apiCallNeeded: true, apiSkipReason: null, apiResult: null, telegramFallback: null }\n}];"
      },
      "id": "1d6c7311-8fff-43e0-a59f-77b2f2c2c2dd",
      "name": "Validate Order ID",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -448,
        96
      ],
      "notes": "DOC Testing Scenarios — Handles: valid ID, missing ID, invalid ID before API call."
    },
    {
      "parameters": {
        "conditions": {
          "conditions": [
            {
              "leftValue": "={{$json.apiCallNeeded}}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ]
        },
        "options": {}
      },
      "id": "3c15f5d8-b2ce-4ab4-8310-f2349f249b32",
      "name": "Has Valid Order ID?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        -224,
        96
      ]
    },
    {
      "parameters": {
        "url": "=https://YOUR_ORDER_API/orders/{{$json.extractedOrderId}}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$env.ORDER_API_KEY}}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "options": {
          "timeout": 5000
        }
      },
      "id": "8226ab43-edde-4613-91e2-4745dbac4a49",
      "name": "⑤ Call Order API",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        0,
        0
      ],
      "continueOnFail": true,
      "notes": "DOC STEP 5 — Call Order API. Retrieves real order data instead of guessing (as per document example: order 8472)."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 5 — Parse Order API Response\n// Merges real API data into the workflow context\n// ══════════════════════════════════════════════\nconst input = $input.item.json;\n\nlet apiResult = null;\nlet apiError  = null;\n\ntry {\n  if (input?.orderId || input?.order_id || input?.id) {\n    apiResult = {\n      orderId:       input.orderId ?? input.order_id ?? input.id,\n      status:        input.status ?? 'unknown',\n      estimatedDate: input.estimated_delivery ?? input.estimatedDate ?? 'N/A',\n      carrier:       input.carrier ?? 'N/A',\n      trackingUrl:   input.tracking_url ?? null,\n      lastLocation:  input.last_location ?? 'N/A',\n      items:         input.items ?? [],\n      totalAmount:   input.total_amount ?? input.amount ?? null\n    };\n  } else {\n    apiError = 'Order not found in system';\n  }\n} catch(e) {\n  apiError = e.message;\n}\n\n// Pass through all prior context\nconst prior = $('Validate Order ID').item.json;\n\nreturn [{\n  json: {\n    ...prior,\n    apiResult,\n    apiError,\n    toolUsed: true,\n    toolName: 'order_api'\n  }\n}];"
      },
      "id": "29a22c5d-fae9-4381-8de9-d33f55280dc9",
      "name": "Parse API Response",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        224,
        0
      ],
      "notes": "DOC STEP 5 — Parses order API response into a structured apiResult object."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 6 — Verification Layer\n// Validates LLM inputs & outputs:\n// 1. Checks confidence threshold (< 0.5 = escalate)\n// 2. Validates API data is present when needed\n// 3. Flags for human escalation if required\n// ══════════════════════════════════════════════\n\n// Merge from all paths\nconst fromApiPath   = $('Parse API Response').first()?.json ?? null;\nconst fromSkipPath  = $('Validate Order ID').first()?.json ?? null;\nconst fromNoTool    = $('Build Memory Context').first()?.json ?? null;\n\n// Determine which path arrived\nconst input = fromApiPath ?? fromSkipPath ?? fromNoTool ?? $input.item.json;\n\nconst confidence = input.confidence ?? 0.5;\nconst intent     = input.intent ?? 'general_question';\nconst apiResult  = input.apiResult ?? null;\nconst apiError   = input.apiError  ?? null;\n\n// Verification checks\nconst checks = {\n  confidenceTooLow:    confidence < 0.50,\n  apiFailedWhenNeeded: input.apiCallNeeded === true && !apiResult && !!apiError,\n  missingOrderId:      input.apiSkipReason === 'missing_order_id',\n  invalidOrderId:      input.apiSkipReason === 'invalid_order_id'\n};\n\nconst shouldEscalate = checks.confidenceTooLow || checks.apiFailedWhenNeeded;\n\nreturn [{\n  json: {\n    ...input,\n    verificationChecks: checks,\n    shouldEscalate,\n    escalationReason: shouldEscalate\n      ? (checks.confidenceTooLow ? 'low_confidence' : 'api_failure')\n      : null\n  }\n}];"
      },
      "id": "2e58900a-c491-4975-9e68-840d28dcc432",
      "name": "⑥ Verification Layer",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        448,
        176
      ],
      "notes": "DOC STEP 6 — Verification Layer. Validates confidence threshold and API data. Escalates to human if confidence < 0.50 or API failed."
    },
    {
      "parameters": {
        "conditions": {
          "conditions": [
            {
              "leftValue": "={{$json.shouldEscalate}}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ]
        },
        "options": {}
      },
      "id": "d6de6e8f-b73d-446c-81f4-feb62926fc04",
      "name": "Escalate to Human?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        672,
        176
      ],
      "notes": "DOC STEP 6 — Routes to human escalation path or AI generation path."
    },
    {
      "parameters": {
        "chatId": "={{ $('① User Message — Telegram Trigger').item.json.message.chat.id }}",
        "text": "=🔴 *Connecting you to a human agent*\n\nOur AI wasn't fully confident about your request and has flagged it for a human specialist.\n\n🎫 Ticket: `TKT-{{ $('① User Message — Telegram Trigger').item.json.message.from.id }}-{{ $('Normalize User Message').item.json.timestamp }}`\n📋 Your request: _{{ $('① User Message — Telegram Trigger').item.json.message.text }}_\n\nA team member will respond within a few minutes. Thank you for your patience!",
        "additionalFields": {
          "parse_mode": "Markdown"
        }
      },
      "id": "8ccd1584-c6a7-427c-8eac-cd0397ec3223",
      "name": "Send Escalation Message",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        896,
        80
      ],
      "operation": "sendMessage",
      "webhookId": "YOUR_WEBHOOK_ID",
      "credentials": {
        "telegramApi": {
          "id": "YOUR_TELEGRAM_CREDENTIAL_ID",
          "name": "Your Telegram Bot"
        }
      },
      "notes": "DOC STEP 6 — Human escalation message sent when confidence < 0.50 or API fails."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://YOUR_CRM_WEBHOOK/escalate",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Content-Type",
              "value": "application/json"
            },
            {
              "name": "X-API-Key",
              "value": "={{$env.CRM_API_KEY}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\"userId\":\"{{$json.userId}}\",\"chatId\":\"{{$json.chatId}}\",\"username\":\"{{$json.username}}\",\"message\":\"{{$json.text}}\",\"intent\":\"{{$json.intent}}\",\"confidence\":{{$json.confidence}},\"escalationReason\":\"{{$json.escalationReason}}\",\"timestamp\":\"{{$json.timestamp}}\"}",
        "options": {
          "timeout": 5000
        }
      },
      "id": "3da48950-a87a-4ba6-bfe7-27e7381619b9",
      "name": "Create Escalation Ticket",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1184,
        80
      ],
      "continueOnFail": true,
      "notes": "DOC STEP 6 — Creates ticket in CRM/helpdesk with full context for human agent."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 7 — Generate Response (Cost Optimized)\n// Builds the minimal-context prompt:\n// - Summarized memory (not full history)\n// - Only relevant API data fields\n// - Intent-specific system instructions\n// - Use tools data instead of LLM reasoning\n// ══════════════════════════════════════════════\nconst input = $input.item.json;\n\nconst intentInstructions = {\n  order_status:    'The user wants to know their order status. Use the API data provided. Be specific about dates, carrier, and tracking.',\n  refund_request:  'The user wants a refund or has a billing issue. Be empathetic, acknowledge the issue, explain the refund process (3-5 business days). If order data is available, reference it.',\n  delivery_issue:  'The user has a delivery problem — late, missing, or damaged. Express genuine empathy first. Provide tracking info if available. Offer next steps.',\n  general_question:'Answer the general question clearly and concisely. If you don\\'t know, say so and offer to connect with a human.'\n};\n\n// Cost optimization: only include relevant API fields, not the whole object\nlet apiContext = '';\nif (input.apiResult) {\n  const r = input.apiResult;\n  apiContext = `\\nORDER DATA (from real API — do NOT guess, use only this):\\nOrder ID: ${r.orderId} | Status: ${r.status} | Est. Delivery: ${r.estimatedDate} | Carrier: ${r.carrier} | Last Location: ${r.lastLocation}${r.trackingUrl ? ` | Tracking: ${r.trackingUrl}` : ''}`;\n} else if (input.apiError) {\n  apiContext = `\\nORDER LOOKUP: Failed (${input.apiError}). Do not guess order status.`;\n} else if (input.apiSkipReason === 'missing_order_id') {\n  apiContext = `\\nNOTE: No order ID found in message. Ask user to provide it.`;\n}\n\n// Cost optimization: summarized history (not raw messages)\nconst historyContext = input.memorySummary && input.memorySummary !== 'No prior history.'\n  ? `\\nPREVIOUS CONTEXT (summarized): ${input.memorySummary}`\n  : '';\n\nconst systemPrompt = `You are a professional AI customer support agent for [YOUR COMPANY].\nBe empathetic, concise, and solution-oriented. Address the user as ${input.firstName}.\nKeep responses under 180 words. Use Telegram Markdown formatting.\nNever invent order data — only use what is provided in ORDER DATA below.\n${intentInstructions[input.intent] ?? intentInstructions.general_question}\n${apiContext}\n${historyContext}`;\n\nconst userPrompt = `Customer message: \"${input.text}\"\nDetected intent: ${input.intent} (confidence: ${input.confidence})`;\n\nreturn [{ json: { ...input, systemPrompt, userPrompt } }];"
      },
      "id": "4c03d1b7-2a6f-4fa2-bd29-a6c7c6e5a9a4",
      "name": "Build Cost-Optimized Prompt",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        896,
        272
      ],
      "notes": "DOC STEP 7 + Cost Optimization — Builds minimal-token prompt: summarized history, only relevant API fields, intent-specific instructions."
    },
    {
      "parameters": {
        "modelId": {
          "__rl": true,
          "value": "gpt-3.5-turbo",
          "mode": "list",
          "cachedResultName": "GPT-3.5-TURBO"
        },
        "messages": {
          "values": [
            {
              "content": "={{$json.systemPrompt}}",
              "role": "system"
            },
            {
              "content": "={{$json.userPrompt}}"
            }
          ]
        },
        "options": {
          "maxTokens": 300,
          "temperature": 0.35
        }
      },
      "id": "47cefaa8-94e0-4d79-9df4-867b29928a90",
      "name": "⑦ Generate Response — GPT-4o",
      "type": "@n8n/n8n-nodes-langchain.openAi",
      "typeVersion": 1,
      "position": [
        1120,
        272
      ],
      "credentials": {
        "openAiApi": {
          "id": "YOUR_OPENAI_CREDENTIAL_ID",
          "name": "Your OpenAI Account"
        }
      },
      "notes": "DOC STEP 7 — Generate Response. maxTokens=300 for cost control. Uses real API data instead of reasoning wherever possible."
    },
    {
      "parameters": {
        "jsCode": "// ══════════════════════════════════════════════\n// DOC STEP 7 — Post-process AI response\n// Final safety check before sending\n// ══════════════════════════════════════════════\nconst input = $input.item.json;\nconst aiText = input?.message?.content ?? input?.choices?.[0]?.message?.content ?? '';\n\nif (!aiText.trim()) throw new Error('Empty AI response');\n\nconst MAX = 3800;\nconst finalText = aiText.length > MAX\n  ? aiText.slice(0, MAX) + '\\n\\n_[Reply to continue]_'\n  : aiText;\n\nreturn [{ json: { ...input, finalResponse: finalText } }];"
      },
      "id": "332ac888-8ed7-4d31-a0d9-ced0f39f65fc",
      "name": "Validate AI Response",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1472,
        272
      ]
    },
    {
      "parameters": {
        "chatId": "={{$json.chatId}}",
        "text": "={{$json.finalResponse}}",
        "additionalFields": {
          "parse_mode": "Markdown",
          "reply_to_message_id": "={{$json.messageId}}"
        }
      },
      "id": "93024186-7927-48c4-ad86-51d429bfb3a0",
      "name": "Send AI Reply — Telegram",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        1696,
        272
      ],
      "operation": "sendMessage",
      "webhookId": "YOUR_WEBHOOK_ID",
      "credentials": {
        "telegramApi": {
          "id": "YOUR_TELEGRAM_CREDENTIAL_ID",
          "name": "Your Telegram Bot"
        }
      },
      "notes": "DOC STEP 7 — Sends the AI-generated response back to the user on Telegram."
    },
    {
      "parameters": {
        "chatId": "={{$json.chatId}}",
        "text": "={{$json.telegramFallback}}",
        "additionalFields": {
          "parse_mode": "Markdown"
        }
      },
      "id": "268cdf2a-c6c4-4a79-a18f-57f93bd46c4c",
      "name": "Send Fallback (No/Invalid Order ID)",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        0,
        192
      ],
      "operation": "sendMessage",
      "webhookId": "YOUR_WEBHOOK_ID",
      "credentials": {
        "telegramApi": {
          "id": "YOUR_TELEGRAM_CREDENTIAL_ID",
          "name": "Your Telegram Bot"
        }
      },
      "notes": "DOC Testing Scenarios — Handles missing_order_id and invalid_order_id cases gracefully."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://YOUR_LOGGING_ENDPOINT/events",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Content-Type",
              "value": "application/json"
            },
            {
              "name": "X-API-Key",
              "value": "={{$env.LOGGING_API_KEY}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"event\": \"support_interaction\",\n  \"userId\": \"{{$json.userId}}\",\n  \"username\": \"{{$json.username}}\",\n  \"chatId\": \"{{$json.chatId}}\",\n  \"userInput\": \"{{$json.text}}\",\n  \"detectedIntent\": \"{{$json.intent}}\",\n  \"confidence\": {{$json.confidence}},\n  \"toolUsed\": {{$json.toolUsed ?? false}},\n  \"toolName\": \"{{$json.toolName ?? 'none'}}\",\n  \"orderId\": \"{{$json.extractedOrderId ?? 'none'}}\",\n  \"apiResult\": {{$json.apiResult ? 'true' : 'false'}},\n  \"escalated\": {{$json.shouldEscalate ?? false}},\n  \"escalationReason\": \"{{$json.escalationReason ?? 'none'}}\",\n  \"hasMemory\": {{$json.hasMemory ?? false}},\n  \"aiResponse\": \"{{($json.finalResponse ?? '').slice(0,200)}}\",\n  \"timestamp\": \"{{$json.timestamp}}\",\n  \"source\": \"telegram\"\n}",
        "options": {
          "timeout": 3000
        }
      },
      "id": "d534039a-7023-493c-b13f-ae60a47d877c",
      "name": "⑧ Log Event",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1920,
        272
      ],
      "continueOnFail": true,
      "notes": "DOC STEP 8 — Log Event. Logs: user input, detected intent, tool usage, confidence score, timestamp. All fields from the document."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://YOUR_MEMORY_SERVICE/history",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Content-Type",
              "value": "application/json"
            },
            {
              "name": "X-API-Key",
              "value": "={{$env.MEMORY_API_KEY}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\"userId\":\"{{$json.userId}}\",\"chatId\":\"{{$json.chatId}}\",\"userMessage\":\"{{$json.text}}\",\"agentResponse\":\"{{($json.finalResponse ?? '').slice(0,300)}}\",\"intent\":\"{{$json.intent}}\",\"timestamp\":\"{{$json.timestamp}}\"}",
        "options": {
          "timeout": 3000
        }
      },
      "id": "b961b2fa-aad4-43fa-ba22-5981c8a7ce1f",
      "name": "Save to Memory Store",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2144,
        272
      ],
      "continueOnFail": true,
      "notes": "DOC STEP 3 (write-back) — Saves this interaction to memory for future context retrieval."
    },
    {
      "parameters": {
        "jsCode": "const input = $input.item.json;\nconsole.error('[Day14 Agent Error]', JSON.stringify(input, null, 2));\nconst chatId = input?.chatId ?? null;\nif (!chatId) return [{ json: { handled: false } }];\nreturn [{ json: { chatId, timestamp: new Date().toISOString() } }];"
      },
      "id": "73c32143-9f8c-454e-922b-1f2537628803",
      "name": "Global Error Handler",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1792,
        496
      ],
      "notes": "Error workflow entrypoint — catches all unhandled failures."
    },
    {
      "parameters": {
        "chatId": "={{$json.chatId}}",
        "text": "⚠️ *Something went wrong*\n\nI ran into an unexpected issue. Please try again, \n\nSorry for the inconvenience!",
        "additionalFields": {
          "parse_mode": "Markdown"
        }
      },
      "id": "2e09b950-02ff-4583-8967-0b17de09d4de",
      "name": "Send Error Fallback",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        -1568,
        496
      ],
      "operation": "sendMessage",
      "webhookId": "YOUR_WEBHOOK_ID",
      "credentials": {
        "telegramApi": {
          "id": "YOUR_TELEGRAM_CREDENTIAL_ID",
          "name": "Your Telegram Bot"
        }
      }
    }
  ],
  "pinData": {},
  "connections": {
    "① User Message — Telegram Trigger": {
      "main": [
        [
          {
            "node": "Normalize User Message",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Normalize User Message": {
      "main": [
        [
          {
            "node": "② Intent Detection",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "② Intent Detection": {
      "main": [
        [
          {
            "node": "③ Retrieve Memory",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "③ Retrieve Memory": {
      "main": [
        [
          {
            "node": "Build Memory Context",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build Memory Context": {
      "main": [
        [
          {
            "node": "④ Tool Decision",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "④ Tool Decision": {
      "main": [
        [
          {
            "node": "Validate Order ID",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "⑥ Verification Layer",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Validate Order ID": {
      "main": [
        [
          {
            "node": "Has Valid Order ID?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Has Valid Order ID?": {
      "main": [
        [
          {
            "node": "⑤ Call Order API",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Send Fallback (No/Invalid Order ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "⑤ Call Order API": {
      "main": [
        [
          {
            "node": "Parse API Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse API Response": {
      "main": [
        [
          {
            "node": "⑥ Verification Layer",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "⑥ Verification Layer": {
      "main": [
        [
          {
            "node": "Escalate to Human?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Escalate to Human?": {
      "main": [
        [
          {
            "node": "Send Escalation Message",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Build Cost-Optimized Prompt",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Send Escalation Message": {
      "main": [
        [
          {
            "node": "Create Escalation Ticket",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build Cost-Optimized Prompt": {
      "main": [
        [
          {
            "node": "⑦ Generate Response — GPT-4o",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "⑦ Generate Response — GPT-4o": {
      "main": [
        [
          {
            "node": "Validate AI Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Validate AI Response": {
      "main": [
        [
          {
            "node": "Send AI Reply — Telegram",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Send AI Reply — Telegram": {
      "main": [
        [
          {
            "node": "⑧ Log Event",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "⑧ Log Event": {
      "main": [
        [
          {
            "node": "Save to Memory Store",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Global Error Handler": {
      "main": [
        [
          {
            "node": "Send Error Fallback",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "availableInMCP": false
  },
  "versionId": "YOUR_VERSION_ID",
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "YOUR_INSTANCE_ID"
  },
  "id": "YOUR_WORKFLOW_ID",
  "tags": [
    {
      "updatedAt": "2026-03-13T16:18:48.618Z",
      "createdAt": "2026-03-13T16:18:48.618Z",
      "id": "YOUR_TAG_ID",
      "name": "telegram"
    },
    {
      "updatedAt": "2026-03-13T16:18:48.411Z",
      "createdAt": "2026-03-13T16:18:48.411Z",
      "id": "YOUR_TAG_ID",
      "name": "production"
    },
    {
      "updatedAt": "2026-03-13T16:18:48.588Z",
      "createdAt": "2026-03-13T16:18:48.588Z",
      "id": "YOUR_TAG_ID",
      "name": "agentic-ai"
    },
    {
      "updatedAt": "2026-03-13T16:19:28.868Z",
      "createdAt": "2026-03-13T16:19:28.868Z",
      "id": "YOUR_TAG_ID",
      "name": "READY"
    }
  ]
}
