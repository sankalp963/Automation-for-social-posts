# Social Media Automation Flow Documentation

This document provides detailed documentation for the **Zapier-powered automation workflow** that enables seamless social media post scheduling using **Airtable, Python (via Code by Zapier), Replicate AI, Placid, Google Drive, and Buffer**.

---

## 📌 Overview

The workflow automates the end-to-end process of creating, designing, and scheduling posts across multiple platforms. It starts with record creation in Airtable, processes content and image generation with AI, enriches visuals via Placid, manages storage in Google Drive, and finally schedules posts to **Instagram and Pinterest via Buffer**. The Airtable record is updated at key checkpoints for tracking.

---

## 📂 Airtable Setup

Two Airtable tables are used:

### 1. **Data Table**
Stores post-specific content and scheduling details.

**Fields:**
1. **Topic** – The subject of the post (e.g., motivational quote, blog highlight, product).  
2. **Platform** – Defines where the post should be published (Instagram, Pinterest, etc.).  
3. **Social Links** – Pulled from Info table, links to profiles or landing pages.  
4. **Image Details** – AI image context or details for image generation.  
5. **Image Context** – Extra description for how the image should look (background, tone, theme).  
6. **Image Footer** – Footer content like contact details (from Info table).  
7. **Logo** – Branding logo (from Info table).  
8. **Reference Image** – Example image for guiding AI generation.  
9. **Post Type** – Defines type (single image, carousel, infographic).  
10. **Schedule Date & Time** – Time when the post should be scheduled.  
11. **Caption** – Social media caption.  
12. **Blog Body** – Long-form content for blog or reference.  
13. **Tags** – Hashtags for the post.  
14. **Prompt** – AI prompt for Replicate.  
15. **Status** – Tracks workflow status (Draft → Queue → Scheduled).  
16. **Last Modified Time** – Timestamp for last edit.  
17. **Info Link** – Link reference to Info table data.  

### 2. **Info Table**
Holds fixed reference details used across posts.

**Fields:**
1. **Social Links** – Links for accounts/brand pages.  
2. **Logo** – Branding logo file.  
3. **Image Footer (Contact No.)** – Business contact or support info.  
4. **Data2** – Extra metadata if required.  
5. **Record ID** – Unique identifier for data linkages.  

---

## 🔄 Zapier Automation Flow

### **Step 1 – Airtable Trigger**
- **What happens:** Whenever a new record is added or updated in the Data table, Zapier listens for the change.  
- **Purpose:** Ensures the workflow starts automatically when a new post idea or scheduled post is added.

### **Step 2 – Airtable Update (Draft → Queue)**
- **What happens:** The record status field is updated from **Draft** to **Queue**.  
- **Purpose:** Marks the post as ready for processing and prevents duplicate triggers.

### **Step 3 – Python (Code by Zapier)**
- **What happens:** The Zap runs custom Python code to connect with **Replicate AI** and generate an image.  
- **Inputs:** AI prompt, optional reference image, Replicate API token.  
- **Process:**
  1. Fetches the latest model version from Replicate.
  2. Builds payload with prompt and optional image.
  3. Sends request to Replicate’s API.
  4. Polls until the image is fully generated.
  5. Returns the final image URL.
- **Purpose:** Generates dynamic visuals tailored to the topic.

#### Code Implementation
```python
import os
import requests
import time

# --- Inputs from Zapier ---
prompt = input_data.get('prompt')
image = input_data.get('image_url')
REPLICATE_API_TOKEN = input_data.get('REPLICATE_API_TOKEN')

# --- Validate Inputs ---
if not prompt:
    return {"error": "Prompt is required."}
if not REPLICATE_API_TOKEN:
    return {"error": "Replicate API token is missing."}

# --- Headers for Replicate ---
headers = {
    "Authorization": f"Token {REPLICATE_API_TOKEN}",
    "Content-Type": "application/json"
}

# --- Step 1: Get latest version of flux-dev ---
model_info_resp = requests.get(
    "https://api.replicate.com/v1/models/black-forest-labs/flux-dev",
    headers=headers
)

if model_info_resp.status_code != 200:
    return {"error": "Failed to fetch model info", "details": model_info_resp.text}

model_info = model_info_resp.json()
latest_version_obj = model_info.get("latest_version")
if not latest_version_obj or not latest_version_obj.get("id"):
    return {"error": "Could not find a valid latest version ID."}

version_id = latest_version_obj["id"]

# --- Step 2: Build the payload ---
payload = {
    "version": version_id,
    "input": {
        "prompt": prompt,
        "output_format": "jpg",
        "safety_tolerance": 2,
        "prompt_upsampling": False,
        "aspect_ratio": "4:5"
    }
}

if image:
    payload["input"]["input_image"] = image

# --- Step 3: Start prediction ---
response = requests.post(
    "https://api.replicate.com/v1/predictions",
    json=payload,
    headers=headers
)

if response.status_code != 201:
    return {"error": f"Prediction request failed (status {response.status_code})", "details": response.json()}

prediction = response.json()
status_url = prediction.get("urls", {}).get("get")
if not status_url:
    return {"error": "Status URL not found in prediction response."}

# --- Step 4: Poll until generation is complete ---
while True:
    poll = requests.get(status_url, headers=headers).json()
    status = poll.get("status")
    if status == "succeeded":
        output = poll.get("output")
        image_url = output[0] if isinstance(output, list) else output
        break
    elif status == "failed":
        return {"error": "Image generation failed.", "details": poll}
    time.sleep(2)

# --- Final Output ---
return {"image_url": image_url}
```

### **Step 4 – Placid Image Processing**
- **What happens:** The generated image URL is sent to **Placid** where a template adds the brand logo and footer (contact number, etc.).
- **Purpose:** Ensures every image has consistent branding and looks professional.

### **Step 5 – Google Drive Upload**
- **What happens:** The branded image is uploaded to Google Drive.  
- **Purpose:** Provides centralized cloud storage and easy retrieval for publishing.

### **Step 6 – Google Drive Retrieve File**
- **What happens:** The uploaded image is retrieved using its file ID.  
- **Purpose:** Makes the file accessible to Buffer for posting.

### **Step 7 – Buffer (Instagram)**
- **What happens:** The image and caption are added to Buffer’s **Instagram queue**.  
- **Purpose:** Automates post scheduling on Instagram according to the predefined date/time.

### **Step 8 – Buffer (Pinterest)**
- **What happens:** The same post (image + caption) is added to **Pinterest queue** via Buffer.  
- **Purpose:** Expands post reach by scheduling to Pinterest automatically.

### **Step 9 – Airtable Update (Queue → Scheduled)**
- **What happens:** Airtable record status is updated from **Queue → Scheduled**.  
- **Purpose:** Provides full visibility into the publishing lifecycle.

---

## ✅ Benefits
- **Automation:** Removes manual work of creating and scheduling posts.
- **AI-Powered:** Content and images are dynamically generated.
- **Consistency:** Standardized branding via Placid.
- **Scalable:** Multi-platform scheduling via Buffer.
- **Tracking:** Airtable ensures complete transparency of post statuses.

---

## 📊 Workflow Summary Diagram (High-Level)

```
Airtable (Data + Info Tables)
   │
   ├──> Zapier Trigger (New/Updated Record)
   │       ↓
   ├──> Airtable Update (Draft → Queue)
   │       ↓
   ├──> Python Code (Replicate AI → Image URL)
   │       ↓
   ├──> Placid (Apply Branding)
   │       ↓
   ├──> Google Drive (Upload → Retrieve)
   │       ↓
   ├──> Buffer (Instagram + Pinterest Queue)
   │       ↓
   └──> Airtable Update (Queue → Scheduled)
```

---

## 🔑 Notes
- The **Info Table** centralizes branding assets for consistency.
- **Replicate AI** requires a valid API token (`REPLICATE_API_TOKEN`).
- **Buffer Integration** requires connected Instagram & Pinterest accounts.
- Error handling is built into Python code for resilience.

---

This document can be used as a **README.md** in the repository for developers and stakeholders to understand and maintain the automation system.

