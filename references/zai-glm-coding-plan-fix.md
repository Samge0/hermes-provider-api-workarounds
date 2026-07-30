# Z.AI Coding Plan Patch Transcript (v2 upgrade)

## Context
After Hermes Web UI restart, the hermes-agent source was **upgraded** (line numbers shifted significantly).
Old patches were lost. Re-applied patches to the new version.

## Key structural changes from v1 → v2

| Item | v1 (before restart) | v2 (after restart) |
|------|---------------------|-------------------|
| `build_system_prompt()` line | 470 | 549 |
| `system_prompt.py` file size | ~18KB | 31KB |
| `build_nvidia_nim_headers` line | 482 | 725 |
| auxiliary header injection points | 4 | 5 |
| New x.ai branch | absent | present in async + run_agent |

## The 5 auxiliary_client.py injection points (v2)
1. Line ~2074: `_gpf_aux` block (pool/cred first)
2. Line ~2114: `_gpf_aux2` block (pool/cred second)
3. Line ~5090: async conversion (`_to_async_client`)
4. Line ~5454: custom endpoint
5. Line ~5711: provider-specific headers

## The run_agent.py injection point
- around line 5229: `_apply_client_headers_for_base_url()`
- Add `elif` between `portal.qwen.ai` and `chatgpt.com` branches

## Base logic (unchanged)
- `provider == "zai"` gate (NO model filter)
- `build_zcode_headers()` returns `{User-Agent, X-ZCode-App-Version, X-ZCode-Agent}`
- Replace "Hermes Agent" → "ZCode" in system prompt
- Match `api.z.ai` or `open.bigmodel.cn` for header injection

## Future upgrade checklist
1. Run `grep -n` on injection point patterns to find new line numbers
2. Re-read `build_system_prompt()` end (near "return joined")
3. Check if any new provider-specific branches were added (new patterns to mimic)
4. Apply same patches with updated context strings
