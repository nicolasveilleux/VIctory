# Windows NVIDIA VRAM integration

This branch integrates upstream [PR #2389](https://github.com/Marvisak/VIctory/pull/2389),
preserving its commits, and adds independent synthetic Windows checks plus a texture eviction fix.
Nothing in this branch changes the default: full texture storage is used unless
you set `VICTORY_LEVELS_ON_DEMAND=1`. D3D12 tiled resources tier 2 or greater
is required for the reserved path. This is not a claim of gameplay performance.

From PowerShell with the MSVC environment available:

```powershell
cargo build --locked -p victory
.\target\debug\victory.exe --help

$env:VICTORY_D3D12_WARP = "0"
foreach ($mode in "0","1") {
  $env:VICTORY_LEVELS_ON_DEMAND = $mode
  cargo test --locked -p victory-gpu --lib --features diagnostics streamed_bc1_readback -- --ignored --nocapture --test-threads=1
  if ($LASTEXITCODE) { throw "BC1 parity failed for mode $mode" }
}
foreach ($pattern in "sparse","full") {
  $env:VICTORY_VRAM_SAMPLE = $pattern
  foreach ($mode in "0","1") {
    $env:VICTORY_LEVELS_ON_DEMAND = $mode
    cargo test --locked -p victory-gpu --lib --features diagnostics vram_sample_for_streamed_bc1 -- --ignored --nocapture --test-threads=1
    if ($LASTEXITCODE) { throw "DXGI allocation check failed for $pattern mode $mode" }
  }
}
Remove-Item Env:VICTORY_VRAM_SAMPLE, Env:VICTORY_LEVELS_ON_DEMAND, Env:VICTORY_D3D12_WARP
```

The readback checks every BC1 block of every level before and after mapping the
top level from synthetic write-watched memory, including zero blocks for the
unmapped level. The allocation test uses 192 live 2048×2048 BC1 textures with
12 levels, with the top two unneeded in the sparse case. It reports the
process-local DXGI usage delta, including heap granularity, and the non-local
delta; the fully mapped case is a control. Run each measurement in a fresh
process. The workload deliberately isolates texture allocation and is not
representative of a full scene. No game assets or runtime dump are used.

## Measured on the Dell

Windows 11, RTX 2070 Super (8 GB), NVIDIA driver 610.60:
for the same partial workload, DXGI local usage rose 603.980 MB with
the option off and 67.109 MB with it on, with no non-local increase.
When every mip was mapped, both configurations rose 603.980 MB.
Hardware readback passed byte-for-byte in both modes.

With the original pressure selector, the three-texture regression selected
all three and credited zero bytes. With the fix, it selects one and credits
2,097,152 bytes. This exercises the selector against a simulated budget target,
not physical memory exhaustion.
