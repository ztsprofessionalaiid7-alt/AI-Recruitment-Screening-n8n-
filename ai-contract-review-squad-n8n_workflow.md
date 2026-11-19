# AI Contract Review Squad: 4 Agents (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `contract-review-squad-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI Contract Review Squad (4 agents, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Contract Review Squad (n8n Cloud) - 4 agents in sequence, 3 model providers\nClause Extractor: Groq llama-3.1-8b (fast extraction)\nRisk Analyst: Groq llama-3.3-70b (legal reasoning)\nCompliance Checker: OpenRouter free model (policy comparison, independent view)\nNegotiation Advisor: Gemini (clear redline wording)\nSetup: credentials Groq, OpenRouter, Gemini, Slack, Postgres (Supabase), SMTP. Tables contract_policies (seed rows) and contract_reviews. Slack channel legal-review.\nEach agent reads the previous agent's result. Rules in code stop a risky or non-compliant contract from being cleared. Demo contracts are invented.",
        "height": 340,
        "width": 640
      },
      "id": "e61b4d60-4dff-4858-8404-989772acb01b",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -300
      ]
    },
    {
      "parameters": {},
      "id": "bbca6e6c-6d38-426e-b405-0cbdb0a4f31f",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { contract_id: 'CT-01', name: 'CloudDesk SaaS Subscription', counterparty: 'CloudDesk Inc', text: 'Term: 12 months. This agreement renews automatically for successive 12-month terms unless cancelled in writing 90 days before the end of the term. Vendor liability is capped at the fees paid in the previous 3 months. Fees are payable within 30 days of invoice. Governing law: Delaware.' } },\n  { json: { contract_id: 'CT-02', name: 'Logistics Services Agreement', counterparty: 'FastFreight Ltd', text: 'Term: 5 years with FastFreight as exclusive supplier for the whole term. Customer shall indemnify the supplier against all claims. The supplier liability is unlimited. Payment terms: 120 days. Termination is only allowed for cause.' } },\n  { json: { contract_id: 'CT-03', name: 'Mutual NDA', counterparty: 'Orbit Labs', text: 'Mutual non-disclosure agreement. Confidential information must be protected for 2 years. Either party may terminate with 30 days notice. Governing law: England and Wales.' } }\n];"
      },
      "id": "8a3308f5-c3d3-4213-8e1c-020567cc89c3",
      "name": "Mock Contracts",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "operation": "select",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "contract_policies",
          "mode": "name"
        },
        "options": {},
        "returnAll": true
      },
      "id": "27053090-515a-467e-81c8-eca5d1202088",
      "name": "Get Policies",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        440,
        200
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      },
      "executeOnce": true
    },
    {
      "parameters": {
        "jsCode": "const policy = $input.all().map((p) => '- ' + p.json.topic + ': ' + p.json.rule).join('\\n') || 'No policies found.';\nreturn $('Mock Contracts').all().map((c) => ({ json: Object.assign({}, c.json, { policy_text: policy }) }));"
      },
      "id": "f31a66a8-6576-4cd6-96e7-42744cda57a2",
      "name": "Attach Policies",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        660,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Extract the key clauses.\nContract: {{ $json.name }}\n\n{{ $json.text }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a contract clause extraction agent. Copy the facts from the text in short phrases and write \"not stated\" for anything missing. Respond with ONLY a JSON object, no markdown, no extra text: {\"term\":\"\",\"auto_renewal\":\"\",\"liability\":\"\",\"payment_terms\":\"\",\"exclusivity\":\"\",\"termination\":\"\",\"indemnity\":\"\"}"
        }
      },
      "id": "4010096b-4ab4-41e3-b9ee-8355b51aae39",
      "name": "Clause Extractor Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        900,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "llama-3.1-8b-instant",
        "options": {}
      },
      "id": "29565a15-d101-4fa8-b40e-2bd796e2c5db",
      "name": "Clause Extractor Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGroq",
      "typeVersion": 1,
      "position": [
        880,
        440
      ],
      "credentials": {
        "groqApi": {
          "id": "REPLACE_GROQ",
          "name": "Groq account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Attach Policies').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst f = (k) => String(r[k] || 'not stated');\nconst clauses = 'Term: ' + f('term') + '\\nAuto-renewal: ' + f('auto_renewal') + '\\nLiability: ' + f('liability') + '\\nPayment terms: ' + f('payment_terms') + '\\nExclusivity: ' + f('exclusivity') + '\\nTermination: ' + f('termination') + '\\nIndemnity: ' + f('indemnity');\nreturn { json: Object.assign({}, ctx, { clauses }) };"
      },
      "id": "dd82dbe8-75ad-4336-b8a6-226eced4b168",
      "name": "Parse Clauses",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1150,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Assess the legal and commercial risk of these clauses for the customer.\nContract: {{ $json.name }}\n{{ $json.clauses }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a contract risk analyst working for the customer. Respond with ONLY a JSON object, no markdown, no extra text: {\"risk_level\":\"low|medium|high\",\"risks\":\"short list in one string\"}"
        }
      },
      "id": "bab7351c-919f-4b35-8b05-c90e2b1132ba",
      "name": "Risk Analyst Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1420,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {}
      },
      "id": "73e7876b-b056-4fc5-bde2-9996304825de",
      "name": "Risk Analyst Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGroq",
      "typeVersion": 1,
      "position": [
        1400,
        440
      ],
      "credentials": {
        "groqApi": {
          "id": "REPLACE_GROQ",
          "name": "Groq account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Clauses').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst lvl = (['low', 'medium', 'high'].includes(String(r.risk_level).toLowerCase()) ? String(r.risk_level).toLowerCase() : 'high');\nreturn { json: Object.assign({}, ctx, { risk_level: lvl, risks: String(r.risks || '').slice(0, 400) }) };"
      },
      "id": "f3bfebf9-193c-4250-9afd-d779d89454b9",
      "name": "Parse Risk",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1670,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Check the clauses against company policy.\nClauses:\n{{ $json.clauses }}\n\nCompany policy:\n{{ $json.policy_text }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a contract compliance checker. List only real violations of the policy. Respond with ONLY a JSON object, no markdown, no extra text: {\"compliant\":true,\"violations\":\"none or a short list in one string\"}"
        }
      },
      "id": "8729701e-a61c-49e5-8e9d-06922cdab264",
      "name": "Compliance Checker Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1940,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "meta-llama/llama-3.3-70b-instruct:free",
        "options": {}
      },
      "id": "0733aa3d-e690-4e9b-9907-b511823930a5",
      "name": "Compliance Checker Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
      "typeVersion": 1,
      "position": [
        1920,
        440
      ],
      "credentials": {
        "openRouterApi": {
          "id": "REPLACE_OPENROUTER",
          "name": "OpenRouter account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Risk').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst ok = r.compliant === true || String(r.compliant).toLowerCase() === 'true';\nreturn { json: Object.assign({}, ctx, { compliant: ok ? 'yes' : 'no', violations: String(r.violations || 'none').slice(0, 400) }) };"
      },
      "id": "2e507461-ab42-426c-80d6-13d38a5d7731",
      "name": "Parse Compliance",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2190,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Advise on this contract.\nContract: {{ $json.name }} with {{ $json.counterparty }}\nRisk level: {{ $json.risk_level }}\nRisks: {{ $json.risks }}\nPolicy compliant: {{ $json.compliant }}\nPolicy violations: {{ $json.violations }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a negotiation advisor. Respond with ONLY a JSON object, no markdown, no extra text: {\"recommendation\":\"sign|negotiate|do_not_sign\",\"summary\":\"two sentences\",\"redlines\":\"suggested replacement wording for the problem clauses, short, in one string\"}"
        }
      },
      "id": "59738d56-5427-4aea-92cb-8f7ad779284a",
      "name": "Negotiation Advisor Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        2460,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "9e150d1c-c879-4cfc-b955-2e234f7af993",
      "name": "Negotiation Advisor Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        2440,
        440
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Compliance').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nlet rec = ['sign', 'negotiate', 'do_not_sign'].includes(r.recommendation) ? r.recommendation : 'negotiate';\nif (rec === 'sign' && (ctx.compliant === 'no' || ctx.risk_level === 'high')) rec = 'negotiate';\nreturn { json: {\n  contract_id: ctx.contract_id,\n  name: ctx.name,\n  counterparty: ctx.counterparty,\n  risk_level: ctx.risk_level,\n  compliant: ctx.compliant,\n  violations: ctx.violations,\n  recommendation: rec,\n  status: rec === 'sign' ? 'cleared' : 'needs_legal',\n  summary: String(r.summary || '').slice(0, 400),\n  redlines: String(r.redlines || '').slice(0, 800)\n}};"
      },
      "id": "51cc026f-1c30-499b-8800-560062b32b13",
      "name": "Parse Advice",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2710,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "b2105efe-0d71-4081-9b14-c494bf6a4419",
              "leftValue": "={{ $json.status }}",
              "rightValue": "needs_legal",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "5ae350cf-f216-42aa-a65e-2881f4e63493",
      "name": "Needs Legal?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2880,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "legal-review",
          "mode": "name"
        },
        "text": "=:scales: *Contract needs legal review: {{ $json.name }}*\nCounterparty: {{ $json.counterparty }}\nRisk: {{ $json.risk_level }} | Policy compliant: {{ $json.compliant }}\nRecommendation: {{ $json.recommendation }}\n{{ $json.summary }}",
        "otherOptions": {}
      },
      "id": "3cb6702e-da07-47ff-869a-cf5f925d71c8",
      "name": "Slack Legal Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        3160,
        40
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "fromEmail": "YOUR_EMAIL@gmail.com",
        "toEmail": "YOUR_EMAIL@gmail.com",
        "subject": "=Contract review: {{ $json.name }} ({{ $json.recommendation }})",
        "emailFormat": "text",
        "text": "=Summary: {{ $json.summary }}\n\nPolicy violations: {{ $json.violations }}\n\nSuggested redlines:\n{{ $json.redlines }}",
        "options": {}
      },
      "id": "7100a585-7a74-477e-b9fd-32d87c6052c1",
      "name": "Send Legal Summary",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [
        3160,
        240
      ],
      "credentials": {
        "smtp": {
          "id": "REPLACE_SMTP",
          "name": "SMTP account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "contract_reviews",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "contract_id": "={{ $json.contract_id }}",
            "name": "={{ $json.name }}",
            "counterparty": "={{ $json.counterparty }}",
            "risk_level": "={{ $json.risk_level }}",
            "compliant": "={{ $json.compliant }}",
            "recommendation": "={{ $json.recommendation }}",
            "status": "={{ $json.status }}",
            "summary": "={{ $json.summary }}",
            "redlines": "={{ $json.redlines }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "contract_id",
              "displayName": "contract_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "name",
              "displayName": "name",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "counterparty",
              "displayName": "counterparty",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_level",
              "displayName": "risk_level",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "compliant",
              "displayName": "compliant",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "recommendation",
              "displayName": "recommendation",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "summary",
              "displayName": "summary",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "redlines",
              "displayName": "redlines",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "3e4c50f9-cc33-4962-b702-5c8d0dea21da",
      "name": "Save Review",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        3160,
        440
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "Clause Extractor Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Clause Extractor Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Clause Extractor Agent": {
      "main": [
        [
          {
            "node": "Parse Clauses",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Risk Analyst Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Risk Analyst Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Risk Analyst Agent": {
      "main": [
        [
          {
            "node": "Parse Risk",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Compliance Checker Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Compliance Checker Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Compliance Checker Agent": {
      "main": [
        [
          {
            "node": "Parse Compliance",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Negotiation Advisor Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Negotiation Advisor Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Negotiation Advisor Agent": {
      "main": [
        [
          {
            "node": "Parse Advice",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Contracts",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Contracts": {
      "main": [
        [
          {
            "node": "Get Policies",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Policies": {
      "main": [
        [
          {
            "node": "Attach Policies",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Attach Policies": {
      "main": [
        [
          {
            "node": "Clause Extractor Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Clauses": {
      "main": [
        [
          {
            "node": "Risk Analyst Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Risk": {
      "main": [
        [
          {
            "node": "Compliance Checker Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Compliance": {
      "main": [
        [
          {
            "node": "Negotiation Advisor Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Advice": {
      "main": [
        [
          {
            "node": "Needs Legal?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Needs Legal?": {
      "main": [
        [
          {
            "node": "Slack Legal Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Send Legal Summary",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Review",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Review",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
