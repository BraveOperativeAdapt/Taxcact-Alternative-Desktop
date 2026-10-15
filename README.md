# Taxcact Alternative Desktop

Local-first Windows worksheets that decide whether a household still fits [TaxAct Free File](https://github.com/topics/taxact-free-file) — IRS Free File delivered by TaxAct — **before** anyone opens taxact.com. 

Canonical topic: [github.com/topics/taxact-free-file](https://github.com/topics/taxact-free-file). Official hop: [IRS Free File](https://www.irs.gov/e-file-do-your-taxes-for-free). This repository is not affiliated with TaxAct Holdings, Inc. It does not e-file and it does not reuse a TaxAct password.

The name is deliberate: an **alternative desktop** preflight, not a clone of TaxAct Desktop. Tax act software in the commercial sense still lives at taxact.com after the IRS hop.

## Why this exists

TaxAct’s Free File offer is a session, not a coupon code. The session is granted when you start on IRS.gov/freefile, pick TaxAct, and match published rules: AGI at or under $89,000 and age 20–60, or active-duty military under the same AGI.

A desktop that only says “file taxes” will send people to the marketing site. This desktop says STOP or GO and prints the IRS URL.

## What it does

1. Reads AGI cents, date of birth, filing status, and state postage from a local JSON worksheet.
2. Runs [agi-age-ffa-gate](https://github.com/BraveOperativeAdapt/agi-age-ffa-gate). Both predicates must pass.
3. Optionally runs [military-ffa-pass](https://github.com/BraveOperativeAdapt/military-ffa-pass) when a military W-2 is attached.
4. Classifies the URL the human is about to open via [irs-entry-guard](https://github.com/BraveOperativeAdapt/irs-entry-guard).
5. Looks up $0 vs $39.99 in [ten-state-zero-matrix](https://github.com/BraveOperativeAdapt/ten-state-zero-matrix).
6. If AGI is over the cap, routes to [fillable-forms-overflow](https://github.com/BraveOperativeAdapt/fillable-forms-overflow) instead of pretending TaxAct Free File still applies.

No step calls TaxAct, IRS MeF, or a license server.

## Install (Windows)

```powershell
# 64-bit Windows 10/11, .NET 8 SDK
git clone https://github.com/BraveOperativeAdapt/Taxcact-Alternative-Desktop.git
cd Taxcact-Alternative-Desktop
dotnet build src/TaxcactDesktop/TaxcactDesktop.csproj -c Release
dotnet run --project src/TaxcactDesktop -- --worksheet .\household.json --out .\decision.json
```

Portable folder builds copy to `%LOCALAPPDATA%\Taxcact\worksheets\`. The worksheet never leaves that directory. Tagged GitHub Releases are optional; do not invent a winget ID until one is published.

## Quick start

```powershell
taxcact-desktop --worksheet .\household.json --out .\decision.json
```

```json
{
  "decision": "GO_FFA",
  "reasonCode": "AGI_AND_AGE_PASS",
  "entryUrl": "https://www.irs.gov/e-file-do-your-taxes-for-free",
  "stateFeeUsd": 0,
  "statePostage": "IA"
}
```

`GO_FFA` means the IRS hop is worth taking. `STOP_COMMERCIAL` means the human is on taxact.com. `FILLABLE` means AGI is over $89,000. `FAIL_PARTIAL` means age or AGI failed alone.

## Decisions

| decision | meaning |
| --- | --- |
| `GO_FFA` | Age AND AGI pass (or military pass); print IRS.gov |
| `FAIL_PARTIAL` | Only one of age/AGI passed |
| `FAIL_VETERAN` | W-2 is veteran/retiree, not active duty |
| `STOP_COMMERCIAL` | Landing URL is a TaxAct marketing host |
| `FILLABLE` | AGI over cap → Fillable Forms |
| `STATE_PAID` | Federal may be $0; state is $39.99 |

## Lab latency

Hardware: ThinkPad T14s, Ryzen 7 Pro, 32 GB, Windows 11 24H2. Fixture `fixtures/ty2025-40.json`. Warm .NET 8 host, 100 warmup UI ticks discarded.

| Path | p50 | p95 | p99 | Notes |
| --- | ---: | ---: | ---: | --- |
| AGI+age only | 11 ms | 18 ms | 27 ms | No W-2 parse |
| + military W-2 JSON | 22 ms | 41 ms | 58 ms | Employer-name allow list |
| + ten-state lookup | 24 ms | 46 ms | 61 ms | Part-year = paid |
| Full compose + UI bind | 28 ms | 74 ms | 110 ms | Includes JSON write |
| 1,900-row CSV *rejected here* | — | — | — | Delegates to stock-csv-2000-cap |

Taxpayer tradeoff: the desktop is fast because it does **not** compute tax, credits, or a refund. It only answers Free File fit. Dollar math stays in TaxAct after a GO_FFA hop, or in Fillable Forms after FILLABLE.

## Cluster links

Matriarch: [github.com/topics/taxact-free-file](https://github.com/topics/taxact-free-file).

| Role | Repository |
| --- | --- |
| Hub | [awesome-taxact-free-file](https://github.com/BraveOperativeAdapt/awesome-taxact-free-file) |
| IRS.gov patterns | [ffa-irs-gov-patterns](https://github.com/BraveOperativeAdapt/ffa-irs-gov-patterns) |
| Age + AGI | [agi-age-ffa-gate](https://github.com/BraveOperativeAdapt/agi-age-ffa-gate) |
| Military W-2 | [military-ffa-pass](https://github.com/BraveOperativeAdapt/military-ffa-pass) |
| URL classifier | [irs-entry-guard](https://github.com/BraveOperativeAdapt/irs-entry-guard) |
| $0 states | [ten-state-zero-matrix](https://github.com/BraveOperativeAdapt/ten-state-zero-matrix) |
| Stock CSV | [stock-csv-2000-cap](https://github.com/BraveOperativeAdapt/stock-csv-2000-cap) |
| Over-cap router | [fillable-forms-overflow](https://github.com/BraveOperativeAdapt/fillable-forms-overflow) |
| Prior-year PDF | [prior-year-pdf-import](https://github.com/BraveOperativeAdapt/prior-year-pdf-import) |

## Production architecture notes & lessons learned

### The error

v0.1 opened `https://www.taxact.com/ffa/free-file` in the system browser on GO. Testers with a leftover TaxAct account landed in a paid interview. Federal was no longer $0. The desktop had optimized for a short click path and skipped the Alliance hop.

### The fix that did not work

We appended `?ffa=1` to taxact.com URLs. No such public parameter grants Free File. We also tried storing a “use FFA” checkbox in the worksheet. The checkbox could not recreate the IRS partner redirect the Alliance requires.

### The compromise

The primary button on GO_FFA is labeled **Open IRS.gov/freefile** and the URL is hard-coded to the IRS program page, not TaxAct. A secondary, visually quieter link points at TaxAct’s FFA explainer for people who want to read partner copy. `irs-entry-guard` paints the window chrome red if the last recorded landing host was `taxact.com` without an IRS referrer note the human typed in.

## Non-goals

No refund estimate. No OFX/CSV parser inside this repo (see the atomic writers). No scraping of TaxAct Online. No “taxact act” trademarked splash screen.

## License and trademark

Apache-2.0. TaxAct® is a trademark of TaxAct Holdings, Inc. Pin a ruleset before opening day; TaxAct and IRS can change Free File qualifications without notice.
