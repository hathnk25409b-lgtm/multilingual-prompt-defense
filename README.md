# LYRA — your LinguisticXLMR detector

Updated specifically for `interprt111.ipynb`: XLM-R first-token representation, 6 stemming features, 12 structure features, fitted StandardScalers, two optional 32-unit branches, and the custom 128-unit classifier. No retraining or conversion is required.

## 1. Copy your Kaggle files

Put the entire contents of `saved_model` into `backend/model/`. The directory must contain:

- model.pt
- scalers.joblib
- config.json
- tokenizer.json (or the tokenizer assets exported by save_pretrained)
- tokenizer_config.json
- Any other exported tokenizer files (keep them)

Only use your own trusted exports: joblib deserializes Python objects. Do not rename model.pt to safetensors. The config architecture field may still say XLMRobertaForMaskedLM; the backend reconstructs the custom class and strictly loads the checkpoint state_dict instead of inferring the classifier from that field.

## 2. Start locally on Windows — inference first

Install Python 3.11 or 3.12 and Node.js 20.19+. Open a PowerShell terminal in the extracted `lyra` directory.

```powershell
powershell -ExecutionPolicy Bypass -File backend/local-test.ps1
```

This one-process script creates .venv, installs Python dependencies, and starts the API at http://127.0.0.1:8000 with authentication explicitly disabled for a local inference check. The ExecutionPolicy option applies only to this child process. You can instead run its commands manually in your terminal.

Keep that terminal open. In a second terminal:

```powershell
npm ci
npm run dev
```

Open http://localhost:4173. In Connections, set `http://localhost:8000/predict`, leave token forwarding off for this local test, then Save connection. Choose the correct language and analyze a short prompt. The result should say Model API and have an actual model score. Check http://localhost:8000/health if it does not connect.

The training labels are 0=BENIGN, 1=JAILBREAK, 2=PROMPT_INJECTION, validated against checkpoint metadata. No weights are included here. Your actual checkpoint has not been run in the development environment; validate on your machine with the exported model.

## 3. Turn on TideCloak after inference works

Keep your existing Docker TideCloak instance running. Do not create a second server. We still need its exact URL, realm, browser client ID, and API token audience to finish your account-specific setup.

In your public browser client, allow:

- Valid redirect URI: http://localhost:4173/
- Web origin: http://localhost:4173

Export the public Tide adapter JSON. In LYRA → Connections, import it, save, and sign in. Never import client secrets/admin credentials.

Stop the local-test API (Ctrl+C). In PowerShell, configure the exact values from your server:

```powershell
$env:MODEL_DIR = (Resolve-Path ./backend/model).Path
$env:REQUIRE_AUTH = 'true'
$env:TIDECLOAK_ISSUER = 'http://localhost:8080/realms/YOUR_REALM'
$env:TIDECLOAK_AUDIENCE = 'YOUR_API_AUDIENCE'
$env:APP_ORIGINS = 'http://localhost:4173'
./.venv/Scripts/python.exe -m uvicorn backend.app:app --host 127.0.0.1 --port 8000
```

`YOUR_REALM` and `YOUR_API_AUDIENCE` are placeholders. Client ID and token audience are not automatically identical: configure an audience mapper in TideCloak if needed. This backend currently verifies RS256 tokens. Enable token forwarding in LYRA only for your trusted backend. A signed-out/invalid token must return HTTP 401; a valid token must pass signature, issuer, audience, and expiry verification. Local history and demo UI are browser-local and not gated by this backend.

## 4. Optional: run the API in Docker

Docker is not required to load your model. Once local inference works, you may run the API with the included Dockerfile and compose.yaml. It mounts `./backend/model` read-only instead of copying the 1 GB model into the image. Your frontend can stay on npm run dev.

For a local inference test:

```powershell
$env:REQUIRE_AUTH = 'false'
docker compose up --build
```

For authenticated Docker inference, stop that container, set the real issuer and audience as above, and also set a container-reachable JWKS URL:

```powershell
$env:REQUIRE_AUTH = 'true'
$env:TIDECLOAK_JWKS_URL = 'http://host.docker.internal:8080/realms/YOUR_REALM/protocol/openid-connect/certs'
docker compose up --build
```

The issuer must still match the token's exact issuer (often localhost on your laptop). `host.docker.internal` is only the network address used by the API container to fetch public verification keys. It assumes your existing TideCloak port is published on host 8080; substitute the real port. The API is bound to host loopback 127.0.0.1:8000. Docker build/runtime have not been tested here; no Docker daemon or real model/server is available in the development environment.

## What is preserved from training

- Original text is fed to tokenizer and feature extraction, without hidden stemming/normalization substitutions.
- Stemmer languages: en, fr, it, pt. Other languages receive zero raw morphology features BEFORE applying the saved scaler, exactly as in the notebook.
- Structure feature expressions and order are copied directly from training.
- Scalers are loaded, never fitted again.
- Checkpoint branch flags and max_length are restored. Your notebook uses max_length=256.
- Long input is rejected with a clear error instead of silently truncating and missing an attack at the end. Short input inference matches the training path.
- No feature attributions or suspicious spans are fabricated. The trained classifier outputs class logits only.
- Prompt source is forwarded by the UI but this trained model was not trained to consume it.

## Frontend features and deployment

Demo rules remain available, visibly labeled. No API failure silently falls back to demo. Single and CSV batch inference are supported; history is plaintext localStorage capped at 200 results. Low-resource accuracy depends on your validation data; stemming support for four languages is not full morphological parsing or etymology.

The hosted UI needs accessible HTTPS model and TideCloak endpoints. Run the downloaded source locally while using Docker localhost; do not paste a localhost address into the hosted UI and expect the same behavior.

Before public use, add suitable rate/request limits and operational controls. This detector does not itself execute or block downstream chatbot actions.

## Verification performed for this adapter

- Compared 36 stemming feature vectors and 6 structure vectors against the uploaded notebook: exact matches.
- Saved/loaded tiny synthetic checkpoints and compared inference against the notebook's original class for all four branch configurations: probabilities matched.
- Checked API health/prediction contract, blank/long-input rejection, checkpoint label/head mismatch rejection, and missing/invalid token rejection.
- These checks verify implementation compatibility, not classification accuracy. The user's real model weights, Docker runtime, and actual TideCloak login still need local testing.
