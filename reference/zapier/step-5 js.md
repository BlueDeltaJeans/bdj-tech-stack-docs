# step-5.js

```jsx
// Input variable: TAGS
let TAGS = inputData.TAGS || "";

// Uppercase for consistent matching
TAGS = TAGS.toString().toUpperCase();

// Default
let RUSH = "";

// Check for specific matches
if (TAGS.includes("RUSH ROYAL")) {
  RUSH = " | RUSH ROYAL";
} else if (TAGS.includes("RUSH COBALT")) {
  RUSH = " | RUSH COBALT";
}

let RAW;
try {
  RAW = JSON.parse(inputData.RAW);
} catch (error) {
  throw new Error("Invalid JSON in inputData.RAW");
}

// Ensure RAW has an order object
if (!RAW.order) throw new Error("Invalid data structure: Missing 'order' object");

let Order = RAW.order; // Extract order object

// Determine Order Type
let OrderType = Order.cart_token ? "Online" : "Direct";

// Customer Details
let CustomerOrdersCount = Order.customer?.orders_count || 0;
let CustomerNew = CustomerOrdersCount > 1 ? "No" : "Yes";

// Extract and Filter Line Items
let LI = (Order.line_items || []).filter(item =>
  item.sku?.startsWith("W-") ||
  item.sku?.startsWith("M-") ||
  item.sku?.startsWith("CB") ||
  item.sku?.startsWith("VID-GIFT-") ||
  item.sku?.startsWith("SHOE-")
).map(item => JSON.stringify(item));

let LIcount = LI.length;

// Function to Clean Up Note Attributes
function cleanNoteAttributes(NA) {
  if (!Array.isArray(NA)) return [];
  
  return NA.map(attr => ({
    name: attr.name,
    value: attr.name === "__aftersell_tamper_proof" ? "" : attr.value
  }));
}

// Extract and Clean Note Attributes
let NA = cleanNoteAttributes(Order.note_attributes);
let NAcount = NA.length;

// Convert Note Attributes to Extras Object
let Extras = {};
NA.forEach(attr => {
  if (attr.name) Extras[attr.name] = attr.value;
});

// BDJ User Data Extraction
let BDJUserData = {};
if (typeof Extras["BDJ User Data"] === "string") {
  Extras["BDJ User Data"].split("\n").forEach(line => {
    let [key, ...valueParts] = line.split(": ");
    let value = valueParts.join(": "); // Handles cases where value contains `:`
    if (key && value) BDJUserData[key.trim()] = value.trim();
  });
  delete Extras["BDJ User Data"];
}
let BDJUserDataCount = Object.keys(BDJUserData).length;

// Extract Order Note
let Note = Order.note || "";

// Final Output
output = [{
  RUSH,
  NAcount,
  LIcount,
  CustomerOrdersCount,
  BDJUserDataCount,
  CustomerNew,
  OrderType,
  Extras,
  BDJUserData,
  Note,
  LI,
  NA
}];

```