# CDNA4 ISA Markdown

The [AMD Instinct CDNA4 Instruction Set Architecture](https://www.amd.com/content/dam/amd/en/documents/instinct-tech-docs/instruction-set-architectures/amd-instinct-cdna4-instruction-set-architecture.pdf) reference guide, converted from PDF to a single Markdown file so LLMs can read it.

Converted with [Marker](https://github.com/datalab-to/marker), then post-processed to fix heading levels, fence pseudocode blocks, strip page headers/footers, and rescue instruction names that OCR misclassified as code.

Also included is `amdgpu_isa_cdna4.xml` from [GPUOpen's machine-readable ISA](https://gpuopen.com/machine-readable-isa/). The XML has precise encoding definitions; the manual has the explanatory context. They complement each other well.

The conversion is automated and may contain OCR errors, misformatted tables, or missing content. When precision matters, verify against the original PDF.

See the original document for the license.

Inspired by [woct0rdho/rdna35-isa-markdown](https://github.com/woct0rdho/rdna35-isa-markdown).

## Navigating the XML with `xq`

The XML is 8.4 MB / 146K lines. [`xq`](https://github.com/sibprogrammer/xq) (the Go CSS-selector tool, not the Python jq-wrapper) makes it navigable. Key flags: `-n` returns XML nodes, `-q` takes a CSS selector.

Below is a worked example researching `V_MFMA_F32_16X16X128_F8F6F4`.

### 1. Find an instruction

```bash
xq -n -q 'Instruction:has(InstructionName:contains("V_MFMA_F32_16X16X128_F8F6F4"))' amdgpu_isa_cdna4.xml
```

This returns everything in one shot: name, description, flags, encoding name, opcode, operands (field, format, type, size), and functional group. For this instruction:

- **Encoding:** `VOP3P_MFMA`, opcode 45
- **VDST** (out): 128b, `FMT_NUM_PK4_F32`, `OPR_VGPR_OR_ACCVGPR` — 4×F32 accumulator/result
- **SRC0** (in): 256b, `FMT_NUM_PK8_B32`, `OPR_SRC_VGPR_OR_ACCVGPR` — matrix A (8 VGPRs of opaque data)
- **SRC1** (in): 256b, same — matrix B
- **SRC2** (in): 128b, `FMT_NUM_PK4_F32`, `OPR_SRC_VGPR_OR_ACCVGPR_OR_CONST` — accumulator C (accepts constants)
- **Group:** VALU / MFMA

Partial-match works for browsing: `InstructionName:contains("MFMA")`. Pipe through `| head -N` for large result sets.

### 2. Get the encoding bit layout

```bash
xq -n -q 'Encoding:has(EncodingName:contains("VOP3P_MFMA")) MicrocodeFormat' amdgpu_isa_cdna4.xml
```

Returns every field with bit offset and width. The VOP3P_MFMA encoding is 64 bits:

| Field | Bits | Offset | Purpose |
|-------|------|--------|---------|
| VDST | 8 | 0 | Destination register |
| CBSZ | 3 | 8 | A-matrix swizzle control |
| ABID | 4 | 11 | A-matrix swizzle block ID |
| ACC_CD | 1 | 15 | AccVGPR select for C/D |
| OP | 7 | 16 | Opcode (45) |
| ENCODING | 9 | 23 | Encoding prefix |
| SRC0 | 9 | 32 | Matrix A register |
| SRC1 | 9 | 41 | Matrix B register |
| SRC2 | 9 | 50 | Accumulator register |
| ACC | 2 | 59 | AccVGPR select for A/B |
| BLGP | 3 | 61 | B-matrix lane-group swizzle |

To get just the encoding metadata (bit count, mask, description) without the identifiers list:

```bash
xq -n -q 'Encoding:has(EncodingName:contains("VOP3P_MFMA")) > :not(EncodingIdentifiers):not(EncodingConditions):not(MicrocodeFormat)' amdgpu_isa_cdna4.xml
```

### 3. Look up data formats

Each operand references a data format. Look them up:

```bash
xq -n -q 'DataFormat:has(DataFormatName:contains("FMT_NUM_PK4_F32"))' amdgpu_isa_cdna4.xml
```

Returns data type, total bit count, component count, and per-component bit fields (sign/exponent/mantissa). The sub-element formats packed inside the opaque B32 containers:

| Format | Bits | Layout | Exponent | Mantissa |
|--------|------|--------|----------|----------|
| FP4 | 4 | S1 E2 M1 | 2 | 1 |
| FP6 | 6 | S1 E2 M3 | 2 | 3 |
| BF6 | 6 | S1 E3 M2 | 3 | 2 |
| FP8 | 8 | S1 E4 M3 | 4 | 3 |
| BF8 | 8 | S1 E5 M2 | 5 | 2 |

```bash
# Query any of them:
xq -n -q 'DataFormat:has(DataFormatName:contains("FMT_NUM_FP4"))' amdgpu_isa_cdna4.xml
```

### 4. Look up operand types

Operand types define which register classes are legal. The predefined-value lists (all 256 VGPRs, etc.) are huge, so filter them out:

```bash
xq -n -q 'OperandType:has(OperandTypeName:contains("OPR_SRC_VGPR_OR_ACCVGPR")) > :not(OperandPredefinedValues)' amdgpu_isa_cdna4.xml
```

This returns the type name and its subtypes. Drop the `:not(...)` if you need the full register enumeration.

### 5. Explore related instructions

```bash
# The SCALE variant (128-bit ENC_VOP3PX2 encoding, adds SCALE_SRC0/SCALE_SRC1)
xq -n -q 'Instruction:has(InstructionName:contains("V_MFMA_SCALE_F32_16X16X128_F8F6F4"))' amdgpu_isa_cdna4.xml

# The 32x32 sibling
xq -n -q 'Instruction:has(InstructionName:contains("V_MFMA_F32_32X32X64_F8F6F4"))' amdgpu_isa_cdna4.xml

# All MFMA instructions
xq -q 'Instruction:has(Subgroup:contains("MFMA")) InstructionName' amdgpu_isa_cdna4.xml

# All instructions in a functional group
xq -q 'Instruction:has(FunctionalGroup > Name:contains("SMEM")) InstructionName' amdgpu_isa_cdna4.xml | head -20
```

### Note on IO layout

The XML defines binary encodings, operand types, and data formats — but **not** the lane-to-matrix-element mapping (which lane holds which row/column of the matrix). For IO layout details, register-to-element mapping, and CBSZ/BLGP swizzle semantics, consult `amd-instinct-cdna4-instruction-set-architecture.md` (or the original PDF), specifically the MFMA instruction chapter.
