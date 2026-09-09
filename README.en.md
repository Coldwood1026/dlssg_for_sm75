# dlssg_sm75 v1.0 — Real DLSS Frame Generation on 20-series (Turing / sm_75)

**2026-09-09**, verified on real hardware: RTX 2060 Max-Q + The Witcher 3 next-gen (DX12). The in-game
**frame-generation toggle can be turned on**, the backend installs successfully
(`backend_install status:0`), and the log shows no `bridge_error`.

> 中文版见 `部署说明.md`。
> Byte-level change list vs. the originals: `改动说明.md` (Chinese).

---

## 1. What this is

It uses the **stock nvngx_dlssg 310.1** as the runtime together with a purpose-built **sm75 backend**:
the backend replaces all 72 kernels embedded in 310.1 (originally sm_89 / sm_120 only) with **sm_75
cubin / PTX** produced by ptxas, then rewrites the architecture gate and the reported architecture to
values that Turing accepts. Three components, three jobs:

| File | Role | What changed |
|---|---|---|
| `version.dll` | stock `dlssg_for_sm86` proxy | **only 10 bytes** differ from the original (all inside the embedded backend resource): 5× SM gate `cmp …,0x56` → `0x4B`; 3× arch report `0x170` → `0x190` (Ada); 2× requirements bridge `0x170` → `0x160` (the real Turing value) |
| `dinput8.dll` | injection vehicle (the game statically imports it, so it is always loaded) | intercepts `nvapi_QueryInterface`: for callers outside the exclusion list it forces `GPU_GetArchInfo` to Ada `0x190` and stubs the Ada-only `D3D12_SetRawScgPriority` to success; it then overrides the runtime's `GetFeatureRequirements` exports to "supported" (Flags=0, arch 0xC0) and NOPs 3 flag sites |
| `runtime\sm75_backend.dll` | sm75 backend (rebuilt) | 144 resources = 72× sm_75 cubin + 72× sm_75 PTX; gate / report / bridge patches identical to those in `version.dll` |
| `runtime\nvngx_dlssg.dll` | runtime | **unmodified**, stock 310.1 |

> `version.dll` and `runtime\sm75_backend.dll` are two independent paths carrying the same patch set.
> This package uses **Pinned + `[Backends]`** to load the external backend (see the INI), so the file
> actually in effect is `runtime\sm75_backend.dll`.

---

## 2. Package contents (SHA256)

| Path | Bytes | SHA256 |
|---|---|---|
| `version.dll` | 10522624 | `2D7DC46E3B81BB29CB698A2B2EA3A4EF2DD3BCB5C90E86A47E035DF4E1EDAA37` |
| `dinput8.dll` | 163840 | `948CADAEAFC24A0CDA1AB541BBC06A4A20DA28A4835646CEA9F5701F4892A210` |
| `dlssg_sm86.ini` | 847 | `8AC5CF8655B3E68C3B44D8039AA43DC11A5E82DAD316B67FD8A5850EDBF36DB7` |
| `runtime\nvngx_dlssg.dll` | 7596088 | `C989C0EBD9CD21DBF3DB0BECD857AA1F218068D94DA371D924331A6F0FB97525` |
| `runtime\sm75_backend.dll` | 2667520 | `D96FD76499E2EBE2926E7EA8266375E528F1F746D7F4D2B97AAA01D0C1996038` |

> The `[Backends]` key **must** equal the actual SHA256 of `runtime\nvngx_dlssg.dll` (i.e. `c989c0eb…`
> in the table above). If you swap the runtime, you must update this key too, otherwise the backend
> will not be loaded.
>
> The `dlssg_sm86.ini` shipped in this package is the **exact byte-for-byte working copy** from the test
> machine (comments included). The comment above its `[Compatibility]` section — "Auto on a forced
> route -> PTX … Try Cubin later" — is out of date: the value actually in effect is `KernelImage=Cubin`,
> so do not "fix" it to match the comment.

---

## 3. Installation

1. Close the game.
2. Copy this folder's `version.dll`, `dinput8.dll`, `dlssg_sm86.ini` and the whole `runtime\` directory
   into the directory that holds the **actual rendering EXE** (Witcher 3 next-gen:
   `...\The Witcher 3\bin\x64_dx12\`), overwriting files with the same name.
3. Clear the old cache: empty `%LOCALAPPDATA%\DlssgSm86\bundles\`.
   (A backend bundle left behind by an older package breaks the SHA matching, so this is mandatory.)
4. Leave the game's own `nvngx_dlssg.dll` alone — the load request is redirected by `version.dll` to
   `runtime\nvngx_dlssg.dll`. If another mod pack (e.g. an FSR3 build) already replaced it, just leave
   it as it is.
5. Launch the game → open **DLSS Frame Generation** in the display / graphics settings.

Requirements: Windows x64 + a DLSS 3 capable driver + an RTX 20-series (Turing) GPU.

---

## 4. Verifying that it really works

**`dlssg_sm86\logs\loader_<pid>.jsonl`**
```
"event":"configuration"   → "runtime_mode":"pinned", "kernel_image_requested":"cubin"
"event":"backend_install" → "status":0, "backend":"...\runtime\sm75_backend.dll"
```

**`dlssg_sm86\logs\backend_<pid>.jsonl`**
```
"event":"mfg_capability" → "changed":true, "reported_max":3      ← multi-frame capability accepted
```
`bridge_error` or `Embedded SM86 kernel missing` **must not** appear.

**`dlssg_sm75_inject.log`** (in the same directory as the EXE)
```
[inject] GetArchInfo arch 0x160 -> 0x190 (Ada)
[inject] GetFeatureRequirements -> success (flags 0, arch 0xC0)
[inject] QI(D3D12_SetRawScgPriority) -> stub
```

In game: the frame-generation toggle switches on and the frame rate clearly goes up.

---

## 5. Known behavior / limitations

- The backend log may contain one entry
  `{"event":"kernel_create", "source_rva":"0xda7a0000", "routed":false, "status":-14}`
  — that is one of nvngx's own empty-blob probe calls (blob=NULL); the backend is not involved and it is
  **harmless**.
- Maximum generated frames: 3 (`MaxGeneratedFrames=3`).
- Only validated with Witcher 3 DX12. Other DLSS-FG capable games should work in principle, but are
  untested.
- `KernelImage=Cubin` is the validated path; switching it to `Auto` falls back to PTX (driver JIT) and
  is meant for troubleshooting only.

---

## 6. Uninstallation

Delete `version.dll`, `dinput8.dll`, `dlssg_sm86.ini`, `runtime\`, `dlssg_sm86\` and
`dlssg_sm75_inject.log`. `version.dll` is the stock proxy file, so Steam's "verify integrity of game
files" will restore the game's own copy.

---

## 7. Sources

- Blueprint: `dlssg_for_sm86-main` (the stock `version.dll` proxy + backend resources).
- Architecture spoofing approach: `dlssg-to-fsr3` (nvapi `GPU_GetArchInfo` reporting Ada).
- `nvngx_dlssg.dll` is an original NVIDIA runtime file, copyright NVIDIA; this package is for local
  testing only.
