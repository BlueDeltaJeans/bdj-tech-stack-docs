# Domain Context Primer

> Confidential — shared for proposal evaluation. This primer gives an outside team the business and product vocabulary needed to read the rest of this bundle. It is a plain-language orientation, not a spec; deeper technical detail lives in the documents linked at the end.

## What Blue Delta makes

Blue Delta Jeans is a bespoke, custom-fit apparel maker based in Tupelo, Mississippi. The flagship product is **custom-fit denim jeans**, but the catalog spans several made-to-order lines:

- **Pants** — Raw Denim jeans, Cotton Chino pants (internally coded "Jones Heritage"), and Performance-fabric pants.
- **Shorts** — Performance-fabric, men's only.
- **Kentucky Derby pants** — seasonal/event styles for the Derby and the Oaks.
- **Custom leather belts** — built to a chosen leather, width, and hardware.
- **Shoes** — a custom shoe line sold only at in-person events (no online shoe products yet).
- **Video gift cards** — a gift product, not a garment.

Every garment is built to order for an individual customer. There is no off-the-rack inventory in the usual sense: a customer chooses the product, the fabric/leather, the cut/style, and the finishing details, and the item is then cut and sewn to that person's measurements.

## The fitting / measurement concept

Blue Delta's core promise is bespoke fit without requiring an in-person tailor. Two paths exist:

- **In person** — at events or the workshop, a customer can be measured directly (or fill out a paper order form that staff enter into the system).
- **Online** — the customer never picks up a tape measure. Instead they answer a short questionnaire and an AI model predicts their full set of body measurements.

The online measurement experience is branded the **Virtual Tailor**. It is powered by a third-party AI body-data vendor, **Bold Metrics** (a San Francisco company). From a short questionnaire — roughly 9 questions for men, 11 for women — Bold Metrics' models predict 50+ body measurements in under a minute, with no selfies, photos, or body scanner required. This is what lets Blue Delta offer made-to-measure fit to online customers anywhere. Adopting it materially reduced the return rate (to roughly 7%).

Those predicted measurements feed Blue Delta's internal measurement/spec engine (referred to in this bundle by its internal nickname; see the Virtual Tailor and reference docs), which turns body measurements into the garment specifications the floor sews to.

## Key product and garment terms

These are the building blocks of a customized order. Understanding them makes the SKU and pipeline docs readable.

- **Fabric** — the cloth a pant is made from. Families include Raw Denim (code `RW`), Cotton Chino / Jones Heritage (`CC` / `JH`), and Performance (`PF`). Each family has many named colors (e.g. "Dark Indigo," "Charcoal"). "Jones Heritage" is the internal name for the Cotton Chino family; `CC`-prefixed codes are the Cashiers Collection colors.
- **Leather** — the belt equivalent of fabric (e.g. Dark Brown, Black, Navy, Football-textured). Each leather has a `CB` code.
- **Style** (or cut) — the silhouette of a pant. Men's styles include Straight, Boot, Skinny, and **Fashion Boot** (the "FASHBOOT" cut). Women's styles include Straight, Boot, Flare, and Skinny.
- **Thread / stitch color** — the color of the stitching on the garment. Can be a named color or **Tonal** (thread that matches the fabric). On belts, a "no thread" option means no decorative stitching.
- **Monogram** — optional personalized lettering (up to a few characters) added to a garment, with its own thread color. On pants this is an online-only paid add-on; on belts it is offered in both channels.
- **Hardware** — the metal fittings on a belt: the buckle finish (e.g. Brass, Nickel, Flat Black).
- **Width** — belt width (e.g. 1", 1.25", 1.5").
- **Size** — for shoes, the footwear size; for garments, fit is driven by the measurement spec rather than a single size label.

## Online vs in-store (POS): why it matters

A single fully-personalized pant has far more option combinations than a storefront product can natively hold, so Blue Delta maintains **two parallel sets of products** — one for the online store and one for the in-person point-of-sale (POS) channel. The two channels capture customization differently (online stores some choices as line-item add-on fields; POS encodes them into the product code/SKU). As a result, the **shape of an order's SKU tells you which channel it came from**, which is central to how the downstream order pipeline parses each order. This split is the single most important structural concept in the rest of this bundle.

## Where to go deeper

This primer intentionally stays high-level. For the full detail an implementation team needs:

- **Online vs POS product architecture, customization capture, and channel detection** — see [04-website-vs-pos-products.md](../../04-website-vs-pos-products.md).
- **The complete SKU catalog (formats, fabric/leather/thread codes, counts, and known data anomalies)** — see [SKU Reference Guide](../zapier/SKU%20Reference%20Guide.md).
- **The Virtual Tailor / Bold Metrics measurement flow** — see the Virtual Tailor document at the bundle root.

> No customer names, contact information, addresses, or body measurements appear in this primer or anywhere in this bundle; any example values are synthetic.
