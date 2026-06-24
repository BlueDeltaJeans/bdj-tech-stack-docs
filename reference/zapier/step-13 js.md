# step-13.js

```jsx
let CartToken = inputData.CartToken;
let DateCreated = inputData.DateCreated.split("T")[0];

let Source = inputData.Source || "";

// Parse Line Items (LI)
let LI;
try {
  LI = JSON.parse(inputData.LI);
} catch (error) {
  throw new Error("Invalid JSON input for LI");
}

// Extract SKU parts
let SKU = LI.sku.split("-"); // "CB6_RYDER-1.25-FLAT_BLACK-SILVER".split("-");

// Initialize Properties object
let Properties = LI.properties.reduce((acc, prop) => {
  acc[prop.name] = prop.value;
  return acc;
}, {});

// Function to set DueDate based on DueDays
const setDueDate = (days) => {
  let dueDate = new Date(DateCreated);
  dueDate.setDate(dueDate.getDate() + days);
  return dueDate.toISOString().split("T")[0];
};

// SKU: Pants - Women
if (LI.sku.startsWith("W-")) {
  Object.assign(Properties, {
    ProductType: "Pant",
    Gender: "F",
    Fabric: SKU[1],
    Style: SKU[2],
    Thread: SKU[3],
    DueDays: CartToken ? 14 : 30,
    DueDate: setDueDate(CartToken ? 14 : 30),
  });
}

// SKU: Pants - Men
if (LI.sku.startsWith("M-")) {
  Object.assign(Properties, {
    ProductType: "Pant",
    Gender: SKU[0],
    Fabric: SKU[1],
    Style: SKU[2],
    Thread: SKU[3],
    DueDays: CartToken ? 14 : 30,
    DueDate: setDueDate(CartToken ? 14 : 30),
  });
}

// SKU: Shoes
if (LI.sku.includes("SHOE-")) {
  Properties.ProductType = "Shoe";
  Properties.Quantity = LI.quantity;
  Properties.Shoe = SKU[1];
  if (LI.variant_title?.includes("/")) {
    let [gender, size] = LI.variant_title.split("/").map(item => item.trim());
    Properties.Gender = gender || "";
    Properties.Size = size || "";
  } else {
    Properties.Gender = "";
    Properties.Size = "";
  }
  Properties.DueDays = 45;
  Properties.DueDate = setDueDate(45);
}

// SKU: Gift - Video
if (LI.sku.startsWith("VID-GIFT-")) {
  Object.assign(Properties, {
    ProductType: "Video Card",
    Message: SKU[2],
    Denomination: SKU[3],
    Quantity: LI.quantity,
    DueDays: 5,
    DueDate: setDueDate(5),
  });
}

// SKU: Belts
if (LI.sku.startsWith("CB")) {
  Object.assign(Properties, {
    ProductType: "Belt",
    LeatherCode: SKU[0].replace("_RYDER", ""),
    Width: SKU[1],
    Hardware: SKU[2],
    Monogram:
      Properties.Monogram ??
      Properties.monogram ??
      Properties.Mono ??
      Properties.mono ??
      "NONE",
    DueDays: 30,
    DueDate: setDueDate(30),
  });

  const isRyder = LI.sku.includes("_RYDER-");
  const isPOS = Source === "pos";

  // Thread color logic
  // Ryder and POS belts store thread color as the last SKU segment
  if (isRyder || isPOS) {
    Properties["Thread Color"] = SKU[SKU.length - 1];
  }

  // Ryder-only lettering logic
  if (isRyder) {
    Properties.Lettering =
      Properties.Monogram ??
      Properties.monogram ??
      Properties.Mono ??
      Properties.mono ??
      "";
  }

  // Belt Leather Legend
  const LeatherNames = {
    CB1: "Dark Brown Leather",
    CB2: "Mid Brown Leather",
    CB3: "Light Brown Leather",
    CB4: "Navy Leather",
    CB5: "Football Leather",
    CB6: "Black Leather",
    CB7: "Natural Derby Leather",
    CB8: "English Tan Leather",
    CB9: "Red Leather",
    CB10: "Light Brown Leather",
    CB11: "Gray",
  };

  Properties.LeatherName = LeatherNames[Properties.LeatherCode] || "";
}

// Process Tags
let Tags = inputData.Tags ? inputData.Tags.split(",").map(tag => tag.trim()) : [];
let TagsProduct = ["Cashiers", "Denim", "Vintage", "Performance", "Chino", "Jhino"];
let Alteration = Tags.includes("alteration");

let EventTags = Tags.filter(tag => !TagsProduct.includes(tag));
Tags = Tags.filter(tag => TagsProduct.includes(tag));

output = [{ Source, Alteration, Tags, EventTags, CartToken, DateCreated, Properties, SKU, LI }];

```