# CDNA4 ISA Markdown

The [AMD Instinct CDNA4 Instruction Set Architecture](https://www.amd.com/content/dam/amd/en/documents/instinct-tech-docs/instruction-set-architectures/amd-instinct-cdna4-instruction-set-architecture.pdf) reference guide, converted from PDF to a single Markdown file so LLMs can read it.

Converted with [Marker](https://github.com/datalab-to/marker), then post-processed to fix heading levels, fence pseudocode blocks, strip page headers/footers, and rescue instruction names that OCR misclassified as code.

Also included is `amdgpu_isa_cdna4.xml` from [GPUOpen's machine-readable ISA](https://gpuopen.com/machine-readable-isa/). The XML has precise encoding definitions; the manual has the explanatory context. They complement each other well.

The conversion is automated and may contain OCR errors, misformatted tables, or missing content. When precision matters, verify against the original PDF.

See the original document for the license.

Inspired by [woct0rdho/rdna35-isa-markdown](https://github.com/woct0rdho/rdna35-isa-markdown).
