# TQ4_0 Vulkan Test & Development Notes

## Current Status (2026-04-25)

- **TQ4_0 Mode**: `Compat` (Vulkan-compatible fallback)
- **Tested on**: Ryzen AI 7 350 (RDNA3.5 / gfx1150)
- **GLSL Compilation**: Working (errors resolved)

## Test Commands

### Windows (PowerShell)
```powershell
# With Flash Attention enabled
.\llama-cli.exe -m "model.gguf" -cnv -ngl 99 -ctk tq4_0 -ctv tq4_0 -fa 1

# Auto-detect Flash Attention
.\llama-cli.exe -m "model.gguf" -cnv -ngl 99 -ctk tq4_0 -ctv tq4_0 -fa auto

# Compare with Q8_0 (reference for fallback quality)
.\llama-cli.exe -m "model.gguf" -cnv -ngl 99 -ctk q8_0 -ctv q8_0 -fa 1
```

### Linux/macOS
```bash
# Same commands without .exe extension
./llama-cli -m "model.gguf" -cnv -ngl 99 -ctk tq4_0 -ctv tq4_0 -fa 1
./llama-cli -m "model.gguf" -cnv -ngl 99 -ctk tq4_0 -ctv tq4_0 -fa auto
./llama-cli -m "model.gguf" -cnv -ngl 99 -ctk q8_0 -ctv q8_0 -fa 1
```

## TQ4_0 Mode Explanation

### Compat Mode (Current/Default)
- **Implementation**: Q8_0-style direct dequant (`d * (q_val - 8)`)
- **Vulkan Compatibility**: Full
- **Compression**: ~3-4 bits effective
- **Quality**: Good for long context inference
- **WHT/QJL**: Not used (simplified)

### Full Mode (Reserved for Future)
- **Implementation**: WHT + QJL correction
- **Vulkan Compatibility**: Requires RDNA3.5+ fixes
- **Compression**: ~3-4 bits
- **Quality**: Higher (preserves original TQ algorithm)
- **Status**: Not yet implemented

## Quality Comparison Notes

To evaluate TQ4_0 Vulkan quality vs CPU reference:

1. **CPU TQ4_0** (reference):
   ```bash
   # Run with CPU backend
   ./llama-cli -m "model.gguf" -cnv -ctk tq4_0 -ctv tq4_0 -ngl 0
   ```

2. **Vulkan TQ4_0 Compat** (current):
   ```bash
   ./llama-cli -m "model.gguf" -cnv -ngl 99 -ctk tq4_0 -ctv tq4_0 -fa 1
   ```

3. **Compare**:
   - Perplexity on same prompt
   - Token output quality
   - VRAM usage

## Development Todo

- [ ] Implement `Full` mode TQ4_0 Vulkan path with WHT + QJL
- [ ] Add RDNA3.5 Wave64 optimizations for future
- [ ] Benchmark Compat vs Full mode quality
- [ ] Test on RDNA2 (gfx1035) hardware

## Architecture Notes

```
TQ4_0 Mode Selection (C++ side):
- ggml-vulkan.cpp: vk_tq_params::TQMode enum
  - TQMode::Compat (default) - Q8_0-style dequant
  - TQMode::Full (reserved) - WHT + QJL

GLSL Side (flash_attn_base.glsl):
- DATA_A_TQ4_0 with TQ4_USE_FALLBACK=1
- tq4_decode_and_store_k/v (Compat mode)
- TODO(TQ4_0 full) comments for future implementation
```