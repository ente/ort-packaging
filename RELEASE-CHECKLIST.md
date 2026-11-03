# Stable release checklist

Follow the canonical procedure in [README.md](README.md#release-process). The workflows enforce everything else, so these are the only manual checks.

Before publishing:

- [ ] Android WebGPU passed on representative supported physical devices.
- [ ] The iOS device slice and vendor-GPU behavior were validated.

Never replace or retag an existing release. Create the next package revision instead.
