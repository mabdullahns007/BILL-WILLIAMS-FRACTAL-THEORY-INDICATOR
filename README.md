# Bill Williams — Fractal Theory (Pine Script v6)

TradingView indicator and strategy implementing Bill Williams' **Fractal Theory / Five
Dimensions of Market Structure**, as taught in his talk
["The Practical Fractal: The Holy Grail to Trading"](https://www.youtube.com/watch?v=bNxt298VVOw).

Both scripts share one identical engine — the same fractal detection, balance line,
momentum/accelerator math and stop-placement rule. The indicator draws and alerts;
the strategy places orders and backtests.

| File | Type | Purpose |
| --- | --- | --- |
| [`bw-fractal-theory-indicator.pine`](bw-fractal-theory-indicator.pine) | `indicator` | Fractal markers, entry-stop levels, balance line, protective stops, breakout + magic-bullet alerts, optional five-dimension status panel |
| [`bw-fractal-theory-strategy.pine`](bw-fractal-theory-strategy.pine) | `strategy` | Resting stop-order entries, pyramiding, ratcheting fractal stop, magic-bullet exit, TradersPost JSON alert payloads |

---

## The five dimensions

1. **Fractal** (space) — the "five-fingered boogie": a minimum of five bars whose centre
   bar has a higher high than the two bars either side of it (mirror image for a down
   fractal). Entry is a **buy stop one tick above** a confirmed up fractal, or a **sell
   stop one tick below** a confirmed down fractal.
2. **Balance line** — a 13-bar smoothed moving average offset **8 bars forward**, the
   "strange attractor": where price would sit given no new information, so distance from
   it measures new information. Defaults to SMMA/RMA (the smoothing Williams' later
   Alligator lines use); SMA and EMA are selectable.
3. **Momentum** — the 5-bar minus 34-bar **simple** moving-average histogram. Williams is
   explicit that these must be simple, not exponential.
4. **Momentum direction** — whether that histogram is rising or falling.
5. **Accelerator** — the 5-bar average layer on momentum. Acceleration changes before
   momentum, which changes before price.

**Five magic bullets** — all five dimensions reversing at once. Williams' signal that a
trend is dead: the indicator shades the bar, the strategy closes the position.

**Stop placement** — the **second-most-recent opposite fractal**. Where that one sits
*closer* to price than the most recent one, use whichever of the two is furthest from
price. The stop ratchets with new fractals and never loosens.

---

## Installation

1. Open TradingView → **Pine Editor**.
2. `Open` → `New indicator` (or `New strategy`), delete the template.
3. Paste the contents of the `.pine` file.
4. **Save**, then **Add to chart**.

Both scripts declare `overlay=false`, so their home pane is the oscillator pane
(momentum + accelerator histograms). Everything that belongs on price — fractal markers,
balance line, entry-stop levels, protective stops, signals — is lifted onto the main
chart with `force_overlay=true`. That is intentional: you get one script, two panes.

Requires **Pine Script v6** (user-defined types, `force_overlay`, `input.*(active=)`).

---

## Indicator: key inputs

| Group | Input | Default | Notes |
| --- | --- | --- | --- |
| Fractals | Bars either side | `2` | The 5-bar fractal. Higher = rarer, slower to confirm |
| Fractals | Allow tied / extra bars | `on` | Source-faithful: valid while nothing in the window *exceeds* the centre bar. Off = strict pivot |
| Fractals | Entry buffer (ticks) | `1` | Williams places the order one tick beyond the fractal |
| Balance line | Length / offset / smoothing | `13` / `8` / SMMA | Signal logic compares price to the value *displayed* at the current bar |
| Momentum | Fast / slow SMA | `5` / `34` | |
| Accelerator | Formula | `AO - SMA(AO, 5)` | See [Ambiguities](#where-the-source-is-ambiguous) |
| Signals | Confirm break on bar close | `off` | Off = signal the moment price trades through the level, which is what a resting stop order actually does |
| Signals | Require balance-line agreement | `on` | Longs only above the balance line, shorts only below |
| Signals | Momentum / direction / accelerator filters | `off` | Opt-in confluence |
| Magic bullets | Dimensions that must reverse | `5` | Lower to fire on partial reversals |

Alerts: long breakout, short breakout, magic bullets bullish, magic bullets bearish, new
fractal confirmed.

---

## Strategy: how it trades

- **Entry** — a resting buy stop one tick above the most recent *unbroken* up fractal
  (sell stop below the most recent unbroken down fractal), re-placed every bar so it
  behaves like a genuine resting order rather than a next-bar market fill. Williams is
  explicit that this is by construction the worst trade location and the maximum risk on
  that entry; the point is never to be left out of a real trend.
- **Filter** — optionally require the other four dimensions to agree before the stop is
  placed. Only the balance line is required by default.
- **Add** — pyramids one unit per successive fractal breakout in the trend direction
  (`Maximum units`, default `6` — the yen case study in the talk ends holding 6 longs).
  Each fill consumes its fractal, so the next add needs a **new** one.
- **Stop** — the two-fractals-back rule above, ratcheting with new fractals. Where no
  opposite fractal sits on the protective side of price, it falls back to the nearest
  valid one, then to `ATR × multiple`.
- **Exit** — five magic bullets against the open position closes everything. An optional
  R-multiple target exists but is **off by default**: Williams runs winners and judges the
  method on money made, not win rate.
- **Reverse** — off by default; opposite-side orders are cancelled while a position is
  open. Turn on `Allow stop-and-reverse` to let an opposite breakout flip the position.

Strategy defaults: `pyramiding=20`, 1 contract per unit, `initial_capital=100000`,
`slippage=1`, zero commission, `calc_on_every_tick=false`. **Set commission and slippage
to match your instrument and broker before you trust any backtest number.**

### TradersPost alerts

The strategy emits TradersPost JSON in `alert_message` on every order. Set the ticker
override and quantity in the **TradersPost alerts** input group (leave blank to send the
chart symbol and let TradersPost size the order), then create the alert with:

```
{{strategy.order.alert_message}}
```

---

## Repainting and timing

- **Fractals** confirm `N` bars after their centre bar (2 by default) and never move once
  confirmed. Markers are drawn back at the centre bar via `offset=-fractalLen`, so a
  historical chart shows them where they belong — but they were not knowable until the
  confirming bar closed.
- **Breakout signals** fire intrabar by default, because that is what a resting stop order
  does. Enable `Confirm break on bar close` for close-only, non-intrabar signals.
- **Entry levels** are carried in from the previous bar: an order can only rest at a level
  that already existed when the bar opened.
- **Balance line** is plotted with `offset=8` (forward, as Williams draws it). All signal
  comparisons use the value displayed at the current bar — i.e. the one computed 8 bars
  ago — not a future value.

---

## Where the source is ambiguous

The talk leaves three things underspecified. Rather than hide a guess, each is called out:

1. **Accelerator formula.** The talk describes it as "a 5-bar moving average of the
   momentum oscillator"; Williams' published Accelerator is `AO - SMA(AO, 5)`. Both are
   selectable — the published form is the default.
2. **Fractal ties.** The definition allows tied or extra bars provided nothing in the
   two-bar window on either side exceeds the centre bar. That is the default
   (`Allow ties`); a strict-pivot mode is provided.
3. **Squat / green / fade / fake bar classification** is *named* but never *defined* in
   the source talk, so it is deliberately **not implemented** rather than guessed at.

One deviation from the literal rules, documented rather than silent: in choppy conditions
the literal "second-most-recent opposite fractal" stop can land on the wrong side of
price. Both scripts fall back to the nearest opposite fractal still on the protective
side, and the strategy falls back again to ATR when there is none.

---

## Disclaimer

For research and education only. **Nothing here is financial advice.** These scripts
implement one trader's published framework; they are not a validated edge, they have not
been walk-forward tested, and past backtest results do not predict future returns.
Trading leveraged instruments can lose more than your deposit. Test on paper first, size
positions yourself, and take responsibility for your own trades.

## Attribution

Method: **Bill Williams** (Profitunity). Implementation derived from his talk
"The Practical Fractal: The Holy Grail to Trading". Scaffolded with PineScript Agents by
TradersPost.

## License

MIT — see [`LICENSE`](LICENSE).
