import logger from "./logger.js";
import { CrmApi } from "./crm-auth.js";

// Tiny fetch wrapper
function fetchJSON(url, opts) {
  return fetch(url, opts).then(res => {
    if (!res.ok) {
      return res.text().then(t =>
        Promise.reject(new Error(`HTTP ${res.status} ${res.statusText}${t ? ` - ${t}` : ""}`))
      );
    }
    const ct = res.headers.get("content-type") || "";
    return ct.includes("application/json") ? res.json() : res.text();
  });
}

// Language helpers
function isHebrew(s){ return /[\u0590-\u05FF]/.test(s||""); }

// Scope gate: allow only Hiltech/business/product topics
function isInScope(text){
  const t = (text||"").toLowerCase();

  // Brand / business keywords (add your brands/services here)
  const brand = /(hiltech|hiltech\.co\.il|זוהו|zoho|crm|billing|support|invoice|order|purchase|return|shipping|warranty|quote|pricing|price|stock|מוצר|מוצרים|קטלוג|מחיר|תשלום|חשבונית|הזמנה|מלאי|אחריות|תמיכה|שירות|פיצ'רים|תכונות)/i;

  // Generic “global topics” we want to block even if Gemini could answer
  const obviousOut =
    /(world|global|top|best|history|who is|what is|explain|define|news|today|weather|movies|sports|celeb|politics|אומי|עולמי|היסטוריה|מי הוא|מה זה|חדשות|מזג אוויר|ספורט|פוליטיקה)\b/i;

  // If they explicitly mention Hiltech/your site/products/services → allow
  if (brand.test(t)) return true;

  // If it looks like a product/business ask (our earlier heuristic), allow
  const producty = /(product|catalog|price|cost|feature|benefit|compare|spec|sku|stock|available|in\s*stock|warranty|model|size|color|under|below|between|מוצר|מוצרים|קטלוג|מחיר|עלות|תכונות|השוואה|מפרט|מלאי|זמין|במלאי|אחריות|דגם|מידה|גודל|צבע|עד|פחות|בין)/i;
  if (producty.test(t)) return true;

  // Block obvious “general knowledge” asks
  if (obviousOut.test(t)) return false;

  // Default: block unless it clearly references our business
  return false;
}

// Optional budget parser (kept for product filtering)
function parseBudget(text) {
  if (!text) return null;
  const t = text.replace(/[,₹$€£₪]/g, "").toLowerCase();
  let m = t.match(/בין\s+(\d+(?:\.\d+)?)\s*(?:ל|עד)\s*(\d+(?:\.\d+)?)/i);
  if (m) return { min: parseFloat(m[1]), max: parseFloat(m[2]) };
  m = t.match(/\bbetween\s+(\d+(?:\.\d+)?)\s+(?:and|to)\s+(\d+(?:\.\d+)?)/i);
  if (m) return { min: parseFloat(m[1]), max: parseFloat(m[2]) };
  m = t.match(/(?:עד|פחות\s*מ|-)\s*(\d+(?:\.\d+)?)/i);
  if (m) return { max: parseFloat(m[1]) };
  m = t.match(/\b(under|below|less than|<=?|upto|up to)\s+(\d+(?:\.\d+)?)/i);
  if (m) return { max: parseFloat(m[2]) };
  m = t.match(/(?:מעל|יותר\s*מ)\s*(\d+(?:\.\d+)?)/i);
  if (m) return { min: parseFloat(m[1]) };
  m = t.match(/\b(over|above|greater than|>=?)\s+(\d+(?:\.\d+)?)/i);
  if (m) return { min: parseFloat(m[2]) };
  return null;
}

export default function handleFallback(request){
  logger.info("Handling fallback request " + JSON.stringify(request));
  const userInput = request.userInput || "";
  const he = isHebrew(userInput);
  const currency = he ? "₪" : "₹";

  // ⛔ Out-of-scope guard
  if (!isInScope(userInput)) {
    return Promise.resolve({
      message: he
        ? "אני עוזר לשאלות על Hiltech: מוצרים, שירותים, מחירים, זמינות ותמיכה. איך אפשר לעזור?"
        : "I’m here for Hiltech questions only—our products, services, pricing, availability, and support. How can I help?"
    });
  }

  const productIntent = true; // we’re in-scope; treat as business/product-oriented
  const budget = parseBudget(userInput);
  const api = new CrmApi(request);
  const geminiUrl = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent";

  // Fetch catalog only when in-scope (business/product)
  const catalogPromise = api.get("/EcomProducts", {
    fields: "Name,Product_image_url,Price,Inventory_Count,Product_image",
    per_page: 50
  })
  .then(CRMresponse => {
    const rows = Array.isArray(CRMresponse && CRMresponse.data) ? CRMresponse.data : [];
    const cat = rows.map(r => ({
      name: r.Name,
      price: r.Price,
      sku: r.id,
      stock: r.Inventory_Count,
      image: r.Product_image_url || r.Product_image
    }));
    if (budget) {
      return cat.filter(p => {
        const val = Number(p.price);
        if (Number.isNaN(val)) return false;
        if (budget.min != null && val < budget.min) return false;
        if (budget.max != null && val > budget.max) return false;
        return true;
      });
    }
    return cat;
  })
  .catch(e => { logger.error("CRM fetch error: " + e.message); return []; });

  return catalogPromise
    .then(catalog => {
      const useCatalog = productIntent && catalog.length > 0;

      // System text—STRICT: answer only about Hiltech; refuse other topics politely
      const sys_en = [
        "You are Hiltech’s assistant. Answer ONLY questions about Hiltech’s products and services.",
        "If the user asks about anything unrelated (e.g., world news, general AI), politely decline and redirect to Hiltech topics.",
        useCatalog
          ? [
              `You are given a JSON 'catalog' with: name, price (${currency}), sku, stock, image.`,
              "Use ONLY this catalog for product details. If budget terms appear (under/between/etc.), filter accordingly.",
              `Format list items as: '• Name — ${currency}Price (SKU)'. Show at most 8 items.`,
              "If no matches, say so briefly."
            ].join("\n")
          : "If no product context is needed, answer concisely about Hiltech."
      ].join("\n");

      const sys_he = [
        "אתה העוזר של Hiltech. ענה אך ורק על שאלות שקשורות למוצרי ושירותי Hiltech.",
        "אם השאלה אינה קשורה (כגון חדשות עולם או AI כללי) — סרב בנימוס והפנה לנושאי Hiltech.",
        useCatalog
          ? [
              `מצורף JSON 'catalog' עם: name, price (${currency}), sku, stock, image.`,
              "השתמש רק בקטלוג למידע על מוצרים. אם מופיע תקציב (עד/בין וכו'), סנן בהתאם.",
              `פורמט רשימה: '• שם — ${currency}מחיר (SKU)'. הצג עד 8 פריטים.`,
              "אם אין התאמות — אמור זאת בקצרה."
            ].join("\n")
          : "אם אין צורך בהקשר מוצרי, ענה בקצרה על Hiltech."
      ].join("\n");

      const systemText = he ? sys_he : sys_en;

      const parts = [
        { text: systemText },
        { text: (he ? "פניית המשתמש:\n" : "User query:\n") + userInput }
      ];
      if (useCatalog) parts.push({ text: (he ? "קטלוג:\n" : "Catalog JSON:\n") + JSON.stringify(catalog) });

      const body = {
        contents: [{ parts }],
        generationConfig: { temperature: 0.3, maxOutputTokens: 600 }
      };

      return fetchJSON(geminiUrl, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "x-goog-api-key": process.env.GEMINI_API_KEY
        },
        body: JSON.stringify(body)
      });
    })
    .then(response => {
      const candidates = response && response.candidates;
      if (Array.isArray(candidates) && candidates.length > 0) {
        const parts = candidates[0]?.content?.parts || [];
        if (parts.length > 0 && parts[0].text) return { message: parts[0].text };
      }
      return { message: he ? "לא נמצאה תשובה מתאימה." : "No valid response content." };
    })
    .catch(err => {
      logger.error("Fallback error: " + err.message);
      return { message: he ? "לא הצלחתי לעבד כרגע. נסה שוב." : "I couldn’t process that right now. Please try again." };
    });
}
