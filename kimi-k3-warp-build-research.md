# Running a ~3T-parameter model (Kimi K3) on AM4 / AM5 / Threadripper with WARP

Researched 2026-10-23. Figures marked **(est.)** are my own estimates, not measurements or quoted prices.

## 1. What I could and could not verify

| Item | Status |
|---|---|
| "Kimi K3" | Verified by several sources: 2.8T total params (WARP says 2.78T), MoE with 16 of 896 experts per token, ~104B active, 1M context, API launched 2026-07-16, weights published ~2026-07-27. Sources disagree slightly on active params (104B vs ~50B) and on license. |
| "Kimi K3.1 Pro" | **Not found.** WARP lists K3, DeepSeek-V4.1-Flash, GLM-5.3-Flash and Kimi-Linear. I assume you mean K3. |
| "warp" | Found: `sqliteai/warp` (Weight-Aware Runtime and Paging, formerly WASTE). C, dependency-free, CPU-only, streams MoE experts from NVMe. README says it is LLM-written; all numbers below are the project's own claims, tested only on a 64 GB MacBook Pro (M5 Pro). |

## 2. How WARP behaves (from its README)

- Container: 982 GB = 962.83 GB experts (3-bit VQ) + 27.28 GB resident trunk. Needs ~1 TB of NVMe.
- RAM floor 29 GB at 4K context; 64 GB recommended. KV cache is tiny (0.21 GB at 4K).
- A cold token reads ~17 GB of experts; long runs average ~23 GB/token (4.6 TB for 200 tokens).
- Measured 0.45-0.62 tok/s with a 12.78 GB/s internal SSD.
- Expert cache barely matters: 3.3 GB cache costs only ~10% vs 17 GB. A bigger cache that causes paging was ~8x slower.
- Fewer experts per token is faster at a quality cost: top-8 gives 0.89 tok/s (KL 0.037), top-4 gives 1.06 tok/s.
- Multi-drive: `WASTE_BANK_SHARDS` exists, but the README claims **no measured speedup** (never tested on matched drives). RAID is not discussed.

## 3. The key finding: decode speed is SSD-bandwidth-bound

12.78 GB/s ÷ 0.60 tok/s ≈ 21 GB read per token, which matches the README's ~17-23 GB/token. So:

**tok/s ≈ aggregate NVMe read bandwidth ÷ ~21 GB (est.)**

Consequences:
- RAM capacity and RAM bandwidth are nearly irrelevant above ~64 GB. DDR5 vs DDR4 and quad-channel Threadripper memory do not help here.
- What matters is how many NVMe drives the platform can feed at full speed (PCIe lanes), and drive speed.
- Caveat: once I/O is fast enough, CPU compute will become the limit. WARP has not been measured past ~13 GB/s, so projections above ~2 tok/s are speculative.

## 4. Platform lane budgets

| Platform | Usable for NVMe | Notes |
|---|---|---|
| AM4 (Ryzen 5000, PCIe 4.0) | x16 slot can bifurcate to 4x x4 on many boards, plus 1-2 M.2 | ~6.5 GB/s per drive. DDR4 max 128 GB. |
| AM5 (Ryzen 9000, PCIe 5.0) | x16 slot bifurcated 4x4x4x4 plus CPU M.2 slots | ~13 GB/s per drive. Board bifurcation support must be checked per model (forum reports say most support it; not verified board-by-board). |
| Threadripper 9000 / TRX50 | Up to 80 PCIe 5.0 lanes; boards with 2-4 x16 Gen5 slots and 4 M.2 | Lanes for 8+ Gen5 drives. Quad-channel DDR5 RDIMM is wasted money for this workload. |

## 5. Builds, estimated speed and cost

Price inputs found (Oct 2026): DDR5 128 GB kits $1,800-2,800 (new; trackers show up to $3,400); 64 GB DDR4 ~$400-700; 2 TB Gen4 NVMe ~$250-400; 2 TB Gen5 NVMe ~$330-395; Threadripper 9960X $1,499; TRX50 boards $580-1,000+. Prices are volatile because of the DRAM/NAND shortage.
Prices I could not source and assumed **(est.)**: 5950X ~$450, AM4 board ~$200, 9950X ~$450, X870E board ~$350, 64 GB DDR5 ~$1,000, 64 GB DDR5 RDIMM ~$1,400, PSU/case/cooler $250-400, bifurcation riser $50-60.
RAID0 is assumed to scale at ~90% of the sum of drive speeds (est.). Use Linux md RAID0 (or LVM striping), not WARP sharding, until WARP sharding is shown to help.

| Build | Drives | Agg. BW (est.) | tok/s (est.) | Cost (est.) | $ per tok/s |
|---|---|---|---|---|---|
| **AM4-A** 5950X, 64 GB DDR4 | 1x 2 TB Gen4 | ~6.5 GB/s | ~0.3 | ~$1,650 | ~$5,300 |
| **AM4-B** 5950X, 64 GB DDR4 | 3x 2 TB Gen4 RAID0 | ~19 GB/s | ~0.9 | ~$2,300 | ~$2,600 |
| **AM5-A** 9950X, 64 GB DDR5 | 1x 2 TB Gen5 | ~13 GB/s | ~0.6 | ~$2,500 | ~$4,100 |
| **AM5-B** 9950X, 64 GB DDR5 | 3x 2 TB Gen5 RAID0 | ~36 GB/s | ~1.7 | ~$3,350 | ~$2,000 |
| **TR-A** 9960X, 64 GB RDIMM | 4x 2 TB Gen5 RAID0 | ~48 GB/s | ~2.2 | ~$5,500 | ~$2,500 |
| **TR-B** 9960X, 64 GB RDIMM | 8x 2 TB Gen5 RAID0 | ~90 GB/s | up to ~4 (speculative, CPU likely binds) | ~$7,300 | ~$1,700 (best case only) |

Optional speed-up on any build: `num_experts_per_token=8` gives ~1.5x for a small quality loss (WARP's own measurement).

## 6. Conclusions

1. **Best proven cost per token: AM5-B** (9950X, 64 GB DDR5, 3 Gen5 drives in RAID0). It is the cheapest build that clearly beats ~1.5 tok/s.
2. **Cheapest to get running: AM4-B**, about 40% cheaper than AM5-B for about half the speed. Its cost per tok/s is within error of AM5-B, so it is a fair budget pick, especially while DDR5 is so expensive.
3. **Threadripper only wins if you scale to 8+ drives** and the CPU is not the bottleneck. That is untested. Its quad-channel memory does not help WARP. Spend the Threadripper premium only if you also want GPUs or other workloads.
4. **Do not buy more than 64 GB RAM** for WARP; at current prices it is the worst-value part of every build.
5. All of these are single-user interactive speeds (about 0.3-2 tok/s). Prompt processing is also costly (~23 GB read per token), so long prompts will be slow.

## 7. Open risks and what to do before buying

- Nobody has published WARP numbers on x86 AMD, on RAID0, or above ~13 GB/s. Run WARP on a small model (for example Kimi-Linear 48B, 19 GB container) on the cheapest AM4/AM5 box you can borrow, then check how throughput scales with stripe width.
- Confirm board bifurcation (x16 to 4x4) and a riser or carrier card per specific board before buying.
- Gen5 drives throttle thermally under sustained reads; plan heatsinks and airflow.
- Prices above are snapshots from aggregator pages, several of which disagree; re-quote before purchase.
- WARP is new, LLM-written and unreviewed; treat its accuracy and quality claims as unverified.

## Sources

- Kimi K3: https://www.kimi.ai/blog/kimi-k3, https://www.labellerr.com/blog/kimi-k3-world-first-open-2-8t-ai-model/, https://www.yottalabs.ai/post/kimi-k3-specs-benchmarks-how-to-access-2026
- WARP: https://github.com/sqliteai/warp
- Related streaming work: https://github.com/ggml-org/llama.cpp/pull/25294, https://github.com/ek15072809/Swap-MoE
- RAM prices: https://hwbusters.com/news/ddr5-memory-prices-are-up-500-in-a-year-and-a-128gb-kit-now-lists-at-3399/, https://pangoly.com/en/price-trends/ram/64gb-ddr4
- Threadripper: https://www.amd.com/en/blogs/2025/designed-to-create-built-to-inspire-amd-introduces-new.html, https://tensorrigs.com/blog/best-motherboard-ai-workstation/
- SSD prices: https://dropreference.com/en/blog/component/ssd-nvme-2-tb, https://www.samsung.com/us/memory-storage/nvme-ssd/9100-pro-nvme-ssd-sku-mz-vap4t0b-am/
- AM5 lanes / bifurcation: https://forum.level1techs.com/t/am5-boards-seem-to-support-4444-bifurication/190101
