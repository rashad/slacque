---
description: Generate a Slack theme from a logo
argument-hint: <logo path> (or paste/drag the image)
---

Generate a custom Slack theme from the provided logo.

Logo source:
- If $ARGUMENTS contains a file path, read that image
- Otherwise, if an image is attached to this message, use it
- Otherwise, ask the user to provide the logo (drag-and-drop, paste, or path)

Once the logo is identified, use the `slacque` skill.
