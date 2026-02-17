# Teachable Machine + SvelteKit

SvelteKit app for running a Teachable Machine image model with webcam inference in the browser.

## Stack

- SvelteKit
- `@tensorflow/tfjs`
- `@teachablemachine/image`

## Setup

1. Install dependencies:

```sh
npm install
```

2. Export your model from Teachable Machine (Image Project).

3. Copy exported files into:

```text
static/tm-my-image-model/
```

At minimum this folder should include `model.json`, `metadata.json`, and the weight shard files.

## Run locally

```sh
npm run dev
```

Open the app, click **Start**, and allow camera access.

## Build

```sh
npm run build
npm run preview
```

## Notes

- The app expects the model at `/tm-my-image-model/`.
- Teachable Machine model artifacts can be large and are currently ignored by git.
