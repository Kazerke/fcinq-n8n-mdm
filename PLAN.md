# Moodboard Integration Plan
## POC Maisons du Monde Workflow Update

---

## Overview

### Objective
Integrate a moodboard generator into the existing workflow (ID: `WgDlTroMpNzmsHA5`) to combine multiple product images into a single collage before sending to Fal.ai API.

### Current Behavior
- AI Agent selects 6-9 products from catalogue
- Fal.ai receives 6 separate images: room photo + 5 product photos
- Results are suboptimal

### Target Behavior
- AI Agent selects 6-9 products from catalogue
- Products are combined into a single moodboard image (grid layout)
- Fal.ai receives 2 images: room photo + moodboard (all products)
- Better results due to unified product reference

---

## Workflow Context

### Current Flow (Relevant Section)
```
Data Cleanup (3648)
    ↓
Map Fields for Fal (3776)
    ↓
API Fal.ai (3984)
```

### Target Flow
```
Data Cleanup (3648)
    ↓
[NEW] Configure Moodboard Images (3680)
    ↓
[NEW] Build Moodboard HTML (3700)
    ↓
[NEW] Generate Moodboard Screenshot (3720)
    ↓
[NEW] Convert to Data URI (3740)
    ↓
[MODIFIED] Map Fields for Fal (3776)
    ↓
API Fal.ai (3984)
```

---

## Implementation Steps

### Step 1: Add "Configure Moodboard Images" Node

**Node Type:** `n8n-nodes-base.set`
**Position:** `[3680, 832]`
**Node Name:** `Configure Moodboard Images`

**Configuration:**
```json
{
  "parameters": {
    "assignments": {
      "assignments": [
        {
          "id": "moodboard-img-urls",
          "name": "imageUrls",
          "type": "array",
          "value": "={{ $json.productUrls }}"
        },
        {
          "id": "moodboard-grid-cols",
          "name": "gridColumns",
          "type": "number",
          "value": 3
        }
      ]
    },
    "options": {}
  }
}
```

**Purpose:** Extracts product URLs from Data Cleanup and sets grid configuration.

**Validation Required:**
- Verify `$json.productUrls` exists from Data Cleanup node
- Confirm array contains 6-9 items

---

### Step 2: Add "Build Moodboard HTML" Node

**Node Type:** `n8n-nodes-base.code`
**Position:** `[3700, 832]`
**Node Name:** `Build Moodboard HTML`

**Configuration:**
```json
{
  "parameters": {
    "jsCode": "const imageUrls = $input.first().json.imageUrls;\nconst columns = $input.first().json.gridColumns || 3;\n\nconst imageCount = imageUrls.length;\nconst rows = Math.ceil(imageCount / columns);\nconst gap = 10;\n\n// Calculate dimensions based on number of images\nconst imageWidth = 380;\nconst imageHeight = 380;\nconst width = (imageWidth * columns) + (gap * (columns + 1));\nconst height = (imageHeight * rows) + (gap * (rows + 1));\n\nconst html = `<!DOCTYPE html>\n<html>\n<head>\n  <meta charset=\"UTF-8\">\n  <style>\n    * { margin: 0; padding: 0; box-sizing: border-box; }\n    body {\n      width: ${width}px;\n      height: ${height}px;\n      background: #ffffff;\n      display: flex;\n      align-items: center;\n      justify-content: center;\n      padding: ${gap}px;\n    }\n    .moodboard {\n      display: grid;\n      grid-template-columns: repeat(${columns}, 1fr);\n      gap: ${gap}px;\n      width: 100%;\n      height: 100%;\n    }\n    .image-container {\n      width: 100%;\n      height: 100%;\n      overflow: hidden;\n      border-radius: 4px;\n    }\n    .image-container img {\n      width: 100%;\n      height: 100%;\n      object-fit: cover;\n      display: block;\n    }\n  </style>\n</head>\n<body>\n  <div class=\"moodboard\">\n    ${imageUrls.map(url => `<div class=\"image-container\"><img src=\"${url}\" alt=\"Moodboard image\" /></div>`).join('')}\n  </div>\n</body>\n</html>`;\n\nreturn [{ json: { html, imageCount, width, height, columns, rows } }];"
  }
}
```

**Purpose:** Generates HTML with CSS Grid layout for product images.

**Output:**
- `html` (string): Complete HTML document
- `width` (number): Canvas width in pixels
- `height` (number): Canvas height in pixels
- `imageCount` (number): Number of products
- `columns` (number): Grid columns
- `rows` (number): Grid rows

---

### Step 3: Add "Generate Moodboard Screenshot" Node

**Node Type:** `n8n-nodes-base.httpRequest`
**Position:** `[3720, 832]`
**Node Name:** `Generate Moodboard Screenshot`

**Configuration:**
```json
{
  "parameters": {
    "method": "POST",
    "url": "https://production-sfo.browserless.io/screenshot?token={{ $env.BROWSERLESS_TOKEN }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {
          "name": "Content-Type",
          "value": "application/json"
        },
        {
          "name": "Cache-Control",
          "value": "no-cache"
        }
      ]
    },
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ { html: $json.html, options: { type: 'png', fullPage: false }, viewport: { width: $json.width, height: $json.height }, gotoOptions: { waitUntil: 'networkidle2' } } }}",
    "options": {
      "response": {
        "response": {
          "responseFormat": "file"
        }
      }
    }
  }
}
```

**⚠️ CRITICAL Configuration:**
- **Response Format:** MUST be set to `"file"` to receive binary data
- **Token:** Replace `{{ $env.BROWSERLESS_TOKEN }}` with actual token or use environment variable
- **waitUntil:** `networkidle2` ensures all images load before screenshot

**Purpose:** Converts HTML to PNG image via Browserless API.

**Output:** Binary PNG file in `$binary` data.

---

### Step 4: Add "Convert to Data URI" Node

**Node Type:** `n8n-nodes-base.code`
**Position:** `[3740, 832]`
**Node Name:** `Convert to Data URI`

**Configuration:**
```json
{
  "parameters": {
    "jsCode": "// Get the binary data from Browserless response\nconst binaryPropertyName = Object.keys($input.first().binary)[0];\nconst binaryData = await this.helpers.getBinaryDataBuffer(0, binaryPropertyName);\n\n// Convert to base64 data URI\nconst base64Image = binaryData.toString('base64');\nconst dataUri = `data:image/png;base64,${base64Image}`;\n\n// Pass through all data from Data Cleanup node + add moodboard URL\nconst previousData = $('Data Cleanup').item.json;\n\nreturn [{\n  json: {\n    ...previousData,\n    moodboardUrl: dataUri\n  }\n}];"
  }
}
```

**Purpose:** Converts binary PNG to base64 data URI format accepted by Fal.ai.

**Output:** All data from Data Cleanup + `moodboardUrl` (string) containing data URI.

**⚠️ CRITICAL:** Must reference `$('Data Cleanup').item.json` to preserve all previous data (roomPhoto, products, legend, style, etc.).

---

### Step 5: Modify "Map Fields for Fal" Node

**Node ID:** `aec62b60-7b5b-4eb5-b354-8f803871c1e9`
**Action:** Update existing node configuration

**Current Configuration:**
```json
{
  "assignments": {
    "assignments": [
      {
        "id": "40dae0b8-fe83-43ae-b0fd-aa331c65a0c4",
        "name": "image_urls",
        "type": "array",
        "value": "={{ [$json.roomPhoto].concat($json.productUrls.slice(0, 5)) }}"
      }
    ]
  }
}
```

**New Configuration:**
```json
{
  "assignments": {
    "assignments": [
      {
        "id": "fal-prompt-field",
        "name": "prompt",
        "type": "string",
        "value": "=Interior design ultra-realistic render of the room shown in the first reference photo.\nRespect the exact orientation, dimensions, and architectural features of the room.\n\nUse the furniture pieces from Maisons du Monde catalog shown in the following reference images.\nYou MUST use these specific products without ANY modifications to their:\n- Colors and finishes\n- Shapes and proportions  \n- Materials and textures\n- Design details\n\nArrange these exact pieces naturally in the room with proper:\n- Scale and perspective\n- Spatial relationships\n- Realistic placement\n\nYou may add complementary elements:\n- Plants in ceramic pots\n- Framed art or prints on walls\n- Decorative accents\n\nStyle: {{ $json.style }}\nLighting: Bright natural daylight from windows, sharp and crisp, professional interior photography quality (AD, Elle Decoration standard)\nMood: Aspirational, refined, cozy yet contemporary\nRender quality: Ultra-realistic, rich textures, perfect proportions, magazine-worthy\nCamera: Wide angle from corner showing depth and window"
      },
      {
        "id": "fal-image-urls-field",
        "name": "image_urls",
        "type": "array",
        "value": "={{ [$json.roomPhoto, $json.moodboardUrl] }}"
      }
    ]
  }
}
```

**⚠️ CRITICAL CHANGE:**
- **OLD:** `[$json.roomPhoto].concat($json.productUrls.slice(0, 5))` - 6 images
- **NEW:** `[$json.roomPhoto, $json.moodboardUrl]` - 2 images (room + moodboard)

---

### Step 6: Update Connections

**Use `n8n_update_partial_workflow` with batch operations:**

```json
{
  "id": "WgDlTroMpNzmsHA5",
  "operations": [
    {
      "type": "addNode",
      "node": {
        "parameters": {
          "assignments": {
            "assignments": [
              {
                "id": "moodboard-img-urls",
                "name": "imageUrls",
                "type": "array",
                "value": "={{ $json.productUrls }}"
              },
              {
                "id": "moodboard-grid-cols",
                "name": "gridColumns",
                "type": "number",
                "value": 3
              }
            ]
          },
          "options": {}
        },
        "id": "moodboard-config-node",
        "name": "Configure Moodboard Images",
        "type": "n8n-nodes-base.set",
        "typeVersion": 3.4,
        "position": [3680, 832]
      }
    },
    {
      "type": "addNode",
      "node": {
        "parameters": {
          "jsCode": "const imageUrls = $input.first().json.imageUrls;\nconst columns = $input.first().json.gridColumns || 3;\n\nconst imageCount = imageUrls.length;\nconst rows = Math.ceil(imageCount / columns);\nconst gap = 10;\n\nconst imageWidth = 380;\nconst imageHeight = 380;\nconst width = (imageWidth * columns) + (gap * (columns + 1));\nconst height = (imageHeight * rows) + (gap * (rows + 1));\n\nconst html = `<!DOCTYPE html>\n<html>\n<head>\n  <meta charset=\"UTF-8\">\n  <style>\n    * { margin: 0; padding: 0; box-sizing: border-box; }\n    body {\n      width: ${width}px;\n      height: ${height}px;\n      background: #ffffff;\n      display: flex;\n      align-items: center;\n      justify-content: center;\n      padding: ${gap}px;\n    }\n    .moodboard {\n      display: grid;\n      grid-template-columns: repeat(${columns}, 1fr);\n      gap: ${gap}px;\n      width: 100%;\n      height: 100%;\n    }\n    .image-container {\n      width: 100%;\n      height: 100%;\n      overflow: hidden;\n      border-radius: 4px;\n    }\n    .image-container img {\n      width: 100%;\n      height: 100%;\n      object-fit: cover;\n      display: block;\n    }\n  </style>\n</head>\n<body>\n  <div class=\"moodboard\">\n    ${imageUrls.map(url => `<div class=\"image-container\"><img src=\"${url}\" alt=\"Moodboard image\" /></div>`).join('')}\n  </div>\n</body>\n</html>`;\n\nreturn [{ json: { html, imageCount, width, height, columns, rows } }];"
        },
        "id": "moodboard-html-node",
        "name": "Build Moodboard HTML",
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [3700, 832]
      }
    },
    {
      "type": "addNode",
      "node": {
        "parameters": {
          "method": "POST",
          "url": "https://production-sfo.browserless.io/screenshot?token=BROWSERLESS_API_TOKEN_HERE",
          "sendHeaders": true,
          "headerParameters": {
            "parameters": [
              {
                "name": "Content-Type",
                "value": "application/json"
              },
              {
                "name": "Cache-Control",
                "value": "no-cache"
              }
            ]
          },
          "sendBody": true,
          "specifyBody": "json",
          "jsonBody": "={{ { html: $json.html, options: { type: 'png', fullPage: false }, viewport: { width: $json.width, height: $json.height }, gotoOptions: { waitUntil: 'networkidle2' } } }}",
          "options": {
            "response": {
              "response": {
                "responseFormat": "file"
              }
            }
          }
        },
        "id": "moodboard-screenshot-node",
        "name": "Generate Moodboard Screenshot",
        "type": "n8n-nodes-base.httpRequest",
        "typeVersion": 4.2,
        "position": [3720, 832]
      }
    },
    {
      "type": "addNode",
      "node": {
        "parameters": {
          "jsCode": "const binaryPropertyName = Object.keys($input.first().binary)[0];\nconst binaryData = await this.helpers.getBinaryDataBuffer(0, binaryPropertyName);\n\nconst base64Image = binaryData.toString('base64');\nconst dataUri = `data:image/png;base64,${base64Image}`;\n\nconst previousData = $('Data Cleanup').item.json;\n\nreturn [{\n  json: {\n    ...previousData,\n    moodboardUrl: dataUri\n  }\n}];"
        },
        "id": "moodboard-datauri-node",
        "name": "Convert to Data URI",
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [3740, 832]
      }
    },
    {
      "type": "removeConnection",
      "source": "6f8ddd23-3a52-4775-9a3d-854b6346da6d",
      "target": "aec62b60-7b5b-4eb5-b354-8f803871c1e9",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "addConnection",
      "source": "6f8ddd23-3a52-4775-9a3d-854b6346da6d",
      "target": "moodboard-config-node",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "addConnection",
      "source": "moodboard-config-node",
      "target": "moodboard-html-node",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "addConnection",
      "source": "moodboard-html-node",
      "target": "moodboard-screenshot-node",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "addConnection",
      "source": "moodboard-screenshot-node",
      "target": "moodboard-datauri-node",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "addConnection",
      "source": "moodboard-datauri-node",
      "target": "aec62b60-7b5b-4eb5-b354-8f803871c1e9",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "updateNode",
      "nodeId": "aec62b60-7b5b-4eb5-b354-8f803871c1e9",
      "changes": {
        "parameters": {
          "assignments": {
            "assignments": [
              {
                "id": "fal-prompt-field",
                "name": "prompt",
                "type": "string",
                "value": "=Interior design ultra-realistic render of the room shown in the first reference photo.\nRespect the exact orientation, dimensions, and architectural features of the room.\n\nUse the furniture pieces from Maisons du Monde catalog shown in the following reference images.\nYou MUST use these specific products without ANY modifications to their:\n- Colors and finishes\n- Shapes and proportions  \n- Materials and textures\n- Design details\n\nArrange these exact pieces naturally in the room with proper:\n- Scale and perspective\n- Spatial relationships\n- Realistic placement\n\nYou may add complementary elements:\n- Plants in ceramic pots\n- Framed art or prints on walls\n- Decorative accents\n\nStyle: {{ $json.style }}\nLighting: Bright natural daylight from windows, sharp and crisp, professional interior photography quality (AD, Elle Decoration standard)\nMood: Aspirational, refined, cozy yet contemporary\nRender quality: Ultra-realistic, rich textures, perfect proportions, magazine-worthy\nCamera: Wide angle from corner showing depth and window"
              },
              {
                "id": "fal-image-urls-field",
                "name": "image_urls",
                "type": "array",
                "value": "={{ [$json.roomPhoto, $json.moodboardUrl] }}"
              }
            ]
          }
        }
      }
    }
  ]
}
```

**⚠️ CRITICAL:**
- Replace `BROWSERLESS_API_TOKEN_HERE` with actual Browserless API token
- All operations must be in a single `n8n_update_partial_workflow` call (batch operation)
- Connection syntax uses four separate string parameters (see AGENTS.md line 185-194)

---

## Validation Steps

### Pre-Deployment Validation

1. **Validate Individual Nodes:**
```javascript
// Validate each new node configuration
validate_node_minimal('n8n-nodes-base.set', configureImagesConfig)
validate_node_minimal('n8n-nodes-base.code', buildHtmlConfig)
validate_node_minimal('n8n-nodes-base.httpRequest', browserlessConfig)
validate_node_minimal('n8n-nodes-base.code', dataUriConfig)
```

2. **Validate Complete Workflow:**
```javascript
n8n_validate_workflow({id: "WgDlTroMpNzmsHA5"})
```

### Post-Deployment Validation

1. **Check Workflow Structure:**
```javascript
n8n_get_workflow_structure({id: "WgDlTroMpNzmsHA5"})
```

2. **Validate Connections:**
```javascript
validate_workflow_connections(workflowJson)
```

3. **Auto-fix if needed:**
```javascript
n8n_autofix_workflow({id: "WgDlTroMpNzmsHA5", applyFixes: true})
```

---

## Testing Procedure

### Test Execution

1. **Manual Trigger Test:**
   - Activate workflow
   - Click manual trigger
   - Monitor execution through each node

2. **Check Node Outputs:**
   - **Data Cleanup:** Verify `productUrls` array exists (6-9 items)
   - **Configure Moodboard Images:** Verify `imageUrls` and `gridColumns` are set
   - **Build Moodboard HTML:** Verify `html` string is generated with correct dimensions
   - **Generate Moodboard Screenshot:** Verify binary PNG data is returned
   - **Convert to Data URI:** Verify `moodboardUrl` starts with `data:image/png;base64,`
   - **Map Fields for Fal:** Verify `image_urls` contains exactly 2 items

3. **Fal.ai API Response:**
   - Should receive both images (room + moodboard)
   - Check rendering quality improvement

### Monitor Executions

```javascript
n8n_list_executions({
  workflowId: "WgDlTroMpNzmsHA5",
  status: "error",
  limit: 10
})
```

---

## Troubleshooting

### Common Issues

#### Issue 1: Binary Data Not Found
**Symptom:** Convert to Data URI node fails with "Cannot read property 'binary' of undefined"
**Solution:**
- Verify Generate Moodboard Screenshot has `responseFormat: "file"`
- Check HTTP Request completed successfully

#### Issue 2: Data URI Too Large
**Symptom:** Fal.ai API returns 413 or timeout
**Solution:**
- Reduce moodboard dimensions (change imageWidth/imageHeight from 380 to 300)
- Reduce number of products (slice array to 6 max)

#### Issue 3: Browserless API Error
**Symptom:** 401 Unauthorized or 403 Forbidden
**Solution:**
- Verify Browserless API token is correct
- Check token has sufficient credits
- Verify URL format: `https://production-sfo.browserless.io/screenshot?token=YOUR_TOKEN`

#### Issue 4: Images Not Loading in Moodboard
**Symptom:** Blank or broken images in moodboard
**Solution:**
- Verify product URLs are publicly accessible
- Check `waitUntil: 'networkidle2'` is set in gotoOptions
- Increase timeout if needed

#### Issue 5: Connection Errors
**Symptom:** "Source node not found" or "Target node not found"
**Solution:**
- Use exact node IDs from workflow (not names)
- Verify all four connection parameters are strings
- Check nodes exist before creating connections

---

## Rollback Plan

If issues occur, revert to previous state:

1. **Remove New Connections:**
```json
{
  "operations": [
    {
      "type": "removeConnection",
      "source": "6f8ddd23-3a52-4775-9a3d-854b6346da6d",
      "target": "moodboard-config-node",
      "sourcePort": "main",
      "targetPort": "main"
    },
    {
      "type": "addConnection",
      "source": "6f8ddd23-3a52-4775-9a3d-854b6346da6d",
      "target": "aec62b60-7b5b-4eb5-b354-8f803871c1e9",
      "sourcePort": "main",
      "targetPort": "main"
    }
  ]
}
```

2. **Restore Original "Map Fields for Fal" Config:**
```json
{
  "type": "updateNode",
  "nodeId": "aec62b60-7b5b-4eb5-b354-8f803871c1e9",
  "changes": {
    "parameters": {
      "assignments": {
        "assignments": [
          {
            "id": "40dae0b8-fe83-43ae-b0fd-aa331c65a0c4",
            "name": "image_urls",
            "type": "array",
            "value": "={{ [$json.roomPhoto].concat($json.productUrls.slice(0, 5)) }}"
          }
        ]
      }
    }
  }
}
```

3. **Remove New Nodes:**
```json
{
  "operations": [
    {"type": "removeNode", "nodeId": "moodboard-config-node"},
    {"type": "removeNode", "nodeId": "moodboard-html-node"},
    {"type": "removeNode", "nodeId": "moodboard-screenshot-node"},
    {"type": "removeNode", "nodeId": "moodboard-datauri-node"}
  ]
}
```

---

## Success Criteria

✅ Workflow validates without errors
✅ All 4 new nodes are created and connected
✅ "Map Fields for Fal" receives `moodboardUrl` with data URI
✅ Fal.ai API receives exactly 2 images (not 6)
✅ Moodboard contains all 6-9 products in grid layout
✅ Final rendered room image quality improves
✅ Pinterest pin publishes successfully

---

## Notes

- **Browserless Token:** User confirms token is already available
- **Data URI Format:** Fal.ai officially supports base64 data URIs per documentation
- **Performance:** Base64 encoding adds ~33% size overhead but acceptable for images <2MB
- **Grid Layout:** 3 columns works for 6-9 items (2-3 rows)
- **Image Dimensions:** 380x380px per product = ~1160x1160px for 3x3 grid

---

## References

- Fal.ai Documentation: Accepts file URLs and Base64 data URIs
- Browserless API: https://docs.browserless.io/screenshot
- N8N Best Practices: See AGENTS.md
- Original Moodboard Workflow ID: 9eE9hkGT4FgnYpLz
- Target Workflow ID: WgDlTroMpNzmsHA5
