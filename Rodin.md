# Fenrir – Rodin (POCO X7 Pro / Redmi Turbo 4) Patch Update Guide

This document explains how to **manually update pattern signatures** when Xiaomi updates firmware and fenrir reports missing pattern warnings such as:

```
Warning: Pattern not found for patch 'bypass_lock_control'
Warning: Pattern not found for patch 'force_green_state'
Warning: Pattern not found for patch 'bypass_imgkey_region_check'
Warning: Pattern not found for patch 'bypass_sku_check'
```

---

## 1. Summary of Patches & Current Patterns

| Patch Name | Description | Current Pattern (Hex) | Current Replacement (Hex) |
|---|---|---|---|
| `bypass_lock_control` | Skip lock error branch in fastboot flash | `00 f8 3b 91 c3 00 00 14 e8 0f 40 b9 1f 05 00 71 21 01 00 54` | `00 f8 3b 91 c3 00 00 14 e8 0f 40 b9 1f 05 00 71 09 00 00 14` |
| `force_green_state` | Store `wzr` (0) into boot state variable | `e8 04 00 f0 00 c1 01 b9 bf 23 03 d5` | `e8 04 00 f0 1f c1 01 b9 bf 23 03 d5` |
| `bypass_imgkey_region_check` | Unconditionally jump to bypass CN region key check | `fe 0a 01 94 00 05 00 34` | `fe 0a 01 94 28 00 00 14` |
| `bypass_sku_check` | Skip SKU check jump to AVB bypass region check | `f6 03 00 aa 4f f0 fe 97 e0 00 00 34` | `f6 03 00 aa 4f f0 fe 97 7c 00 00 14` |

---

## 2. Understanding Each Patch & How to Update Them

### A. `bypass_lock_control`
- **Purpose**: Fastboot flashing checks if device lock state equals 1 (`cmp w8, #1`). If locked, it falls through to print `"Flashing is not allowed in Lock state"`. If unlocked, it takes `b.ne` to skip the error.
- **Mechanism**: The patch converts the conditional branch `b.ne` into an unconditional branch `b +0x24` (`09 00 00 14`), skipping the error block unconditionally.
- **Finding the pattern**:
  1. Search for the load/compare anchor in `lk`: `e8 0f 40 b9 1f 05 00 71` (`ldr w8, [sp, #12] ; cmp w8, #1`).
  2. The hit right before string `"Flashing is not allowed in Lock state"` has `b.ne +0x24` (`21 01 00 54`).
  3. Capture the 20 bytes starting 2 instructions before `ldr`.
  4. Replace the last 4 bytes (`21 01 00 54`) with `09 00 00 14`.

### B. `force_green_state`
- **Purpose**: Sets device boot state to Green (0) instead of Orange (1) / Red (2) so Android and TEE verify the boot integrity as locked/green.
- **Mechanism**: The bootloader function `set_boot_state(uint32_t state)` performs `adrp x8, <page>` followed by `str w0, [x8, #448] ; autiasp ; ret`. The patch changes `str w0` (opcode `00 c1 01 b9`) to `str wzr` (opcode `1f c1 01 b9`).
- **Finding the pattern**:
  1. Search in `lk` for `autiasp` (`bf 23 03 d5`) preceded by `str w0, [x8, #448]` (`00 c1 01 b9`).
  2. The preceding `adrp x8` instruction varies depending on the page offset (e.g. `e8 04 00 d0` vs `e8 04 00 f0`).
  3. Pattern: `<adrp_bytes> 00 c1 01 b9 bf 23 03 d5`.
  4. Replacement: `<adrp_bytes> 1f c1 01 b9 bf 23 03 d5`.

### C. `bypass_imgkey_region_check`
- **Purpose**: Allows booting cross-region or CN firmware by skipping the image key region check.
- **Mechanism**: Calls `strcmp(region, "yes")` followed by `cbz w0, <bypass_label>`. When not matching "yes", it errors with `"img region not match!\n"`. The patch replaces `cbz` with unconditional branch `b +0x28` (`28 00 00 14`) to jump directly to the `"lk bypass check region img key\n"` branch.
- **Finding the pattern**:
  1. Find string `"lk bypass check region img key"` in `lk`.
  2. Inspect the cross-reference: locate `bl <strcmp>` followed by `cbz w0, <target>` (`00 05 00 34`).
  3. Pattern: `<bl_strcmp_bytes> 00 05 00 34`.
  4. Replacement: `<bl_strcmp_bytes> 28 00 00 14`.

### D. `bypass_sku_check`
- **Purpose**: Bypasses the AVB SKU verification check on POCO X7 Pro / Redmi Turbo 4.
- **Mechanism**: Reads property `bypass_check_imgkey`, checks string length (`bl <strlen>`), followed by `cbz w0, <strict_check>`. If bypassed, it jumps to `[AVB] bypass check region img key\n` (target +0x1F0 bytes). The patch replaces `cbz w0` (`e0 00 00 34`) with unconditional `b +0x1f0` (`7c 00 00 14`).
- **Finding the pattern**:
  1. Find string `bypass_check_imgkey` in `lk`.
  2. Disassemble around the cross-reference: find `mov x22, x0 ; bl <strlen> ; cbz w0, <fallback>`.
  3. Pattern: `f6 03 00 aa <bl_strlen_bytes> e0 00 00 34`.
  4. Replacement: `f6 03 00 aa <bl_strlen_bytes> 7c 00 00 14`.

---

## 3. How to Build & Flash Safely Without Bricking

1. **Verify All Patches Match**:
   Always run:
   ```bash
   ./build.sh rodin
   ```
   Check the output carefully! Every patch must report `Successfully applied patch ...`. There must be **ZERO** `Warning: Pattern not found` warnings.

2. **Always Keep Stock Backup**:
   Before flashing anything, always keep an untouched copy of your original bootloader image:
   ```bash
   fastboot flash lk_a rodin-fenrir.bin
   ```
   If needed, you can always flash back the stock bootloader in fastboot mode.
