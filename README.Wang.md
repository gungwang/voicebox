# 重大福利：支持 Intel Arc A770 及其它 GPU |　Ｓｕｐｐｏｒｔ　Intel Arc A770　ＧＰＵ
## 仅支持开发者模式运行：｜　ｒｕｎ　ｄｅｖ　ｍｏｄｅ　ｏｎｌｙ
- `just setup`
- `just dev`


# Voicebox Work Summary - 2026-04-28

## Goal

Enable local Voicebox development on Windows with Intel Arc A770 using PyTorch XPU, diagnose the false CPU fallback, and make the normal `just dev` path work without any special runtime flag.

## Environment

- Repository: `voicebox`
- Branch: `dev.1`
- OS: Windows
- GPU: `Intel(R) Arc(TM) A770 Graphics`
- Effective runtime Python environment: `backend/venv`
- Important note: Voicebox `justfile` uses `backend/venv`, not the workspace root `.venv`

## Problems Observed

- The initial XPU wheel install emitted pip dependency resolver warnings for `chatterbox-tts` and `hume-tada`.
- `backend/venv` originally contained CPU-only PyTorch (`2.11.0+cpu`).
- `intel_extension_for_pytorch` could not be installed from the PyTorch XPU index in this environment.
- Even after `torch.xpu` could see the Arc A770, Voicebox still reported CPU in some runtime paths.
- The `/health` endpoint initially reported `gpu_available: false` and `gpu_type: null` even though raw XPU execution worked.

## Root Cause

- Voicebox still assumed Intel XPU required a successful `intel_extension_for_pytorch` import before treating `torch.xpu` as usable.
- In this environment, the current PyTorch XPU wheels already expose a working `torch.xpu` runtime without that extra package.
- That stale import gate existed in three places:
	- `backend/backends/base.py`
	- `backend/app.py`
	- `backend/routes/health.py`
- The Windows Intel Arc setup path in `justfile` also tried to install `intel-extension-for-pytorch`, which failed and was not needed for the validated runtime.

## Why The Pip Warnings Were Not The Real Failure

- Voicebox intentionally installs `chatterbox-tts` and `hume-tada` with `--no-deps` in `justfile`.
- `backend/requirements.txt` manually provides the subdependencies Voicebox actually wants.
- `chatterbox-tts` upstream pins older versions like `torch==2.6.0`, `torchaudio==2.6.0`, `diffusers==0.29.0`, and specific transformer versions.
- `hume-tada` upstream pins `torch>=2.7,<2.8` and expects `descript-audio-codec`.
- Because Voicebox intentionally overrides those upstream pins, the resolver warnings after swapping to XPU wheels were expected and did not mean the XPU install had failed.

## Commands Run Today

### 1. Install XPU PyTorch In The Voicebox Backend Environment

```powershell
cd C:/AI/intel-ai/voicebox
./backend/venv/Scripts/pip.exe install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/xpu
```

Result:

- Install succeeded.
- pip printed dependency warnings, but the XPU wheel install itself completed.

### 2. Attempt To Install IPEX Separately

```powershell
cd C:/AI/intel-ai/voicebox
./backend/venv/Scripts/pip.exe install intel-extension-for-pytorch --index-url https://download.pytorch.org/whl/xpu
```

Result:

```text
ERROR: Could not find a version that satisfies the requirement intel-extension-for-pytorch (from versions: none)
ERROR: No matching distribution found for intel-extension-for-pytorch
```

### 3. Verify Raw Torch XPU Visibility

```powershell
cd C:/AI/intel-ai/voicebox
./backend/venv/Scripts/python.exe -c "import json, torch; info={'torch': torch.__version__, 'cuda': getattr(torch.version, 'cuda', None), 'has_xpu': hasattr(torch, 'xpu'), 'xpu_available': torch.xpu.is_available(), 'xpu_count': torch.xpu.device_count(), 'xpu_name': torch.xpu.get_device_name(0) if torch.xpu.device_count() else None}; print(json.dumps(info, indent=2))"
```

Validated state after the XPU wheel install:

- `torch`: `2.11.0+xpu`
- `cuda`: `null`
- `has_xpu`: `true`
- `xpu_available`: `true`
- `xpu_count`: `1`
- `xpu_name`: `Intel(R) Arc(TM) A770 Graphics`

### 4. Verify XPU Can Execute Tensors

```powershell
cd C:/AI/intel-ai/voicebox
./backend/venv/Scripts/python.exe -c "import torch; x=torch.tensor([1.0,2.0]).to('xpu'); y=(x*2).cpu(); print(y.tolist())"
```

Result:

```text
[2.0, 4.0]
```

This proved the Arc A770 was already usable through `torch.xpu` even without `intel_extension_for_pytorch`.

### 5. Verify The Voicebox Device Selector

```powershell
cd C:/AI/intel-ai/voicebox
./backend/venv/Scripts/python.exe -c "from backend.backends.base import get_torch_device; print(get_torch_device(allow_xpu=True, allow_directml=True))"
```

Result after the code fix:

```text
xpu
```

### 6. Verify The User-Facing GPU Label

```powershell
cd C:/AI/intel-ai/voicebox
./backend/venv/Scripts/python.exe -c "from backend.app import _get_gpu_status; print(_get_gpu_status())"
```

Result after the code fix:

```text
XPU (Intel(R) Arc(TM) A770 Graphics)
```

### 7. Start The Backend Normally Through Just

```powershell
cd C:/AI/intel-ai/voicebox
just dev-backend
```

Observed startup:

```text
INFO:     Uvicorn running on http://127.0.0.1:17493
```

### 8. Validate The Health Endpoint

```powershell
curl -sf http://127.0.0.1:17493/health
```

Relevant fields after the health route fix:

```json
{
	"status": "healthy",
	"gpu_available": true,
	"gpu_type": "XPU (Intel(R) Arc(TM) A770 Graphics)",
	"vram_used_mb": 0.0,
	"backend_type": "pytorch",
	"backend_variant": "xpu"
}
```

## Code Changes Made Today

### 1. `backend/backends/base.py`

Change:

- Removed the hard dependency on `intel_extension_for_pytorch` from `get_torch_device(... allow_xpu=True)`.
- Kept the real XPU check as `hasattr(torch, "xpu") and torch.xpu.is_available()`.
- Broadened the guard from `except ImportError` to `except Exception` so probe failures do not force a false CPU fallback.

Before:

```python
import intel_extension_for_pytorch  # noqa: F401
if hasattr(torch, "xpu") and torch.xpu.is_available():
		return "xpu"
```

After:

```python
if hasattr(torch, "xpu") and torch.xpu.is_available():
		return "xpu"
```

### 2. `backend/app.py`

Change:

- Removed the same stale IPEX gate from `_get_gpu_status()`.
- The runtime status label now reports the Arc A770 directly from `torch.xpu`.

### 3. `backend/routes/health.py`

Change:

- Removed the same stale IPEX gate from the `/health` endpoint.
- The public health JSON now correctly reports XPU availability and `backend_variant: "xpu"`.

### 4. `justfile`

Change:

- Removed the Windows Intel Arc step that attempted to install `intel-extension-for-pytorch` from the XPU index.
- Removed the same stale manual installation hint from the CPU fallback message.
- Kept the Arc setup path installing only the XPU-enabled PyTorch packages.

Before:

```powershell
./backend/venv/Scripts/pip.exe install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/xpu
./backend/venv/Scripts/pip.exe install intel-extension-for-pytorch --index-url https://download.pytorch.org/whl/xpu
```

After:

```powershell
./backend/venv/Scripts/pip.exe install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/xpu
```

### 5. `README.Wang.md`

Change:

- Replaced the temporary Arc launch note with this full technical work log.

## Verified Outcomes

- Raw `torch.xpu` runtime is working on the Arc A770.
- Voicebox device selection now returns `xpu`.
- Voicebox GPU status string now reports `XPU (Intel(R) Arc(TM) A770 Graphics)`.
- The normal `just dev-backend` path starts successfully.
- The `/health` endpoint now reports `gpu_available: true` and `backend_variant: "xpu"`.
- No special runtime flag is required for Intel Arc after these code changes.

## Current Run Instructions

```powershell
cd C:/AI/intel-ai/voicebox
just dev
```

Optional verification:

```powershell
curl -sf http://127.0.0.1:17493/health
```

Expected fields:

```json
{
	"gpu_available": true,
	"gpu_type": "XPU (Intel(R) Arc(TM) A770 Graphics)",
	"backend_type": "pytorch",
	"backend_variant": "xpu"
}
```

## Final Notes

- Always validate Voicebox against `backend/venv`; the workspace root `.venv` is a different interpreter.
- There is no separate `just dev --xpu` or IPEX launch flag. The correct path is the normal `just dev` flow.
- Current local modifications are in `README.Wang.md`, `backend/app.py`, `backend/backends/base.py`, `backend/routes/health.py`, and `justfile`.
- No commit was created in this session; this summary describes the current local diff and the validated runtime behavior.

C:\AI\intel-ai\voicebox\tauri\src-tauri\target\release\bundle\msi