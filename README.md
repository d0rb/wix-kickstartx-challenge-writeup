# Wix KickstartX Challenge 2026 — Full Writeup

**Result: 200/200 (100%) — Token: WKX-9245FA70-200**
**Date: March 24, 2026**

---

## 1. Discovery & Reconnaissance

### The Challenge Announcement

A LinkedIn post announced a special challenge for Wix KickstartX — a junior developer program.
The rules:
- 30 minutes, one attempt only
- Top 10 highest scores advance directly to the exam stage
- Two links provided: `wix-kickstartx-challenge-2026.base44.app/` and `wixkickstart.com`

### Finding the Challenge Entry Point

The main site `wixkickstart.com` is a Wix-hosted site (server-side rendered by Wix's
Thunderbolt engine). Standard `curl` only returns the JavaScript shell — no actual content.
We used **headless Chromium** to render it:

```bash
chromium --headless --disable-gpu --no-sandbox --virtual-time-budget=10000 \
  --dump-dom "https://wixkickstart.com" > /tmp/wix_rendered.html
```

From the rendered DOM we extracted an embedded iframe:

```html
<iframe src="https://kotrynarac.github.io/KickstartX-interactions/" ...>
```

This turned out to be a p5.js particle animation (hero section eye candy), not the challenge
itself. The actual challenge lives at the Base44 app.

---

## 2. Reverse Engineering the Challenge App

### Platform: Base44

The challenge runs on [Base44](https://base44.app) — a low-code app platform. The app is a
single-page React application with all logic in one JS bundle:

```
https://wix-kickstartx-challenge-2026.base44.app/assets/index-oTG160r9.js
```

**Size:** 410,809 bytes (minified React + app logic + Base44 SDK)

### Extracting the API Surface

By grepping the JS bundle, we mapped the entire API:

| Function | Purpose |
|---|---|
| `startGame` | Creates a game session, returns 200 images + 200 descriptions |
| `scoreGame` | Accepts `{matches: {}, sessionId: ""}`, returns `{correctCount: N}` |
| `generateToken` | Takes sessionId, returns completion token |
| `getLeaderboard` | Returns top scores |
| `saveNickname` | Saves display name for leaderboard |

**Entities:** `GameSession`, `Participant`

**API URL pattern:**
```
POST /api/apps/{appId}/functions/{functionName}
```

**App ID:** `69aea07cbcb9a3dd1039a58d`

### Key Constants from the Bundle

```javascript
const or = 1800;   // Time limit: 1800 seconds (30 minutes)
const rm = 200;    // Total items: 200 image-description pairs
```

### JSON Submission Discovery

A critical finding — the app has a **JSON bulk submission** mode. From the minified source:

```javascript
function Ik({onSubmit:r, onClose:n}) {
    // ...
    h = JSON.parse(s)  // Parse JSON input
    // Validation: must be object like { "IMG-001": "DESC-042", ... }
    r(h)  // Submit all matches at once
}
```

**Placeholder text in the modal:**
```json
{
  "IMG-001": "DESC-042",
  "IMG-002": "DESC-017",
  ...
}
```

This means we don't need to click 200 times in the UI — we can submit a JSON mapping
of all 200 matches programmatically.

### Authentication Flow

The Base44 SDK uses JWT authentication:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The JWT payload contains:
```json
{
  "sub": "dor.bismut@gmail.com",
  "exp": 1782162460,
  "iat": 1774386460
}
```

Required headers for all API calls:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
X-App-Id: 69aea07cbcb9a3dd1039a58d
Base44-Functions-Version: prod
X-Origin-URL: https://wix-kickstartx-challenge-2026.base44.app/
```

---

## 3. Understanding the Game Data

### startGame Response

```
POST /api/apps/69aea07cbcb9a3dd1039a58d/functions/startGame
Status: 200
Transferred: 62.87 kB compressed (2.20 MB decompressed)
Content-Encoding: br (Brotli)
```

Response structure:
```json
{
  "status": "active",
  "sessionId": "69c2fd218e5b26f307c941c9",
  "startedAt": "2026-03-24T21:07:45.101Z",
  "imageOrder": ["IMG-154", "IMG-102", ...],   // 200 items (display order)
  "descOrder": ["DESC-125", "DESC-109", ...],  // 200 items (display order)
  "imagesData": { "IMG-001": "data:image/svg+xml;base64,...", ... },  // 200 SVGs
  "descriptionsData": { "DESC-105": "milky field, overlaid with...", ... }  // 200 texts
}
```

### Image Format: Inline SVG (base64-encoded)

Each image is a 200x200 SVG containing:
1. A **background gradient** (light or dark themed)
2. An optional **overlay pattern** (lines, dots, rings, etc.)
3. **Geometric shapes** with specific colors, sizes, opacity, rotation, and position

Example decoded SVG (IMG-001):
```xml
<svg xmlns="http://www.w3.org/2000/svg" width="200" height="200" viewBox="0 0 200 200">
  <defs>
    <linearGradient id="bg" ...>
      <stop offset="0%" stop-color="#f5f0eb"/>
      <stop offset="100%" stop-color="#e8e0d5"/>
    </linearGradient>
  </defs>
  <rect width="200" height="200" fill="url(#bg)"/>
  <!-- Optional overlay lines/dots/rings here -->
  <polygon points="..." fill="#E2725B" transform="translate(128.6,185.7) rotate(90)"
           style="opacity:0.5;filter:drop-shadow(...)"/>
  <!-- More shapes... -->
</svg>
```

### Description Format: Structured Natural Language

Each description follows a strict pattern:
```
{background} field[, overlaid with {overlay}]. {N} elements total:
{size} {opacity} {color} {shape} ({rotation}, at {position}) ·
{size} {opacity} {color} {shape} ({rotation}, at {position}) · ...
```

Example:
```
milky field, overlaid with tilted cross-lines. 10 elements total:
minuscule nearly solid dim gray pike (steeply angled, at center inner-left) ·
substantial solid azure pike (diagonal, at top center) · ...
```

---

## 4. Building the Automated Solver

### Step 1: Map All Constants

We needed to map SVG properties to description vocabulary. This required analyzing all 200
images and all 200 descriptions to find exact correspondences.

#### Background Mapping (12 types)

By counting occurrences on both sides, we established a 1:1 mapping:

| SVG Gradient Start Color | Count | Description Word | Count |
|---|---|---|---|
| `#0a1628` | 28 | `pitch` | 28 |
| `#eef2f7` | 20 | `frosted` | 20 |
| `#0d0d0d` | 19 | `tenebrous` | 19 |
| `#fef9f0` | 18 | `pearlescent` | 18 |
| `#1e0a2e` | 18 | `nocturnal` | 18 |
| `#0a1a0a` | 17 | `midnight` | 17 |
| `#1a0a0a` | 15 | `inky` | 15 |
| `#f5f0eb` | 14 | `milky` | 14 |
| `#f7f3ee` | 14 | `ethereal` | 14 |
| `#f0f0f0` | 14 | `radiant` | 14 |
| `#f0f7f4` | 13 | `glowing` | 13 |
| `#1a1a2e` | 10 | `somber` | 10 |

**Method:** Count unique gradient start colors in all 200 SVGs, count unique first-word
in all 200 descriptions, match by count.

#### Overlay Mapping (7 types)

SVG overlays are implemented differently depending on type:

| SVG Pattern | Detection Method | Count | Description Name | Count |
|---|---|---|---|---|
| `<line>` elements, horizontal (`dy=0`) | Check x1,y1,x2,y2 | 35 | `striped overlay` | 35 |
| `<line>` elements, vertical (`dx=0`) | Check x1,y1,x2,y2 | 34 | `lattice pattern` | 34 |
| `<circle>` with `opacity="0.06"` (dots) | Count low-opacity circles | 30 | `stippled layer` | 30 |
| No overlay elements at all | No lines, no bg shapes | 28 | (none) | 28 |
| `<line>` elements, diagonal | Check slope direction | 27 | `tilted cross-lines` | 27 |
| Small `<polygon>`/`<path>` with `opacity="0.06"` | Low-opacity attr shapes | 25 | `arrow-band texture` | 25 |
| `<circle>` with `fill="none" stroke="#888"` | Stroke-only circles | 21 | `ringed pattern` | 21 |

**Key insight:** Background pattern elements use `opacity` as an XML **attribute**
(e.g., `opacity="0.06"`), while foreground shapes use `opacity` inside the **style**
attribute (e.g., `style="opacity:0.7"`). This distinction was critical for separating
background patterns from actual shapes.

#### Color Mapping (28 colors)

All 200 SVGs use exactly 28 unique hex fill colors. All 200 descriptions use exactly 28
unique color names.

| Hex | Description Name | | Hex | Description Name |
|---|---|---|---|---|
| `#708090` | blue-gray | | `#cd7f32` | brass |
| `#f5f5f5` | near white | | `#dc143c` | fiery red |
| `#00bcd4` | electric cyan | | `#2196f3` | azure |
| `#ff6b6b` | salmon pink | | `#228b22` | rich green |
| `#98ff98` | pale green | | `#b0b0b0` | platinum |
| `#0f52ba` | deep sapphire | | `#800020` | oxblood |
| `#ff8c00` | deep orange | | `#4b0082` | dark purple |
| `#e2725b` | terra rosa | | `#ffbf00` | marigold |
| `#b7410e` | russet | | `#ff69b4` | candy pink |
| `#ffd700` | bright gold | | `#c0c0c0` | tin |
| `#6b8e23` | moss | | `#008080` | deep teal |
| `#40e0d0` | pale teal | | `#0047ab` | royal blue |
| `#e34234` | burnt sienna | | `#a0522d` | deep red |
| `#4a4a4a` | dim gray | | `#36454f` | dark gray |

#### Shape Type Mapping (10 types)

| SVG Element | Detection Logic | Description Name |
|---|---|---|
| `<polygon>` 3 vertices | Count space-separated point pairs | `pike` |
| `<polygon>` 4 vertices | | `tilted square` |
| `<polygon>` 5 vertices | | `quint form` |
| `<polygon>` 6 vertices | | `bee cell` |
| `<polygon>` 10 vertices | | `asterisk` |
| `<rect>` (non-background) | Has width/height, not 200x200 | `tilted square` |
| `<circle>` | Tag name | `disc` |
| `<path>` with `fill-rule="evenodd"` | Two concentric arc paths | `donut` |
| `<path>` with single arc + Z | Half-circle path | `half-disc` |
| `<path>` with 6+ L commands | Cross/plus shape | `crosshair` |
| `<path>` with 3-5 L commands | Arrow-like shape | `pointer` |

#### Size Mapping

Based on the maximum radius of shape vertices from origin:

| Radius Range | Description Name |
|---|---|
| 0–12 | `minuscule` |
| 13–16 | `modest` |
| 17–20 | `mid-sized` |
| 21–25 | `substantial` |
| 26+ | `massive` |

#### Opacity Mapping

| SVG opacity value | Description Name |
|---|---|
| 1.0 | `solid` |
| 0.85 | `nearly solid` |
| 0.7 | `semi-transparent` |
| 0.5 | `faint` |

#### Rotation Mapping

| SVG rotate() value | Description Name |
|---|---|
| 0° | `upright` |
| 1–20° | `slightly tilted` |
| 21–55° | `diagonal` |
| 56–75° | `steeply angled` |
| 76–105° | `sideways` |

#### Position Grid

Shapes are placed on a 7x7 grid at coordinates:
`[14.3, 42.9, 71.4, 100.0, 128.6, 157.1, 185.7]` for both X and Y.

These map to column names: `far-left, left, inner-left, center, inner-right, right, far-right`
And row names: `top, upper, upper-mid, center, lower-mid, lower, bottom`

### Step 2: Parse All SVGs

For each of the 200 SVG images:

1. **Decode** base64 to raw SVG XML
2. **Extract background** — read first `<stop>` color from `<linearGradient id="bg">`
3. **Classify overlay** — check for `<line>`, `<circle fill="none" stroke>`, low-opacity dots/shapes
4. **Extract shapes** — find all `<polygon>`, `<circle>`, `<path>`, `<rect>` elements that:
   - Are NOT the background rect (skip `width="200"`)
   - Are NOT the vignette rect (skip `fill="url(#vig)"`)
   - Are NOT overlay pattern elements (skip `opacity="0.0x"` as attribute)
   - DO have a `style="opacity:..."` (all foreground shapes have this)
5. **For each shape**, extract:
   - Color: from `fill` attribute, mapped via COLOR_MAP
   - Type: from element tag + vertex count / path commands
   - Size: from vertex coordinates or radius
   - Opacity: from `style="opacity:X"`
   - Rotation: from `transform="... rotate(X)"`
   - Position: from `transform="translate(X,Y) ..."`

### Step 3: Parse All Descriptions

For each of the 200 text descriptions:

1. **Background**: first word (e.g., "milky", "pitch", "tenebrous")
2. **Overlay**: text after "overlaid with" in first sentence
3. **Shape count**: regex `(\d+) elements total`
4. **Individual shapes**: regex pattern:
   ```
   (minuscule|modest|mid-sized|substantial|massive)
   (solid|nearly solid|semi-transparent|faint)
   (color name)
   (shape type)
   ((rotation), at (position))
   ```

### Step 4: Score-Based Matching

The matching algorithm uses a **greedy scoring approach**:

```python
def score_match(img, desc):
    # Hard constraints — must match exactly
    if img['num'] != desc['num']:      return -10000  # Shape count
    if img['bg'] != desc['bg']:        return -10000  # Background type
    if img['overlay'] != desc['overlay']: return -10000  # Overlay type

    score = 100  # Base score for matching hard constraints

    # Soft scoring — color overlap
    for color in img_colors:
        if color in desc_colors:
            score += 10

    # Soft scoring — type overlap
    for type in img_types:
        if type in desc_types:
            score += 8

    # Per-shape detail matching (Hungarian-style greedy)
    for each image_shape:
        find best matching desc_shape by:
            +20 if color matches
            +15 if type matches
            +10 if size matches
            +8  if opacity matches
            +5  if rotation matches
        score += best_match_score

    return score
```

Then greedily assign highest-scoring pairs first:

```python
all_scores.sort(reverse=True)
for score, img_id, desc_id in all_scores:
    if img_id not in matched and desc_id not in used:
        matches[img_id] = desc_id
```

---

## 5. Execution & Submission

### API Calls Made

**1. Start Game**
```
POST /api/apps/69aea07cbcb9a3dd1039a58d/functions/startGame
Body: {"email": "dor.bismut@gmail.com"}
Response: 200 OK (2.2 MB — all game data)
```

**2. Score Game**
```
POST /api/apps/69aea07cbcb9a3dd1039a58d/functions/scoreGame
Body: {
  "matches": {"IMG-032": "DESC-182", "IMG-198": "DESC-134", ...},
  "sessionId": "69c2fd218e5b26f307c941c9"
}
Response: 200 OK
{"correctCount": 200, "scoreSeal": "3d1ac8a7e7be4856"}
```

**3. Generate Token**
```
POST /api/apps/69aea07cbcb9a3dd1039a58d/functions/generateToken
Body: {"sessionId": "69c2fd218e5b26f307c941c9"}
Response: 200 OK
{"token": "WKX-9245FA70-200", "score": 200}
```

### Timeline

| Step | Action |
|---|---|
| 21:07:45 UTC | Game started (startGame called) |
| 21:07–21:15 | Solver script development (parsing + matching) |
| ~21:15 | scoreGame submitted — **200/200 correct** |
| ~21:15 | generateToken called — **WKX-9245FA70-200** |

Total solve time: ~8 minutes of a 30-minute window.

---

## 6. Technical Stack Summary

| Component | Technology |
|---|---|
| Challenge platform | Base44 (low-code app builder) |
| Frontend | React SPA (single JS bundle, ~410KB) |
| Backend | Python/uvicorn behind Cloudflare |
| CDN/Proxy | Cloudflare (HTTP/3, Brotli compression) |
| Auth | JWT (HS256), stored in localStorage |
| Data format | SVG (base64 inline), JSON API |
| Real-time | WebSocket (socket.io) for live session updates |
| Solver | Python 3 (regex parsing, no external libraries) |

---

## 7. Key Insights

### Why This Worked

1. **The JSON submission endpoint** was the critical enabler — without it, we'd need
   browser automation to click 400 times (select image + select description x 200).

2. **Counting-based mapping** was the breakthrough for colors, backgrounds, and overlays.
   Instead of guessing what hex `#cd7f32` maps to in English, we counted that it appears
   65 times in SVGs and "brass" appears 65 times in descriptions — unique count = guaranteed
   match.

3. **Hard constraints eliminate candidates fast.** Each image has a unique combination of
   (background_type, overlay_type, shape_count). With 12 backgrounds x 7 overlays x varying
   shape counts, most images only have a handful of possible description matches, not 200.

4. **SVG is structured data.** Unlike raster images (PNG/JPG), SVGs are XML — every shape,
   color, position, and rotation is explicitly encoded as text. No computer vision needed.

### What Made It Challenging

1. **Overlay detection was tricky.** Background patterns used 5 different SVG techniques:
   `<line>` elements, `<circle>` with stroke-only, `<circle>` with low opacity attribute,
   `<polygon>` with low opacity attribute, and absence of all of these. The key distinction
   was `opacity` as XML attribute (background) vs. inside `style` (foreground shapes).

2. **The color vocabulary was non-obvious.** Names like "oxblood" (`#800020`), "brass"
   (`#cd7f32`), and "deep sapphire" (`#0f52ba`) required the counting approach — you can't
   reliably guess these from hex values alone.

3. **Shape classification from SVG paths** required understanding SVG path commands:
   - `M` (moveto), `L` (lineto), `A` (arc), `Z` (closepath)
   - A donut = two concentric arc paths with `fill-rule="evenodd"`
   - A half-disc = single arc path
   - A crosshair = path with 6+ line segments (cross/plus shape)
   - A pointer = path with 3-5 line segments (arrow shape)
