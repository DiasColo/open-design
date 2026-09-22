---
name: imagegen
description: Generate or edit raster images for project assets through OpenDesign's configured image model. Use for illustrations, mockups, icons, social graphics, visual references, and localized image edits; do not use for code-native SVG, HTML, or canvas artwork.
triggers:
  - "generate image"
  - "create image"
  - "edit image"
  - "image gen"
  - "openai image"
  - "icon design"
  - "mockup"
od:
  mode: image
  surface: image
  category: image-generation
  upstream: "https://github.com/openai/skills"
---

# Image generation

Create the requested image asset, not merely a prompt. Use the OpenDesign media
dispatcher so the project's configured model, credentials, storage, and task
lifecycle remain authoritative. Never call a provider API directly.

## Route the request

- **Generate:** create a new bitmap from a written brief.
- **Edit:** change an existing image while preserving everything the user did
  not ask to change. A referenced source image is required.
- **Do not use this skill** when the deliverable should remain editable as SVG,
  HTML/CSS, canvas, or another code-native format.

Ask a question only when a missing choice would materially change the result,
such as an ambiguous source image or required output aspect. Otherwise infer a
reasonable composition from the request and proceed. Do not invent names,
dates, prices, claims, logos, or brand rules.

## Prepare the job

1. Inspect every referenced image before writing the prompt.
2. Resolve the output aspect from the user's request, then project metadata. If
   neither specifies it, choose the aspect that best matches the intended use.
3. Build a prompt that states:
   - subject and intended use;
   - composition, camera or viewpoint, lighting, palette, and style;
   - exact elements that must appear;
   - exclusions and preservation constraints;
   - for edits, the unchanged regions and identity details that must survive.
4. Treat visible text as exact data. Keep it short. If the configured model
   cannot render it reliably, generate the visual base without text and tell the
   user that a deterministic text-layout pass is still required.
5. Choose a descriptive project-relative `.png` output name. Never overwrite a
   source image or an existing output; add a numeric suffix when needed.

For edits, repeat the preservation constraints in the final prompt. Do not
silently reinterpret an edit as a fresh generation.

## Generate through OpenDesign

Use the project metadata values already supplied to the agent for project ID,
image model, and aspect. Add one `--image` argument for each required reference.

```bash
out=$("$OD_NODE_BIN" "$OD_BIN" media generate \
  --project "$OD_PROJECT_ID" \
  --surface image \
  --model "<imageModel from project metadata>" \
  --aspect "<resolved aspect>" \
  --output "<project-relative-output>.png" \
  --prompt "<complete prompt>" \
  [--image "<project-relative-reference>"])
ec=$?
if [ "$ec" -ne 0 ]; then printf '%s\n' "$out" >&2; exit "$ec"; fi

last=$(printf '%s\n' "$out" | tail -1)
task_id=$(printf '%s\n' "$last" |
  python3 -c "import json,sys; print(json.load(sys.stdin).get('taskId',''))" 2>/dev/null)
since=$(printf '%s\n' "$last" |
  python3 -c "import json,sys; print(json.load(sys.stdin).get('nextSince',0))" 2>/dev/null)
since="${since:-0}"

while [ -n "$task_id" ]; do
  out=$("$OD_NODE_BIN" "$OD_BIN" media wait "$task_id" --since "$since")
  ec=$?
  last=$(printf '%s\n' "$out" | tail -1)
  since=$(printf '%s\n' "$last" |
    python3 -c "import json,sys; print(json.load(sys.stdin).get('nextSince',0))" 2>/dev/null)
  since="${since:-0}"
  if [ "$ec" -eq 0 ]; then
    task_id=""
  elif [ "$ec" -ne 2 ]; then
    printf '%s\n' "$out" >&2
    exit "$ec"
  fi
done

printf '%s\n' "$last"
```

The final line must be JSON containing `file.name`. Record that returned name;
do not assume the requested filename was used unchanged.

If required environment metadata is absent, the model is not image-capable, or
an edit model cannot accept references, stop and report the missing capability.
Do not work around the dispatcher with direct API calls.

## Verify and hand off

Inspect the generated image and check the user-visible requirements: subject,
composition, aspect, preserved details, prohibited elements, and exact visible
text. For an edit, compare it with the source and confirm that unrelated regions
did not drift.

If a clear prompt-level defect is found, make at most one adjusted retry for
that output. Do not loop indefinitely or broaden the requested change. If the
second result still fails, return the best result and describe the remaining
issue plainly.

Reply with the final filename, a concise summary of what was generated or
changed, and any limitation that still needs human review. Do not emit an
`<artifact>` tag.

## Invariants

- Produce a real image file; never stop at a prompt when generation is
  available.
- Preserve source files and unrelated regions in edits.
- Use project-relative references and outputs.
- Use `"$OD_NODE_BIN" "$OD_BIN" media generate` and `media wait` only.
- Never fabricate factual or brand content.
- Never claim successful generation without a final `file.name` result.
