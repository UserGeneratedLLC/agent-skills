---
name: analyze-image
description: Analyze images using Google Gemini via the CLI. Use when the user asks to analyze an image, describe a screenshot, review a UI mockup, examine visual content, compare images, read text from an image, or diagnose visual errors.
---

# Analyze Image

Send images to Gemini for analysis via the `gemini` CLI and the Shell tool.

## Command

Gemini CLI is an agent with built-in file tools. There is no `-f` flag. Instead, include the image's directory and reference the file in the prompt:

```bash
gemini --include-directories "<directory_containing_image>" --approval-mode yolo -m pro "<prompt referencing the filename>"
```

The `--include-directories` flag grants Gemini access to read files outside its default workspace. The `--approval-mode yolo` auto-approves Gemini's internal tool calls (file reads).

## Model Aliases

| Alias | Resolves To | Use For |
|-------|-------------|---------|
| `pro` | gemini-3-pro-preview | Complex analysis, detailed descriptions |
| `flash` | gemini-2.5-flash | Fast responses, simple analysis |
| `auto` | Best for task | Default, recommended |

Do NOT use concrete model names like `gemini-3-pro` -- they error. Use aliases.

## Shell Tool Settings

Image analysis is slow (Gemini reads the file, processes it, generates response). Always set `block_until_ms` to at least `120000` (2 minutes).

## Workflow

1. Get the absolute path to the image.
2. Extract the directory and filename from the path.
3. Build the prompt referencing the filename.
4. Run the command via Shell with `block_until_ms: 120000`.
5. Return Gemini's analysis to the user.

## Prompt Templates

Adapt to the user's request. The prompt must tell Gemini to read the specific file.

**General description:**
```
Read and describe the image file <filename> in the included directory in detail.
```

**UI/UX review:**
```
Read the image file <filename> and review it as a UI screenshot. Identify layout issues, accessibility concerns, and suggest improvements.
```

**Error diagnosis:**
```
Read the image file <filename>. This is a screenshot of an error. Identify the error, explain the cause, and suggest a fix.
```

**Code from screenshot:**
```
Read the image file <filename> and extract all visible code. Return only the code.
```

**Compare (two images in same directory):**
```
Read both image files <file1> and <file2> and describe the differences between them.
```

**OCR:**
```
Read the image file <filename> and extract all readable text verbatim.
```

## Example

```
User: "Analyze this screenshot at D:/screenshots/error.png"

Agent extracts:
  directory: D:/screenshots
  filename: error.png

Agent runs Shell:
  command: gemini --include-directories "D:/screenshots" --approval-mode yolo -m pro "Read the image file error.png and describe what you see. If there are errors, explain them."
  block_until_ms: 120000

Agent returns Gemini's response to the user.
```

## Troubleshooting

- **429 rate limit**: Gemini retries automatically with backoff. If persistent, switch from `pro` to `flash`.
- **"Path not in workspace"**: The image directory was not included. Add `--include-directories`.
- **Model not found**: Use aliases (`pro`, `flash`, `auto`), not concrete model names.
