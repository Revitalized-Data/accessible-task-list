# "Goose Process" Siri Shortcut — hands-free photo scan

Goal: she says "Hey Siri, take photo and run goose process" (or whatever phrase
you name the Shortcut), takes the photo when prompted, and the app opens
straight to the Review screen — no camera button, no copy/paste, no menus.

This calls the same Claude vision API the app uses, so it keeps the same
few-cents-per-scan cost and the same clean name/tags/details structuring —
it just removes every manual step around the photo itself.

Build this in the **Shortcuts** app on her iPhone (Automation isn't needed —
a plain Shortcut named with the phrase you want is enough for Siri to trigger
it by name).

## Steps

1. **New Shortcut.** Name it exactly the phrase she'll say, e.g.
   `take photo and run goose process`. (Siri triggers shortcuts by their name,
   said after "Hey Siri, …".)

2. **Take Photo** action.
   - Show Camera Preview: off (keeps it fully hands-free; turn on later if she
     wants a chance to retake before it processes)
   - Photo Count: 1

3. **Convert Image** action.
   - Input: the photo from step 2
   - Format: JPEG
   - Quality: High
   (Belt-and-suspenders with the app-side fix — guarantees Shortcuts never
   hands Claude a HEIC file either.)

4. **Base64 Encode** action (sometimes listed as "Encode Media").
   - Input: the converted JPEG from step 3
   - Encode: Base64

5. **Get Contents of URL** action — this is the actual API call.
   - URL: `https://api.anthropic.com/v1/messages`
   - Method: POST
   - Headers:
     - `content-type`: `application/json`
     - `x-api-key`: *(her Anthropic API key — type this directly into the
       Shortcut on her device; don't put it in any file or send it to me)*
     - `anthropic-version`: `2023-06-01`
   - Request Body: **JSON**, built as a dictionary:
     ```
     model:      claude-sonnet-5
     max_tokens: 16000
     system:     <paste the exact system prompt below>
     messages:   [ one dictionary ]
       role:    user
       content: [ two items ]
         1) type:   image
            source:
              type:       base64
              media_type: image/jpeg
              data:       <magic variable → Base64 Encoded output from step 4>
         2) type: text
            text: Extract every row from this list as JSON, following the schema exactly.
     ```
   - System prompt to paste in verbatim (same one the app uses, so parsing
     matches exactly):
     ```
     You extract structured task data from a photo of a printed daily list for a blind employee who will hear it read aloud. The photo may be a table with columns such as a name, tags/notes, owner/contact, location, and a time. Return ONLY a JSON array, no prose, no markdown code fences. Each array element must be an object: {"name": string, "tags": [string], "details": [{"label": string, "value": string}]}. "name" is the primary identifier for that row (e.g. a pet or person's name). "tags" should contain only short safety-relevant or attention-relevant flags (e.g. "Dog Aggressive", "Special Needs"), not every note. "details" should contain every other piece of information in that row as separate label/value pairs (e.g. breed/age, owner and phone, space or location, departing date/time, section/package name this row belongs to). Include every row from every section/table on the page. If the image is unclear or empty, return an empty JSON array.
     ```

6. **Extract Claude's reply** — three small actions. The API response from
   step 5 is a JSON envelope; the task-list text is buried at
   `content → first item → text`, so we dig down one layer per action.
   Add them in order and Shortcuts will auto-wire each one's output into
   the next — you only type the two key names.

   6a. **Get Dictionary Value** action.
   - Key: `content`
   - Dictionary: *Contents of URL* (the response from step 5 — should
     fill in automatically; if not, tap the field → Select Variable →
     the output of step 5)

   6b. **Get Item from List** action.
   - Get: **First Item** (the default)
   - List: *Dictionary Value* from 6a (auto-fills)

   6c. **Get Dictionary Value** action (a second one).
   - Key: `text`
   - Dictionary: *Item from List* from 6b (auto-fills)
   - The output of this action is the actual task-list JSON text.

   (Why not one action with key `content.0.text`? Dotted key paths with a
   numeric index are unreliable in Shortcuts — on some iOS versions they
   silently return nothing and the import comes up empty with no error.
   The three explicit actions work everywhere.)

7. **URL Encode** action.
   - Input: the text from step 6c (the second *Dictionary Value*) — make
     sure it's this, not the raw *Contents of URL*

8. **Text** action to build the final link.
   - `https://revitalized-data.github.io/accessible-task-list/#import=` +
     (the URL-encoded text from step 7, inserted as a magic variable)
   - Important: this is `#import=`, not `?import=`. A full day's list is
     large enough that GitHub's server rejects it as a query string ("414
     URI Too Long"). A `#` fragment is never sent over the network at all —
     the app reads it entirely on-device — so there's no length limit.

8b. **Get Text from Input** action.
   - Input: the Text from step 8
   - Fixes a "couldn't convert from Rich Text to URL" error that shows up
     otherwise.

9. **Open URLs** action.
   - URLs: the output of step 8b (not step 8 directly)
   - This opens the PWA (Safari, or the installed home-screen app if she's
     added it) directly on the Review screen, already populated, already
     read aloud.

## Optional but recommended

- Add an **If** action right after step 5, checking whether the response
  dictionary has an `error` key. If it does, use **Speak Text** to say
  "Scan failed, try again" and **Stop This Shortcut**, instead of opening a
  broken URL. This mirrors the app's own error handling.
- Add the Shortcut to her Home Screen or the "Today View" widget as a backup
  trigger in case Siri mishears the phrase.

## Why not fully free (on-device OCR only)?

The `OCR POC.shortcut` already in this folder proves out Apple's free,
on-device text extraction (Take Photo → Extract Text from Image). It costs
nothing and needs no network — but it just reads text in roughly the order
it appears on the page. For this dense, multi-column printed sheet (name,
tags, owner phone, space, date across several sections), that comes out
jumbled — there's no reliable way to know which phone number belongs to
which pet without the structured understanding a vision model provides. If
per-scan cost ever becomes the concern instead of reliability, the fallback
that already works today is: photograph it in the Claude app itself (uses
your subscription, not metered billing) and paste the reply into
`index-paste-version.html`.
