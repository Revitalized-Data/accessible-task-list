# "Goose Process" Siri Shortcut — hands-free photo scan

**Status: this exact recipe was built and verified working end-to-end on 2026-07-10** —
photo → Claude vision API → app opens on a populated Review screen. ~95% field
accuracy on the reference sheet (16/16 pets, all names, split safety tags,
package headers applied per animal); the Review screen catches the rest
(occasional single-digit phone misreads from the API's internal image
downscaling).

Flow: say the Shortcut's name to Siri (or tap it), take/pick the photo, wait
~30–60 seconds, and the task-list app opens with everything filled in.

Cost: a few cents per scan, billed to the Anthropic API key inside the
Shortcut. Rejected/failed requests are never billed.

## The action chain (in order)

1. **Take Photo** (or Select Photos while testing).

2. **Convert Image** — Format: JPEG, Quality: High.
   (Guarantees no HEIC ever reaches the API.)

3. **Resize Image** — input: Converted Image, width **2600**, height Auto.
   (Keeps the upload small and under the API's 5MB image limit. Going bigger
   doesn't help: the API downscales internally to ~1568px on the long edge.)

4. **Base64 Encode** — input: **Resized Image** (NOT Converted Image — easy
   mis-bind). Expand the action and set **Line Breaks: None** (the default
   inserts a newline every 76 chars and corrupts the request body).

5. **Text** — the raw JSON request body. Paste this exactly, then delete the
   `[[BASE64]]` placeholder and insert the Base64 Encoded variable in its
   place (it must sit alone between the quotes — no leftover brackets):

   ```
   {"model":"claude-sonnet-5","max_tokens":16000,"messages":[{"role":"user","content":[{"type":"image","source":{"type":"base64","media_type":"image/jpeg","data":"[[BASE64]]"}},{"type":"text","text":"Extract every row from every table in this photo of a printed pet-boarding list into a JSON array. Return ONLY the JSON array, no prose, no markdown fences. Each element: {\"name\": string, \"tags\": [string], \"details\": [{\"label\": string, \"value\": string}]}. name = the pet's name only, without the (F) or (M). tags = only short printed safety/attention flags like Dog Aggressive or Special Needs, split into separate entries, empty array if none. details = one separate label/value pair per fact, never combined: Gender (Female or Male from the (F)/(M) after the name), Breed, Weight, Age (always as X yrs Y mo, using 0 for whichever isn't shown), Owner(s), Phone, Space, Departing, Package Type (from that table's section header), Date of Service (from that same header), Notes (any other text printed in the Tags column that is not a safety flag, like File already made or Medications), Written Note (only if there is a handwritten note for that row). Include every row from every section. If a value is genuinely unreadable, use UNREADABLE rather than guessing. Return the JSON minified on a single line, with no indentation, spaces, or line breaks between elements."}]}]}
   ```

   Notes on this body:
   - `claude-sonnet-5`, not Haiku: Haiku was tried and misread names/phones
     badly and mangled section headers on this dense sheet.
   - The "minified" instruction at the end is what keeps the response inside
     Shortcuts' ~60s network timeout (pretty-printed output is ~3x the tokens
     and caused `499 Client disconnected` timeouts). Don't remove it.

6. **Get Contents of URL** — the API call.
   - URL (typed as plain text in the top slot — NEVER a variable; putting the
     Text variable here causes "couldn't convert Rich Text to URL" and hangs):
     `https://api.anthropic.com/v1/messages`
   - Method: POST
   - Headers (3):
     - `x-api-key` → the API key (one unbroken line starting `sk-ant-`; use
       the console's copy button — a wrapped/partial paste silently breaks auth)
     - `anthropic-version` → `2023-06-01`
     - `content-type` → `application/json`
   - Request Body: **File** → File: the Text from step 5.
   - First run pops an "Allow ... to connect to api.anthropic.com?" dialog —
     it can open behind other windows and the action waits on it forever.

7. **Get Dictionary Value** — key `content`, in: Contents of URL.
8. **Get Item from List** — First Item, from: Dictionary Value.
9. **Get Dictionary Value** — key `text`, in: Item from List.

10. **URL Encode** — input: the Dictionary Value from step 9 (careful: two
    steps share that output name — bind to the step-9 one).

11. **Text** — the hand-off link, typed text plus the encoded variable:
    `https://revitalized-data.github.io/accessible-task-list/#import=` + URL Encoded Text
    - It must be `#import=`, NOT `?import=`: a full day's list exceeds
      GitHub Pages' query-string limit (HTTP 414); a fragment never goes to
      the server so it has no limit. The app handles the fragment even when
      it's already open in a tab (hashchange listener, added 2026-07-10).

12. **Get Text from Input** — input: the Text from step 11.
    (Fixes "couldn't convert Rich Text to URL" on the next step.)

13. **Open URLs** — input: the output of step 12 only (one chip, nothing else).

## Debugging

- Insert a **Quick Look / Show** action after step 6 showing "Contents of
  URL". API rejections aren't billed and don't stop the Shortcut — they flow
  through silently as an empty payload — so this is the only place the real
  error text (invalid key, image too large, bad base64) is visible.
- `#import=` with nothing after it in the final URL = the extraction chain
  got an error response; check the Quick Look output.
- API-log `499 Client disconnected` = Shortcuts timed out waiting; the
  response was too slow (see the minified note in step 5).

## If accuracy needs a boost later

The API downscales images to ~1568px on the long edge, which is what causes
occasional single-digit phone/age misreads on this dense sheet. The upgrade
path: crop the photo into top and bottom halves in Shortcuts and send both
images in the same request's content array — each half then effectively
doubles its resolution. Not currently wired in; ask Claude Code for the spec.
