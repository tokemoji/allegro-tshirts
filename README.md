# Allegro T-shirts — Print-on-Demand Project

> Mass-generate unique T-shirt designs via Nano Banana (Gemini image gen), list on Allegro (Polish marketplace), fulfill via BaseLinker + InPost. Polish sole proprietorship (JDG) on VAT.

**Audience.** AI agents working on listing automation, design generation, and fulfillment workflows. NOT a marketing document.

---

## 1. Brief

### 1.1 Goal
Generate **hundreds to thousands** of unique T-shirt designs with Nano Banana and sell them on Allegro at scale. Target market: Polish casual gamers, meme-lovers, gift shoppers.

### 1.2 Business profile
- **Owner:** Piotrek, JDG on VAT (NIP registered)
- **Invoicing:** Fakturownia (separate invoice series for T-shirts: `TS/2026/...` vs existing `VR/2026/...` for VirtualCity)
- **Marketplace:** Allegro (Polish, dominant e-commerce)
- **Shipping:** BaseLinker + InPost paczkomaty, plus couriers
- **Volume target:** 500-5000 SKUs in first 12 months

---

## 2. Architecture overview

```
┌────────────────────────────────────────────────────────────┐
│                   Design generation (offline)              │
│  Nano Banana (Gemini image) → per DESIGN_ID asset pack     │
│   • print.png (transparent, exact print spec)              │
│   • mockup_flat.png (design on blank shirt)                │
│   • mockup_model.png (lifestyle shot)                      │
│   • meta.json (title/desc/tags/variants/base_price)        │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│              Allegro listing (API, mass create)            │
│   DESIGN_ID → stable SKU / product code / BL field         │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                  Order arrives (Allegro webhook)           │
│   No AI generation. Just link pre-generated assets.        │
│   Create lightweight production card.                      │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│   Fulfillment (BaseLinker)                                 │
│   • Shipping label (InPost paczkomaty)                     │
│   • Courier selection (Allegro Delivery / InPost)          │
│   • Label printer integration                             │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│   Internal ops UI (custom web page for workers)            │
│   • Print queue (status: printed/packed)                   │
│   • Support inbox (Gmail + Allegro unified)                │
│   • Agent drafts reply → human edits + approves → send     │
└────────────────────────────────────────────────────────────┘
```

---

## 3. Pre-generation decision (CRITICAL)

**Generate asset packs UPFRONT, not at order time.**

Why:
- Speed: orders link instantly, no waiting
- Consistency: design is locked, no per-order variation
- Cost: AI generation is expensive per call; bulk is cheaper
- Reliability: no API failures blocking orders

**Asset pack per DESIGN_ID:**
```
DESIGN_<id>/
├── print.png          # transparent, exact print spec (DTF/DTG TBD)
├── mockup_light.png   # design on light shirt (white/cream)
├── mockup_dark.png    # design on dark shirt (black/navy)
├── mockup_model.png   # lifestyle, person wearing shirt
├── meta.json          # {title, description, tags, variants, base_price}
└── thumbs/
    ├── 200x200.png    # Allegro search results
    └── 800x800.png    # Allegro offer gallery
```

Open question: **DTF vs DTG** (decide once, encode in print spec):
- DTF: 3000×3600 px, sRGB, transparent PNG, 300 DPI
- DTG: 3600×4800 px, sRGB, transparent PNG, 300 DPI
- Spec TBD → once decided, write to `templates/allegro/print-spec.md`

---

## 4. Style guidelines

For Nano Banana prompts generating shirt designs:

- **Centered composition** (works for all shirt fits: classic, slim, oversize)
- **Bold, simple shapes** (avoid fine detail that gets lost in fabric)
- **High contrast** (prints well on both white and black shirts)
- **Avoid gradients in core artwork** (only OK for background flourishes)
- **Text on shirt:** max 3 words, large sans-serif
- **Color variants:** generate 3 per design (light shirt + dark print, dark shirt + light print, neutral)

For full style + technical rules, see `nano-banana-workflows/templates/allegro/` (coming soon).

---

## 5. Allegro listing automation

### 5.1 Listing fields (Allegro API)
- `name` — from meta.json
- `description` — HTML, includes mockups
- `category` — main + leaf (T-shirts, Męskie/Damskie/Dziecięce, etc.)
- `parameters` — size, color, material
- `price` — from meta.json base + variant deltas
- `stock` — from BaseLinker warehouse
- `delivery` — InPost paczkomaty (free above 50 PLN), Allegro Delivery Smart!

### 5.2 DESIGN_ID mapping
Stable identifier per design, stored in:
- Allegro `sellerSku` field
- BaseLinker `product_id` field
- Local DB primary key

This way the same design can be updated, restocked, or removed across all 3 systems via one ID.

### 5.3 Mass listing script (planned)
```python
# scripts/list_on_allegro.py
import json
from pathlib import Path
from allegro_api import AllegroAPI  # wrapper

DESIGNS_DIR = Path("./designs")
api = AllegroAPI(token=os.environ["ALLEGRO_API_TOKEN"])

for design_dir in DESIGNS_DIR.iterdir():
    meta = json.loads((design_dir / "meta.json").read_text())
    images = [f for f in design_dir.glob("*.png") if "thumb" not in f.name]
    
    offer_id = api.create_offer(
        name=meta["title"],
        description=render_description(meta, images),
        category=meta["allegro_category"],
        parameters=meta["parameters"],
        price=meta["base_price"],
        stock=meta["initial_stock"],
        seller_sku=f"DESIGN_{design_dir.name}",
    )
    api.upload_images(offer_id, images)
    print(f"Listed {design_dir.name} as {offer_id}")
```

---

## 6. Fulfillment (BaseLinker)

### 6.1 Order sync
- Allegro order → BaseLinker (auto via integration)
- BaseLinker → print queue (webhook or polling)

### 6.2 Shipping label
- InPost paczkomaty: select locker, generate label, print
- Courier (Allegro Delivery): rate-shop, generate label
- Label printer: small thermal printer in worker's area

### 6.3 Status sync
- Mark as "packed" → BaseLinker notifies Allegro
- Generate tracking number → updates buyer

---

## 7. Internal ops UI (planned)

A simple internal web page with two main panels:

### 7.1 Print queue
- List of orders waiting to be printed
- For each: DESIGN_ID, size, color, design preview, ship-to
- Click "Mark printed" → status updates in BaseLinker
- Click "Mark packed" → triggers label print + status sync

### 7.2 Support inbox
- Unified feed: Gmail + Allegro messages
- Agent generates reply draft (context: order, customer history)
- Worker can edit + approve (or reject + re-draft)
- On approve: send via Gmail API or Allegro API

Tech stack: simple HTML + a small Node backend, served from the same workspace as VirtualCity.

---

## 8. Open items

| Item | Status | Action |
|---|---|---|
| Print spec (DTF vs DTG, px dimensions) | TBD | Decide, encode in `templates/allegro/print-spec.md` |
| Allegro API access | Need to register app | Get client_id + client_secret |
| BaseLinker API access | Need to configure | Get token, set up Allegro integration |
| Fakturownia separate series | Need to create | TS/2026/... in Fakturownia settings |
| Design pipeline (Nano Banana batch) | Not started | Write `scripts/generate_designs.py` |
| Listing pipeline | Not started | Write `scripts/list_on_allegro.py` |
| Ops UI | Not started | Spec + implementation |
| Mockups (model lifestyle shots) | TBD | Nano Banana or template-based |

---

## 9. Related

- `nano-banana-workflows` — image generation rules (Allegro section §2.3)
- VirtualCity (`virtualcity-pl` repo) — base for ops UI tech stack
- Memory: `memory/2026-03-26.md` — original planning session
- Fakturownia: separate configuration needed (not in repo scope)

## 10. Brand spec at-a-glance

| | |
|---|---|
| **Marketplace** | Allegro (Polish) |
| **Owner VAT** | JDG, Piotrek |
| **Invoice series** | TS/2026/... (separate from VR) |
| **Shipping** | InPost paczkomaty, Allegro Delivery |
| **Order mgmt** | BaseLinker |
| **Design gen** | Nano Banana (Gemini image) |
| **Volume target** | 500-5000 SKUs in 12 months |
| **Critical rule** | Pre-generate assets. NO AI at order time. |