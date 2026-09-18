# Traps — geo-aux-heat-monitor

## The repo copies silently drift from canonical (2026-09-18)

`docs/adr/sln-adr-013-…md` and `esphome/sln-geo-aux-heat.yaml` in THIS repo are
copies. The README says they are kept byte-identical to `sln-ha-config`. They
were not. Found 2026-09-18: the ADR copy was **215 lines against canonical's
450** — missing the entire 2026-08-10 BUILT AND BENCH-VALIDATED amendment — and
the YAML copy still carried the **pre-erratum pin map**.

**What it cost:** answering a live hardware question from the copy gave the
as-built pins as GPIO25/26/27. The board actually uses **GPIO32/33** (optos) and
**GPIO34** (CT); 25/26/27 were the *design intent* that the pin-number/pin-name
erratum invalidated. The wrong pins were stated out loud before canonical was
read.

**Rule:** read `sln-ha-config` for anything as-built. Treat this repo's copies as
possibly-stale until `diff`'d. Both were re-synced 2026-09-18.

**Cheap check before trusting either copy:**

```
diff ~/code/sln-ha-config/esphome/sln-geo-aux-heat.yaml \
     ~/code/geo-aux-heat-monitor/esphome/sln-geo-aux-heat.yaml
```

A byte-identical copy with no drift detector is a copy that will drift. If this
recurs, the fix is a symlink or a CI diff, not more discipline.
