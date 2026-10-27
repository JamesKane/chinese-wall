# Provenance: 01-platform

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - Documentation/devicetree/bindings/display/arm,komeda.yaml: all (summarised; dual-licensed interface definition)
  - drivers/gpu/drm/arm/display/komeda/komeda_dev.c: 108-178 (DT parse: irq, memory-region, pipeline nodes, pxclk, ports, dual-link), 180-273 (create: aclk, identify, IOMMU domain), 314-342 (resume/suspend order)
  - drivers/gpu/drm/arm/display/komeda/komeda_drv.c: 56-105 (probe: 40-bit DMA mask, PM), 107-157 (match table, PM ops)
  - drivers/gpu/drm/arm/display/komeda/komeda_crtc.c: 112-201 (aclk/pxclk enable policy), 404-486 (aclk minimum, mode_valid, mode_fixup), 615-680 (bridge on endpoint 0, TMDS encoder)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline.c: 325-329 (dual-link ignored without support)
  - drivers/gpu/drm/arm/display/komeda/komeda_framebuffer.c: 202 (is_va from IOMMU presence)
  - drivers/gpu/drm/arm/display/komeda/komeda_kms.c: 248-250 (shared IRQ)
