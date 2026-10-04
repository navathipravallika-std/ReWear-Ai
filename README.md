# ReWear-AI

A modern, sustainability-focused landing page for **ReWear-AI** — an AI-powered
clothing resale platform. Built with React and Vite.

> Frontend only. There is no backend yet: all page content lives in local React
> components and the service layer answers from sample data. The image upload page
> at `/upload` validates and previews files in the browser, but its "Analyze" step
> returns sample data. Analysed items are saved to a **Virtual Wardrobe** held in
> `localStorage`, so they survive a refresh without a server. The `/style` page then
> styles outfits from those saved items with a local, rule-based recommendation engine.
> The `/stylist` page chats through that same engine to suggest looks you can refine,
> and the outfits it saves can be revisited on `/saved-outfits`.

## Getting started

```bash
npm install
npm run dev
```

The dev server starts on <http://localhost:5173> and opens automatically.
The upload page is at <http://localhost:5173/upload>, the wardrobe at
<http://localhost:5173/wardrobe>, the stylist chat at
<http://localhost:5173/stylist> and saved outfits at
<http://localhost:5173/saved-outfits>.

## Scripts

| Script            | What it does                       |
| ----------------- | ---------------------------------- |
| `npm run dev`     | Start the dev server with hot reload |
| `npm run build`   | Build a production bundle into `dist/` |
| `npm run preview` | Serve the production build locally  |
| `npm run lint`    | Check the code with ESLint          |
| `npm test`        | Run the unit tests with Vitest      |

## Routes

| Path        | Page           | What it does                                                    |
| ----------- | -------------- | --------------------------------------------------------------- |
| `/`         | `LandingPage`  | The original home page                                          |
| `/upload`   | `UploadPage`   | Pick a photo, preview it, remove it, analyse it and add it to the wardrobe |
| `/wardrobe` | `WardrobePage` | Every saved item, with category filters, search and remove      |
| `/style`    | `StylePage`    | Pick an occasion, style and weather, then build an outfit from saved items |
| `/stylist`  | `StylistPage`  | Chat with the stylist about an occasion, a piece you own, jewellery or several options |
| `/saved-outfits` | `SavedOutfitsPage` | Every saved outfit, with a detail view and delete |
| anything else | —            | Redirects to `/`                                                |

`App.jsx` holds the shared shell (skip link, navbar, footer) and the route list.
The navbar and footer sit outside `<Routes>`, so they appear on every page.

## The API service

`src/services/api.js` is the only file that knows how to talk to a server. No
component imports `fetch` or `FormData`, so the shape of a request can change in
one place.

| Function                              | Mock behaviour                                        | Real call                        |
| ------------------------------------- | ----------------------------------------------------- | -------------------------------- |
| `analyzeImage(file, { signal })`      | waits 1.6s, returns labelled sample data              | `POST /analyze` (multipart)      |
| `createListing(draft, { signal })`    | waits 0.6s, assigns an id, stores in memory           | `POST /listings` (JSON)          |
| `getWardrobe({ signal })`             | waits 0.22s, reads `localStorage`                     | `GET /wardrobe`                  |
| `addToWardrobe(analysis, { file, signal })` | waits 0.22s, re-encodes the photo, saves to `localStorage` | `POST /wardrobe` (multipart) |
| `removeFromWardrobe(id, { signal })`  | waits 0.22s, deletes from `localStorage`              | `DELETE /wardrobe/:id`           |

`createListing` is kept for the future publishing flow, where a seller edits a
draft before listing it. The upload page uses `addToWardrobe` instead.

Every response carries a `source` field — `"mock"` or `"live"`. The upload page
uses it to decide whether to show the **Sample result** badge, so that badge
disappears on its own once a real backend is connected. No UI change needed.

### Connecting a real backend

1. In `src/services/api.js`, set `MOCK_MODE = false`.
2. Copy `.env.example` to `.env` and set `VITE_API_URL` to your backend address.
3. Make the backend answer with the same shapes shown in `SAMPLE_ANALYSIS`.

That is the whole migration. `handleAnalyze` and `handleAddToWardrobe` stay as they
are, because they only ever talk to these functions.

Both accept an `AbortSignal`. The pages pass one in and abort it when the user picks
another image, re-runs the analysis, or leaves the page, so a slow response can never
overwrite a newer one.

## Upload page

`/upload` accepts **JPG, PNG and WEBP** images up to **10 MB**.

- **Validation** — `validateImage()` rejects anything else before it leaves the
  device, and shows a plain-language reason.
- **Preview** — built with `URL.createObjectURL()`. The old URL is revoked whenever
  the image changes and again when the page unmounts, so nothing leaks.
- **Remove image** — clears the file, the preview, the error and the result, and
  resets the input so picking the same file twice still works.
- **Analyze image** — calls `analyzeImage()` from the service. In mock mode the
  result is fixed sample data, clearly labelled so it cannot be mistaken for a real
  analysis.
- **Add to Wardrobe** — calls `addToWardrobe()`, which re-encodes the photo and
  saves the item to `localStorage`. A confirmation links straight to `/wardrobe`.

Images can be chosen by clicking the dropzone or by dragging one onto it.

## Wardrobe page

`/wardrobe` lists every saved item newest first, in a responsive card grid. Each
card shows the photo, category, colour, condition, estimated price and the model's
recommendation, and offers **Style This** and **Remove**.

- **Filters** — a row of category buttons: All, Tops, Bottoms, Dresses, Shoes,
  Accessories. The list lives in `src/data/wardrobeCategories.js` so the upload
  analysis, the filters and the cards all share one vocabulary.
- **Search** — free text matched (case-insensitively) against the item name,
  category and colour.
- **Matching rules** — `filterWardrobeItems()` in `src/utils/wardrobe.js` is a pure
  function, kept out of the page so it is easy to read and test.
- **Empty states** — one for a wardrobe with no items yet (with a link to upload),
  and one for filters or a search that match nothing (with a **Clear filters**
  button).
- **Storage** — items are kept under the `rewear-ai:wardrobe` key. Photos are
  re-encoded as JPEG data URLs, no bigger than 640px on the longest side, so a
  wardrobe of dozens of items still fits in the roughly 5 MB browsers allow.

`Style This` links to `/style` with the item, which pins it as the starting piece
for the outfit suggestion.

## Style Me page

`/style` builds an outfit from the clothes already in the wardrobe. The user picks
an **occasion** (College, Office, Party, Wedding, Date, Vacation, Casual Outing), a
**style** (Casual, Formal, Elegant, Traditional, Trendy, Minimal) and the **weather**
(Hot, Mild, Rainy, Cold), optionally taps a wardrobe item to build around it, then
presses **Generate Outfit**.

- **Recommendation engine** — `generateOutfit()` in `src/services/styleService.js` waits 1.2s
  and then calls `recommendOutfit()` in `src/engine/recommendationEngine.js`. The choice
  is made only from saved items; nothing is invented and nothing leaves the browser.
- **Pieces** — the result shows the top, bottom-or-dress, shoes and an optional
  accessory, each with its photo and condition. Suggested jewellery is listed too,
  because jewellery is not something the wardrobe stores.
- **Completeness** — an outfit needs a dress *or* a top and bottom, plus shoes. If
  the wardrobe is missing a piece, the page explains what is needed and links to
  `/upload` instead of inventing an item.
- **ReWear Score** — a 0–100 dial shown with the outfit. It is labelled on screen as
  a **ReWear-AI estimate** based on condition and reuse, not an official
  environmental measurement.
- **Save Outfit** — writes a light summary (no images) under the
  `rewear-ai:outfits` key. **Try Another Outfit** re-runs the stylist while excluding
  the pieces just used, so the next suggestion differs.
- **Storage** — `src/services/api.js` is left untouched; `styleService.js` only
  imports `getWardrobe` from it, keeping the mock/live boundary intact.

This is an independent module: `api.js` holds the existing wardrobe endpoints, and
`styleService.js` is the seam to replace when a real styling model exists. Keep the
function names and shapes and swap the internals, as with `api.js`.

### How the recommendation engine works

The engine lives in `src/engine/`, split so each stage can be read and tested on its
own:

| File                      | Responsibility                                                 |
| ------------------------- | -------------------------------------------------------------- |
| `styleRules.js`           | The rule data: keywords, colour families, weights and jewellery |
| `itemSignals.js`          | Turns a raw item into role / style / weather / colour signals   |
| `scoring.js`              | One small score per rule, plus the colour-harmony calculation   |
| `recommendationEngine.js` | Filters, ranks and searches for the best set, then explains it  |
| `reuseScore.js`           | The labelled ReWear Score, shared with saved outfits            |

It runs in five steps: **normalise** each item, **filter** out anything wrong for the
weather, **score** what remains, **combine** the best candidates into a set, then
**explain** the winner. Scores come from transparent weights, so the explanation can
point at the rules that actually fired:

- occasion match (30), style match (25) and weather match (20) per item;
- category suitability (10) and a condition bonus (10) per item;
- colour harmony across the finished set (15);
- a large penalty for pieces the user asked to avoid, which powers **Try Another**.

Nothing is invented — every piece comes from the wardrobe. When a real AI/ML model
exists, replace the scoring and combination steps inside `recommendationEngine.js`
while keeping `recommendOutfit()` and its return shape; `src/utils/style.js` is only a
thin facade, so the pages never change.

## Saved Outfits page

`/saved-outfits` lists every outfit written by **Save Outfit**, newest first.

- **Loading** — the page reads `getSavedOutfits()` and the wardrobe together with
  `Promise.all`, so each card can still show a photo when the item is in the wardrobe.
  An `AbortController` cancels both reads if the visitor leaves the page.
- **Cards** — `SavedOutfitCard` shows each piece (with the photo when it still
  exists), the occasion, style, weather, suggested jewellery, the ReWear Score and
  the date saved. `resolveOutfitPieces()` in `src/utils/savedOutfits.js` maps the
  stored summary back onto the top/bottom/dress/shoes/accessory slots.
- **Detail view** — **View Outfit** opens a modal that reuses `OutfitResult` with its
  actions hidden, so a saved outfit looks exactly like a freshly generated one. Close
  it with the button, a click on the backdrop, or the Escape key.
- **Delete** — **Delete** asks for confirmation first, then removes the outfit
  optimistically. `deleteSavedOutfit()` in `styleService.js` rewrites the
  `rewear-ai:outfits` list, and the card is put back if the request fails.
- **Empty state** — when nothing is saved yet, the page explains where outfits come
  from and links to `/style`.

## Stylist chat page

`/stylist` is the conversational front door to the same recommendation engine. The
user types a question — or taps one of the starter prompts — and the stylist replies
with an outfit card and follow-up actions. It is **not** a real language model: it is
a transparent, rule-based assistant, labelled **Mock assistant** in the UI.

- **What it understands** — `src/stylist/intentParser.js` reads each message for an
  occasion, a style, a weather word, a named wardrobe item, or a request for several
  outfits. It returns one of `outfit`, `item`, `item-missing`, `multiple`, `more`,
  `jewellery`, `help` or `unknown`.
- **What it answers** — `src/stylist/stylistEngine.js` turns that intent into the
  reply: the wording, the outfit payloads and the quick actions. It calls the same
  `recommendOutfit()` and `suggestJewellery()` the Style Me page uses, so a chat
  suggestion is a real suggestion drawn from the user's own wardrobe.
- **Follow-ups** — the page remembers the last preferences and the pieces already
  used. **Try Another** (and "give me another") exclude those pieces so the next idea
  differs, and a named piece is pinned so it always appears in the result.
- **Actions on a card** — **Style This Outfit** opens `/style` with the occasion,
  style and weather (and the main piece) pre-filled; **Save Outfit** writes to the
  same `rewear-ai:outfits` list as the Style Me page; **Try Another** refines; **View
  Wardrobe** opens `/wardrobe`.
- **No wardrobe?** The reply explains that it styles only what the user owns and
  links to `/upload`, rather than inventing pieces.
- **Memory** — `src/utils/chatStorage.js` keeps the conversation under
  `rewear-ai:stylist-chat`. Images are stripped before saving and rebuilt from the
  current wardrobe on load, so the chat does not bloat storage.

`src/services/stylistService.js` is the async seam. It waits 0.7s (so the typing
indicator is visible), calls the engine and marks the reply `source: "mock"`. Keep
`sendStylistMessage(text, { items, context, signal })` and its return shape, and a
real LLM can be dropped in later without touching the chat UI:

```js
{
  text,            // the assistant's sentence
  outfits,         // zero or more outfit payloads (each with `id` and `preferences`)
  quickActions,    // page-level actions such as 'upload' or 'wardrobe'
  chips,           // suggested follow-up questions
  needsWardrobe,   // true when the wardrobe is empty
  missing,         // slot names still needed to complete a look
  preferences,     // the occasion/style/weather the reply was built for
  context,         // { preferences, lastUsedIds, lastIntent } for the next turn
  source: 'mock',
}
```

## Tests

`vitest` + `@testing-library/react` cover the recommendation engine, the pure helpers
and the saved-outfit UI. `vite.config.js` sets `test.environment` to `jsdom`. The page
tests mock `services/styleService` and `services/api` so they never touch
`localStorage` or wait on the mock latencies.

The engine tests sit next to the code: `itemSignals.test.js`, `scoring.test.js` and
`recommendationEngine.test.js`. They check occasion, style, weather and colour
matching, the unsuitable-clothing filter, incomplete wardrobes, pinned items, Try
Another, jewellery and the ReWear Score, so the rules cannot silently drift.

The Stylist tests sit next to their code too: `intentParser.test.js` checks what each
message is understood to mean, `stylistEngine.test.js` checks the replies (including
distinct multi-outfit results and the empty-wardrobe case), and `stylistService.test.js`
checks the async seam. `StylistPage.test.jsx` mocks `services/stylistService`,
`services/styleService` and `services/api` so it never waits on the mock latencies.

## Project structure

```
public/
  favicon.svg            Leaf icon used as the browser tab icon
src/
  main.jsx               Entry point — mounts <App /> into #root inside <BrowserRouter>
  App.jsx                Shared shell + route list
  index.css              All styling, grouped into 19 labelled blocks
  data/
    wardrobeCategories.js  The shared category list used by filters and analysis
    styleOptions.js      Occasions, styles and weather choices for /style
  stylist/
    intentParser.js      Reads a chat message into an occasion/style/item intent
    stylistEngine.js     Builds the reply from that intent, reusing the engine
  engine/
    styleRules.js        Rule data: keywords, colour families, weights, jewellery
    itemSignals.js       Normalises a wardrobe item into scoreable signals
    scoring.js           One transparent score per rule, plus colour harmony
    recommendationEngine.js  Filters, ranks and combines the best set
    reuseScore.js        The labelled ReWear Score
  services/
    api.js               The only file that talks to a server (mock or real)
    styleService.js      Style service: generates, saves, lists and deletes outfits
    stylistService.js    Chat seam: sends a message to the stylist engine (mock)
  utils/
    format.js            Shared price, condition, colour and date formatting
    image.js             Downsizes a photo and re-encodes it as a data URL
    wardrobe.js          Pure filtering rules for the wardrobe grid
    style.js             Thin facade re-exporting the engine for the service
    savedOutfits.js      Maps a saved record back onto the outfit UI shapes
    chatStorage.js       Saves the stylist chat, stripping and restoring images
  pages/
    LandingPage.jsx      Home page, lists the sections in reading order
    UploadPage.jsx       Image upload, preview, remove, analyze and add to wardrobe
    WardrobePage.jsx     Saved items with filters, search and remove
    StylePage.jsx        Occasion/style/weather picker + generated outfit
    StylistPage.jsx      Chat with the stylist and act on the suggested outfits
    SavedOutfitsPage.jsx Saved outfits with a detail modal and delete
  components/
    Button.jsx           Renders a <Link> with `to`, an <a> with `href`, or a <button>
    Section.jsx          Page band with consistent spacing + background tone
    SectionHeading.jsx   Reusable eyebrow / title / description block
    Icon.jsx             Small inline SVG icon set (no icon library needed)
    Logo.jsx             Wordmark reused in the navbar and footer
    Navbar.jsx           Sticky navbar with a mobile menu
    SearchBox.jsx        Labelled search field
    CategoryFilter.jsx   Row of category buttons
    ClothingCard.jsx     One saved wardrobe item
    OptionPicker.jsx     Pill group for occasion, style and weather
    WardrobePicker.jsx   Tap a saved item to build the outfit around it
    OutfitSlot.jsx       One slot (top/bottom/shoes/accessory) in a result
    OutfitResult.jsx     The finished outfit, explanation and actions
    ReWearScore.jsx      Dial showing the project-specific ReWear Score
    SavedOutfitCard.jsx  One saved outfit with a view and confirm-delete actions
    ChatMessage.jsx      One chat row: bubble, outfit cards and follow-up chips
    ChatInput.jsx        Message box for the stylist chat
    SuggestionChips.jsx  Tappable question prompts
    OutfitChatCard.jsx   One suggested outfit with its chat actions
    Hero.jsx             Main headline, calls to action, trust points
    HeroVisual.jsx       Product-style preview card
    HowItWorks.jsx       Four-step explainer
    Features.jsx         Six feature cards
    Impact.jsx           Dark stats band with checklist
    Testimonials.jsx     Three customer quotes
    CallToAction.jsx     Closing sign-up band
    Footer.jsx           Link columns and legal row
```

## How the reusable pieces fit together

`Button`, `Section`, `SectionHeading`, `Icon` and `Logo` are the shared building
blocks. Every content section is composed from them, so restyling the whole site
usually means editing `index.css` only.

`Section` accepts a `tone` of `light` (default), `mint` or `dark`:

```jsx
<Section id="features" tone="mint" labelledBy="features-title">
  <SectionHeading
    id="features-title"
    eyebrow="What you get"
    title="Everything resale needs"
    description="Short supporting sentence."
  />
  {/* ...cards */}
</Section>
```

`Button` picks its own element based on the props you pass:

```jsx
<Button to="/upload">Start selling</Button>        {/* navigates, no page reload */}
<Button href="#features">Jump down</Button>          {/* in-page anchor */}
<Button onClick={handleAnalyze}>Analyze</Button>    {/* runs a function */}
```

Section links use the `/#id` form (for example `/#impact`) so they keep working
when a visitor is on another page. `App.jsx` reads the hash and scrolls to it.

To add an icon, add a small function to `Icon.jsx` and register it in the `icons`
object at the bottom of that file.

## Design and accessibility notes

- **Responsive**: mobile-first CSS with breakpoints at 1040px, 900px, 860px, 620px,
  520px and 480px.
- **Design tokens**: colours, radii, shadows and spacing live in `:root` at the top
  of `index.css`, so the palette can be changed in one place.
- **Accessible**: semantic landmarks, a skip link, visible focus rings, labelled
  navigation, decorative SVGs marked `aria-hidden`, `role="alert"` on validation
  messages, and full support for `prefers-reduced-motion`.
- **Keyboard friendly**: the dropzone is a `<label>` wrapping a real file input, so
  it is reachable by Tab and opens the picker with Enter or Space.
- **No UI dependencies**: plain CSS and inline SVG only, which keeps the bundle tiny
  and the code easy to follow.

## Deploying

`BrowserRouter` means `/upload`, `/wardrobe`, `/style`, `/stylist` and
`/saved-outfits` are real URLs, so the host must rewrite unknown paths to
`index.html` or a refresh on one of them will 404. Netlify: `/* /index.html 200`.
Vercel: add a `rewrites` entry. Nginx: `try_files $uri /index.html`.

## Next steps

- Add more pages (buy, impact report, seller dashboard) on top of `services/api.js`.
- Add a button to publish a saved outfit as a listing.
- Replace the scoring and combination steps in `engine/recommendationEngine.js` with a
  real AI/ML model, keeping `recommendOutfit()` and the same `generateOutfit` /
  `saveOutfit` / `getSavedOutfits` / `deleteSavedOutfit` shapes.
- Connect a real language model behind `services/stylistService.js`: keep
  `sendStylistMessage(text, { items, context, signal })` and the same response shape,
  then swap `intentParser.js` (and any wording in `stylistEngine.js`) for the model,
  while the engine still does the actual outfit selection.
- Grow the test suite (Vitest + Testing Library) to cover `api.js`,
  `utils/wardrobe.js` and the upload/wardrobe/style flows.
- Replace placeholder copy and stats with real product data.
