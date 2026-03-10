# Swiggy MCP Teardown — PM Observations

> Swiggy has released an official MCP server for food delivery. This is a teardown based on actually using the MCP — not just reading the docs.

---

## Setup Flow

1. Download Claude Desktop
2. Go to Settings → Developer → Edit Config
3. Add the following config:

```json
{
  "mcpServers": {
    "swiggy-food": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.swiggy.com/food"
      ]
    }
  }
}
```

4. Kill Claude Desktop and restart
5. On restart, a Swiggy login popup appears — authenticate with your Swiggy account
6. Done — you can now use Swiggy via Claude

> Note: This took significant trial and error to set up. Documenting here to save other PMs time.

---

## What Swiggy MCP Can Do — Tools Exposed

| Tool | What it does |
|---|---|
| get_addresses | Fetches user's saved delivery addresses |
| search_restaurants | Search by cuisine, location, name |
| search_menu | Search dishes within a restaurant |
| get_restaurant_menu | Full menu of a restaurant |
| add_to_cart / update_food_cart | Add or modify items in cart |
| get_food_cart | View current cart |
| flush_food_cart | Clear cart entirely |
| place_food_order | Confirm and place the food order |
| get_food_orders | Get active order status |
| fetch_food_coupons | Get available coupons and offers |
| apply_food_coupon | Apply a coupon to the order |
| track_food_order | Track delivery status |
| get_food_order_details | Detailed info on a specific order |

---

## What Works Well

**1. Clean tool separation**
Each tool does one thing. Search → Menu → Cart → Order → Track is a logical, well-scoped flow that maps closely to how users think about ordering food.

**2. Auth handled via popup**
Swiggy login happens via a native popup on Claude Desktop restart. Smooth, no manual token handling needed.

**3. End-to-end ordering flow**
From restaurant discovery to order placement to tracking — the full flow is covered, including coupons.

**4. Images are available in the API response**
Restaurant search results return `imageUrl` fields with real Swiggy CDN links. Images are available via MCP — however Claude Desktop did not render these images in the conversation. The image URLs are returned as plain text. Rendering rich media from MCP responses is a client-side gap, not a Swiggy MCP gap.

**5. Offers come through**
Discount badges like "₹100 OFF ABOVE ₹449" and "60% OFF" are returned in restaurant results. Plus there are dedicated coupon tools (`fetch_food_coupons`, `apply_food_coupon`) for full offer management.

**6. Basic order tracking exists**
`get_food_orders` returns active order status. `track_food_order` also exists for tracking.

---

## What is Missing

**1. No Payments Layer — The Biggest Gap**
Swiggy MCP stops at order placement. There is no tool for payment initiation, payment method selection, or payment confirmation.

Why this is intentional:
- Payment aggregators (Razorpay) are RBI regulated — exposing payment initiation through a third-party AI layer requires explicit compliance clearance
- Saved card data cannot be shown to third-party apps (PCI-DSS compliance)
- UPI requires device binding + MPIN — cannot be delegated to an AI agent today
- Liability in AI-initiated payment flows is legally unsettled

**Implication:** Users must complete payment manually outside the MCP flow. The conversational experience breaks at the most critical step — checkout.

**2. No Personalisation**
Search is location-aware (uses address ID) but not personalised. No order history, no time-of-day signals, no "your usual" recommendations. The Swiggy app's ML-driven recommendations are absent from the MCP layer.

**3. No Editorial / Curated Layer**
No chef specials, trending dishes, or "people also ordered." Raw menu data is returned without the curation layer that makes the Swiggy app experience rich.

**4. No Real-Time Rider Tracking**
`track_food_order` exists but likely returns status updates only. Real-time rider location on a map is unavailable via MCP.

---

## Summary Assessment

Swiggy MCP is a strong first version — the ordering flow from search to placement to tracking is clean and functional, and the inclusion of coupon tools and image URLs shows attention to a complete experience.

However it remains a **discovery, cart, and order management tool**. The missing payments layer means every MCP-initiated order still requires a manual handoff at checkout — breaking the conversational experience at its most important moment.

Until payments is solved — either via a mandate/pre-authorisation layer or a compliant delegated auth flow — conversational commerce via MCP will always be incomplete.

---

*Teardown by Aman Jain | March 2026*
*Based on actual usage, not just documentation*
