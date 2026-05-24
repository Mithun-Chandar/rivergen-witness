# @rivergen/witness

Type contracts for [RiverGen](https://github.com/Mithun-Chandar/rivergen) field continuity audits.

Install alongside `@rivergen/cli`:

```bash
npm install @rivergen/witness
```

This package is automatically imported by every witness file that `rivergen gen` generates.
You do not call it directly — fill the generated `<domain>.witness.ts` and run `rivergen verify`.

## Types exported

- `DomainWitness<TPayload>` — the interface every `<domain>.witness.ts` must satisfy
- `WitnessAssertion` — `{ name, ok, detail? }` — unit returned by lifecycle/signals
- `LayerResult` — `{ ok, violations[] }` — one layer's pass/fail result
- `DomainWitnessReport` — layer1–4 results for one domain
- `WitnessResult` — top-level runner output

## License

Apache 2.0
