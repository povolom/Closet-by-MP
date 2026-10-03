# Closet by MP: build spec (version 3: hosted, free, hands-off)

One of the "by MP" projects. Read `MP-PROJECTS-BRIEF.md` first: it sets the branding, hosting, install-as-an-app and learning-pack rules that apply to every app. This file is the product spec. Formerly called Closet Index.

A database of everything in one person's closet, clothes and shoes, that fills itself in from the photos he already takes. It is a page on his website that installs as an app on a phone or computer, and it costs nothing to run.

## The idea

Closet apps fail at setup: nobody finishes photographing 150 items one by one. This design has no setup session.

> **He never catalogues his closet. He takes one mirror photo of what he is wearing each day, and the closet builds itself.**

- **Daily (5 seconds):** one full-length mirror photo with the phone's normal camera, shoes in frame.
- **Weekly (1 minute):** he opens the app, taps **Add photos** and selects the week's photos from his camera roll. The app finds each garment in each photo, works out whether it has seen that garment before, and either logs a wear or adds a new item.
- **After about a month:** the closet holds everything he actually wears, each with a clean cut-out image, a category, colours and a wear count.

No typing and no forms. The app asks a question only when it is unsure.

## Hard rules

1. **It costs nothing.** No paid APIs, no paid hosting, no subscriptions. All image analysis runs on the device, in the browser, with free open models.
2. **Photos never leave the device.** No upload, no server, no account. The site only serves the app's own files. Say this plainly on the About screen.
3. **No typing required, ever.** Every field is optional and editable later.
4. **Faces are never stored.** Only garment cut-outs are kept. The original mirror photo is discarded after processing unless he turns on "keep originals".
5. **Failures are visible.** A failed photo shows why and offers Retry.

## What happens to each photo

```
mirror photo (chosen from the camera roll)
  -> find the garments   (clothes segmentation model: top, bottoms, dress,
                          shoes, hat, bag, scarf ...)
  -> cut each one out    (mask from the same model; transparent background)
  -> fingerprint it      (image embedding + colour profile)
  -> compare with closet (nearest existing item of the same garment type)
        very similar   -> log a wear on that item
        clearly new    -> create the item, auto-tagged
        in between     -> ask one question in the inbox
```

The wear date comes from the date the photo was taken (EXIF), so adding a week late still logs the right days.

**Auto-tagging a new item:**

- **Type:** from the segmentation label (top, bottoms, shoes and so on).
- **Subtype and pattern:** zero-shot image classification against a fixed list of labels (t-shirt, hoodie, button-up, jeans, joggers, sneakers, boots; solid, striped, graphic, plaid).
- **Colours:** cluster the cut-out's pixels, map the top one or two clusters to named colours.
- **Brand, size, price:** left blank. He can add them later; nothing depends on them.

**Models (free, run in the browser with Transformers.js):**

| Job | Model | Notes |
|---|---|---|
| Find and cut out garments | `Xenova/segformer_b2_clothes` | Published with ONNX weights for Transformers.js. Check its licence in M0. |
| Fingerprint and subtype | A CLIP model with ONNX weights for Transformers.js (for example `Xenova/clip-vit-base-patch32`; confirm it exists and loads) | Image embeddings for matching, zero-shot labels for subtype |

- Load models from the Hugging Face hub at first use and let the browser cache them. Do not copy model files into the site: Cloudflare Pages refuses files over 25 MiB.
- Run all model work in a Web Worker so the screen stays responsive. Use WebGPU when the browser has it and fall back to WASM.
- Put each model behind a small interface with a mock version so tests run without downloading anything.

## Known limits (state them in the app)

- **Layers:** a jacket over a shirt comes out as one "upper" piece. The jacket gets logged; the shirt underneath does not.
- **Lookalikes:** two plain black t-shirts will be treated as one item. He can split them by hand if it matters.
- **Lighting and pose** change how a garment looks. The inbox question exists for this; expect a few per week at first.
- **Shoes** need to be in frame.
- **Each device has its own closet.** Data is stored on the device, so the phone and the computer do not share a closet. Export and import move it between them. Syncing through a server is out of scope because photos must stay on the device.
- **Speed depends on the device.** A recent phone should manage; an old one may be slow. M0 measures this before anything else is built.

## The inbox (the only "work")

A short list of yes or no cards, for example:

- "Is this the same hoodie?" with the new cut-out beside the closest match. Tap **Same** or **New**.
- "New item spotted: grey joggers." Tap **Keep** or **Not mine**.

Each answer improves later matching: an item keeps up to five reference fingerprints from confirmed wears.

## Optional accelerator (later)

**Bulk item photos:** photos of single items on a bed or hanger, many at once, for things he wants in the closet without waiting to wear them. Same pipeline; the whole photo is treated as one garment.

## Once the closet has data

| Feature | What it gives him |
|---|---|
| Closet grid | Cut-outs on clean cards; filter by type, colour, subtype |
| Wear counts | Most and least worn, last worn date |
| Not worn lately | Items with no wear in 60 or 90 days |
| Outfit history | Each day's combination, saved automatically from the photo |
| Repeat finder | Combinations he wears most; pieces never worn together |
| Cost per wear | Only for items where he added a price |
| Export and import | One file with everything, to back up or move to another device |

## Stack

- TypeScript with Vite, built to static files and served at `marcantoniopovolo.com/closet/`
- Transformers.js for the models, in a Web Worker
- IndexedDB for records and cut-out images (through a small typed wrapper such as Dexie), with keys and database name prefixed `closet` so it never collides with other "by MP" apps on the same site
- Installable app: web manifest and service worker scoped to `/closet/`, per `MP-PROJECTS-BRIEF.md`. The app shell works offline; models need one online visit to download.
- Vitest for unit tests
- No server code at all

## Data model (IndexedDB object stores)

```
items         id, type, subtype, pattern, colours, name, brand, size, price,
              status (active|archived), cutout (image blob), createdAt,
              firstSeen, lastWorn, wearCount
fingerprints  id, itemId, embedding (Float32Array), colourProfile, sourceWearId
photos        id, takenOn, status (queued|working|done|failed), error,
              original (image blob, only if "keep originals" is on)
wears         id, itemId, photoId, wornOn, confidence, decidedBy (auto|user)
questions     id, photoId, cutout (image blob), candidateItemId,
              kind (same_or_new|keep_or_not), status (open|answered), answer
outfits       id, photoId, wornOn, itemIds
settings      key, value
```

## Screens

1. **Closet:** the grid and filters
2. **Inbox:** open questions, newest first
3. **Add:** photo picker and processing progress
4. **Item:** cut-out, tags, wear history, optional details
5. **Insights:** most and least worn, not worn lately, repeat finder
6. **Settings:** keep or discard originals, matching thresholds, export and import
7. **About:** what the app is, how it works, the privacy promise and the known limits, in his own voice

A visitor who opens the page for the first time sees a demo closet made from sample cut-outs, labelled as a demo, with a button to start their own.

## Milestones

Build one at a time. Stop after each and show it working.

**M0. Model spike (half a day).** A bare test page that takes one mirror photo and shows a cut-out per garment with its type label, and prints the similarity between two photos of the same garment and two different garments. Run it on his laptop and on his phone. Record load time, time per photo and results in `docs/model-notes.md`.
*Done when:* cut-outs look right on five test photos, same-garment similarity is clearly higher than different-garment similarity, and a photo processes on his phone in a time he accepts. If not, stop and report before building further.

**M1. Pipeline on mock models.** Data stores, photo picker, queue, mock segmentation and fingerprints, item creation, wear logging by EXIF date, closet grid.
*Done when:* adding 10 fixture photos produces the expected items and wears, and tests cover the match, new and unsure paths.

**M2. Real models.** Wire in segmentation, embeddings, zero-shot subtype and colour naming in a Web Worker. Thresholds in settings.
*Done when:* over 14 real daily photos, at least 8 in 10 garments are either correctly matched or correctly sent to the inbox, and no face pixels appear in any stored cut-out.

**M3. Inbox and learning.** Question cards, Same or New, Keep or Not mine, reference fingerprints updated from answers, merge and split items by hand.
*Done when:* answering a question changes later matching in a test, and merging two items combines their wears.

**M4. App polish.** Install as an app, offline shell, insights, export and import, demo closet, About screen.
*Done when:* it installs on his phone and computer, opens offline, and an export from one device imports cleanly on the other.

**M5. Bulk item photos (optional).**
*Done when:* 20 single-item photos become 20 items.

## Starter prompt for Claude Code

```
Read MP-PROJECTS-BRIEF.md, then this spec. Follow the Hard rules in both exactly:
nothing paid, nothing uploaded, no faces stored.

Start with M0. Ask me for five full-length mirror photos, then build the test
page and docs/model-notes.md. Show me the cut-outs, the similarity numbers and
the timings on my phone, and wait for my OK before M1.

For M1, show me the folder layout, the data stores and the test list first,
then build it with tests. Stop when M1's "Done when" is met and tell me how
to run it.
```
