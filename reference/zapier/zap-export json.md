# zap-export.json

```json
{
  "metadata": { "version": 2 },
  "zaps": [
    {
      "id": 1,
      "title": "Orders - PRODUCTS: Shopify > Code > Looping > Code > Paths > Asana",
      "nodes": {
        "1": {
          "id": 1,
          "paused": false,
          "type_of": "read",
          "params": {},
          "meta": {
            "$editor": {
              "added_step_correlation_id": "1",
              "has_automatic_issues": false
            },
            "timezone": "America/Los_Angeles"
          },
          "triple_stores": {
            "copied_from": null,
            "created_by": null,
            "polling_interval_override": 0,
            "block_and_release_limit_override": 0,
            "spread_tasks": 1
          },
          "parent_id": null,
          "root_id": null,
          "action": "new_order_hook_v2",
          "selected_api": "ShopifyCLIAPI@6.1.1",
          "title": "Orders - PRODUCTS: Shopify > Code > Looping > Code > Paths > Asana",
          "authentication_id": "<REDACTED — Shopify Zapier credential id>"
        },
        "2": {
          "id": 2,
          "paused": true,
          "type_of": "filter",
          "params": {
            "filter_criteria": [
              {
                "id": 3894260820715527,
                "group": 7129187410957887,
                "key": "1__id",
                "value": "",
                "match": "iexist",
                "action": "continue"
              }
            ]
          },
          "meta": { "$editor": { "has_automatic_issues": false } },
          "triple_stores": {
            "copied_from": null,
            "created_by": null,
            "polling_interval_override": 0,
            "block_and_release_limit_override": 0,
            "spread_tasks": 1
          },
          "parent_id": 1,
          "root_id": 1,
          "action": "filter",
          "selected_api": "FilterAPI",
          "title": null,
          "authentication_id": null
        },
        "3": {
          "id": 3,
          "paused": true,
          "type_of": "write",
          "params": { "delay_for_value": "1.5", "delay_for_unit": "minutes" },
          "meta": {
            "$editor": { "has_automatic_issues": false },
            "parammap": {}
          },
          "triple_stores": {
            "copied_from": null,
            "created_by": null,
            "polling_interval_override": 0,
            "block_and_release_limit_override": 0,
            "spread_tasks": 1
          },
          "parent_id": 2,
          "root_id": 1,
          "action": "delay_for",
          "selected_api": "DelayCLIAPI@1.1.1",
          "title": null,
          "authentication_id": null
        },
        "4": {
          "id": 4,
          "paused": true,
          "type_of": "write",
          "params": {
            "fail_on_errors": "false",
            "method": "GET",
            "url": "https://blue-delta-jeans.myshopify.com/admin/api/2024-10/orders/{{1__id}}.json"
          },
          "meta": {
            "$editor": {
              "has_automatic_issues": false,
              "replaced_step_id": "_GEN_1739988540101"
            },
            "parammap": {},
            "stepTitle": "GET Shopify API - Find Order by ID"
          },
          "triple_stores": {
            "copied_from": null,
            "created_by": null,
            "polling_interval_override": 0,
            "block_and_release_limit_override": 0,
            "spread_tasks": 1
          },
          "parent_id": 3,
          "root_id": 1,
          "action": "_zap_raw_request",
          "selected_api": "ShopifyCLIAPI@6.1.1",
          "title": null,
          "authentication_id": "<REDACTED — Shopify Zapier credential id>"
        },
        "5": {
          "id": 5,
          "paused": true,
          "type_of": "write",
          "params": {
            "input": {
              "RAW": "{{4__response__body}}",
              "TAGS": "{{=gives['281795060']['response']['data']['order']['tags']}}"
            },
            "code": "// Input variable: TAGS\nlet TAGS = inputData.TAGS || \"\";\n\n// Uppercase for consistent matching\nTAGS = TAGS.toString().toUpperCase();\n\n// Default\nlet RUSH = \"\";\n\n// Check for specific matches\nif (TAGS.includes(\"RUSH ROYAL\")) {\n  RUSH = \" | RUSH ROYAL\";\n} else if (TAGS.includes(\"RUSH COBALT\")) {\n  RUSH = \" | RUSH COBALT\";\n}\n\nlet RAW;\ntry {\n  RAW = JSON.parse(inputData.RAW);\n} catch (error) {\n  throw new Error(\"Invalid JSON in inputData.RAW\");\n}\n\n// Ensure RAW has an order object\nif (!RAW.order) throw new Error(\"Invalid data structure: Missing 'order' object\");\n\nlet Order = RAW.order; // Extract order object\n\n// Determine Order Type\nlet OrderType = Order.cart_token ? \"Online\" : \"Direct\";\n\n// Customer Details\nlet CustomerOrdersCount = Order.customer?.orders_count || 0;\nlet CustomerNew = CustomerOrdersCount > 1 ? \"No\" : \"Yes\";\n\n// Extract and Filter Line Items\nlet LI = (Order.line_items || []).filter(item =>\n  item.sku?.startsWith(\"W-\") ||\n  item.sku?.startsWith(\"M-\") ||\n  item.sku?.startsWith(\"CB\") ||\n  item.sku?.startsWith(\"VID-GIFT-\") ||\n  item.sku?.startsWith(\"SHOE-\")\n).map(item => JSON.stringify(item));\n\nlet LIcount = LI.length;\n\n// Function to Clean Up Note Attributes\nfunction cleanNoteAttributes(NA) {\n  if (!Array.isArray(NA)) return [];\n  \n  return NA.map(attr => ({\n    name: attr.name,\n    value: attr.name === \"__aftersell_tamper_proof\" ? \"\" : attr.value\n  }));\n}\n\n// Extract and Clean Note Attributes\nlet NA = cleanNoteAttributes(Order.note_attributes);\nlet NAcount = NA.length;\n\n// Convert Note Attributes to Extras Object\nlet Extras = {};\nNA.forEach(attr => {\n  if (attr.name) Extras[attr.name] = attr.value;\n});\n\n// BDJ User Data Extraction\nlet BDJUserData = {};\nif (typeof Extras[\"BDJ User Data\"] === \"string\") {\n  Extras[\"BDJ User Data\"].split(\"\\n\").forEach(line => {\n    let [key, ...valueParts] = line.split(\": \");\n    let value = valueParts.join(\": \"); // Handles cases where value contains `:`\n    if (key && value) BDJUserData[key.trim()] = value.trim();\n  });\n  delete Extras[\"BDJ User Data\"];\n}\nlet BDJUserDataCount = Object.keys(BDJUserData).length;\n\n// Extract Order Note\nlet Note = Order.note || \"\";\n\n// Final Output\noutput = [{\n  RUSH,\n  NAcount,\n  LIcount,\n  CustomerOrdersCount,\n  BDJUserDataCount,\n  CustomerNew,\n  OrderType,\n  Extras,\n  BDJUserData,\n  Note,\n  LI,\n  NA\n}];\n"
          },
          "meta": {
            "$editor": {
              "added_step_correlation_id": "51a48362-aed0-4e5c-b387-c52982bb1198",
              "has_automatic_issues": false
            },
            "parammap": {},
            "stepTitle": "Run Javascript: DO NOT DELETE!!!"
          },
          "triple_stores": {
            "copied_from": null,
            "created_by": null,
            "polling_interval_override": 0,
            "block_and_release_limit_override": 0,
            "spread_tasks": 1
          },
          "parent_id": 4,
          "root_id": 1,
          "action": "01929fad-d3dd-62c2-52ed-7868d5fcc691",
          "selected_api": "CodeCLIAPI@1.0.1",
          "title": "Run Javascript in Code by Zapier - DO NOT DELETE!!!",
          "authentication_id": null
        },
        "6": {
          "id": 6,
          "paused": true,
          "type_of": "filter",
          "params": {
            "filter_criteria": [
              {
                "id": 3008391963825563,
                "group": 7110484325007600,
                "key": "5__LIcount",
                "value": "0",
                "match": "igreater",
                "action": "continue"
              }
            ]
          },
          "meta": {
            "$editor": {
              "added_step_correlation_id": "a3b0fdfd-2664-4a80-8e79-16500627fc41",
              "has_automatic_issues": false
            },
            "stepTitle": "Only continue if...Line Item Count > 0"
          },
          "triple_stores": {
            "copied_from": null,
            "created_by": null,
            "polling_interval_override": 0,
            "block_and_release_limit_override": 0,
            "spread_tasks": 1
          },
          "parent_id": 5,
          "root_id": 1,
          "action": "filter",
          "selected_api": "FilterAPI",
          "title": "Only continue if...Line Item Count > 0",
          "authentication_id": null
        }
      }
    }
  ]
}
```