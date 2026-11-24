# Car License Plate Detection App

A modern web app for detecting and analyzing license plates from uploaded vehicle images or live camera captures. The project uses FastAPI on the backend, Jinja templates for the frontend UI, and an external Ultralytics-based prediction service for inference.

The UI is branded **PlateVision AI**.

## Table of Contents

- [How It Works](#how-it-works)
- [Features](#features)
- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Configuration](#configuration)
- [Run the App](#run-the-app)
- [Using the App](#using-the-app)
- [Inference Settings](#inference-settings)
- [API Reference](#api-reference)
- [Error Responses](#error-responses)
- [Troubleshooting](#troubleshooting)
- [Notes](#notes)
- [License](#license)
- [Credits](#credits)

## How It Works

```text
┌──────────────┐   image + settings   ┌──────────────────┐   image + Bearer key   ┌──────────────────────┐
│   Browser    │ ───────────────────▶ │  FastAPI server  │ ─────────────────────▶ │ Ultralytics predict  │
│ (upload or   │                      │  POST /predict   │                        │ endpoint (YOLO model)│
│  camera)     │ ◀─────────────────── │  validates input │ ◀───────────────────── │                      │
└──────────────┘   JSON detections    └──────────────────┘   JSON detections      └──────────────────────┘
```

1. The user uploads an image or captures a frame from the camera in the browser.
2. The frontend (`static/script.js`) sends the image together with the confidence, IoU, and image size settings to `POST /predict`.
3. The FastAPI backend (`app.py`) validates the file type, file size, and settings, then forwards the image to the Ultralytics prediction endpoint, authenticating with your API key.
4. The JSON response is returned to the browser, which draws bounding boxes on the image and shows the detection list, statistics, and raw JSON.

The API key stays on the server and is never sent to the browser.

## Features

- Upload vehicle images in JPG, PNG, or WEBP format (file picker or drag and drop)
- Capture images from the browser camera (prefers the rear camera on phones)
- Adjust inference settings such as confidence, IoU, and image size, with a one-click reset to defaults
- Display detection results with bounding-box overlays drawn on the image
- Summary stats: number of detections, top confidence, and inference time
- Per-detection list with label and confidence percentage
- Raw JSON output for debugging
- Clear error messages for invalid files, missing API key, and server timeouts
- Clean, responsive UI with a polished dark theme

## Project Structure

```text
car-license-plate-detection-system/
├── app.py              # FastAPI app: routes, validation, call to the prediction API
├── requirements.txt    # Python dependencies
├── LICENSE             # Apache License 2.0
├── .gitignore
├── static/
│   ├── script.js       # Upload, camera, settings, API call, drawing results
│   └── style.css       # Dark theme styles
└── templates/
    └── index.html      # Main page (Jinja2 template)
```

## Tech Stack

| Layer     | Technology                                  |
|-----------|---------------------------------------------|
| Backend   | Python 3.10+, FastAPI, Uvicorn              |
| HTTP      | HTTPX (async client to the prediction API)  |
| Templates | Jinja2                                      |
| Config    | python-dotenv                               |
| Uploads   | python-multipart                            |
| Frontend  | HTML, CSS, vanilla JavaScript (Canvas API)  |
| Inference | Ultralytics-hosted YOLO model               |

## Prerequisites

Make sure you have the following installed:

- Python 3.10 or newer
- pip
- Git (to clone the repository)
- A working internet connection for the prediction API
- An Ultralytics API key (from your Ultralytics Platform account)

## Setup

1. Clone the repository:

```bash
git clone https://github.com/SonalNikam02/car-license-plate-detection-system.git
cd car-license-plate-detection-system
```

2. Create and activate a virtual environment:

```bash
python -m venv venv
source venv/bin/activate
```

On Windows PowerShell:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

> If PowerShell blocks the activation script, run
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once and try again.

3. Install the dependencies:

```bash
pip install -r requirements.txt
```

4. Create a `.env` file in the project root (see [Configuration](#configuration)).

## Configuration

The app reads its settings from environment variables, loaded from a `.env` file in the project root.

| Variable                     | Required | Default                        | Description                                   |
|------------------------------|----------|--------------------------------|-----------------------------------------------|
| `ULTRALYTICS_API_KEY`        | Yes      | *(empty)*                      | API key sent as a Bearer token to the prediction service |
| `ULTRALYTICS_PREDICTION_URL` | No       | Built-in Ultralytics endpoint  | Override to use your own deployed model endpoint |

Example `.env`:

```env
ULTRALYTICS_API_KEY=your_api_key_here
# Optional:
# ULTRALYTICS_PREDICTION_URL=https://your-prediction-endpoint/predict
```

`.env` is listed in `.gitignore`, so your key will not be committed. Never share or commit your API key.

## Run the App

Start the development server:

```bash
python app.py
```

This runs Uvicorn on `127.0.0.1:8000` with auto-reload enabled. Then open your browser at:

```text
http://127.0.0.1:8000/
```

Alternatively, run Uvicorn directly (useful for changing host/port):

```bash
uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

FastAPI's interactive API docs are also available at:

- Swagger UI: `http://127.0.0.1:8000/docs`
- ReDoc: `http://127.0.0.1:8000/redoc`

## Using the App

1. **Choose an input**
   - **Upload tab:** click the drop zone to pick a file, or drag and drop an image onto it.
   - **Camera tab:** click *Start Camera*, allow camera access, then click *Capture*.
2. **Adjust settings** (optional): confidence, IoU, and image size. Click *Reset* to restore defaults.
3. **Click Predict.** A loading overlay appears while the image is processed.
4. **Review the results:**
   - Bounding boxes with labels drawn over the image
   - Detection count, top confidence, and inference time
   - A list of each detection with its confidence
   - The raw JSON response
5. Click **Clear** to start over.

## Inference Settings

| Setting        | Form field | Default | Allowed range | What it does |
|----------------|------------|---------|---------------|--------------|
| Confidence     | `conf`     | `0.25`  | 0 – 1         | Minimum score for a detection to be kept. Raise it to reduce false positives. |
| IoU            | `iou`      | `0.70`  | 0 – 1         | Overlap threshold for Non-Maximum Suppression. Lower it to remove more duplicate boxes. |
| Image size     | `imgsz`    | `640`   | 32 – 2048     | Size (in pixels) the image is resized to for inference. Larger can find small plates but is slower. |

## API Reference

### `GET /`

Renders the homepage UI.

### `GET /health`

Health check. Useful for confirming the API key is loaded.

```bash
curl http://127.0.0.1:8000/health
```

Example response:

```json
{
  "status": "ok",
  "api_key_configured": true,
  "prediction_url": "https://.../predict"
}
```

### `POST /predict`

Accepts an image as `multipart/form-data` and returns the prediction results from the Ultralytics service.

| Field   | Type   | Required | Default |
|---------|--------|----------|---------|
| `file`  | file   | Yes      | –       |
| `conf`  | float  | No       | `0.25`  |
| `iou`   | float  | No       | `0.70`  |
| `imgsz` | int    | No       | `640`   |

Example with curl:

```bash
curl -X POST http://127.0.0.1:8000/predict \
  -F "file=@car.jpg" \
  -F "conf=0.25" \
  -F "iou=0.7" \
  -F "imgsz=640"
```

Example response shape (the frontend reads these fields):

```json
{
  "images": [
    {
      "results": [
        {
          "name": "license_plate",
          "class": 0,
          "confidence": 0.91,
          "box": { "x1": 412.5, "y1": 288.1, "x2": 560.3, "y2": 331.7 }
        }
      ],
      "speed": { "inference": 42.3 }
    }
  ]
}
```

Box coordinates are in pixels of the original image (top-left `x1, y1` to bottom-right `x2, y2`). The exact fields may vary with the model deployed at your prediction endpoint.

## Error Responses

| Status | Cause |
|--------|-------|
| `400`  | Uploaded image is empty |
| `413`  | Image is larger than 10 MB |
| `415`  | File is not JPG, PNG, or WEBP |
| `422`  | `conf`/`iou` outside 0–1, or `imgsz` outside 32–2048 |
| `500`  | `ULTRALYTICS_API_KEY` is missing from `.env` |
| `502`  | Could not reach the prediction server, or it returned non-JSON |
| `504`  | The prediction server timed out (read timeout is 120 s) |
| other  | Errors from the prediction server are passed through with its status code and message |

## Troubleshooting

- **"ULTRALYTICS_API_KEY is missing"** – Make sure `.env` is in the project root (next to `app.py`), the variable name is spelled exactly, and restart the server. Check `GET /health` → `api_key_configured` should be `true`.
- **401 / 403 from the prediction server** – The API key is invalid or does not have access to the endpoint.
- **Camera does not start** – Browsers only allow camera access on `localhost` or HTTPS. Allow the camera permission when prompted, and close other apps using the camera.
- **Timeout (504)** – The hosted model may be cold-starting. Try again, or use a smaller image size.
- **No detections** – Lower the confidence threshold, use a clearer/closer image of the plate, or increase the image size.
- **`ModuleNotFoundError`** – Activate the virtual environment and run `pip install -r requirements.txt` again.
- **Port 8000 already in use** – Run `uvicorn app:app --port 8001 --reload` instead.

## Notes

- The app expects a valid API key for the prediction service.
- Uploaded files must be smaller than 10 MB.
- Supported image types: JPG, PNG, and WEBP.
- Inference runs remotely, so no GPU or local model weights are needed.
- Images are forwarded to the prediction service and are not stored by this app.

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

## Credits

Based on [entbappy/Car-License-Plate-Detection-App-with-Ultralytics-Platform-](https://github.com/entbappy/Car-License-Plate-Detection-App-with-Ultralytics-Platform-), licensed under the Apache License 2.0. See [LICENSE](LICENSE).
