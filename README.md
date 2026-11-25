# orderflow-lm

[![tests](https://github.com/daltonoscar0/orderflow-lm/actions/workflows/tests.yml/badge.svg)](https://github.com/daltonoscar0/orderflow-lm/actions/workflows/tests.yml)

Language-model analysis toolkit pointed at limit order book message
streams. Market events are treated as a discrete symbol system: parse the
raw feed, reconstruct the book, tokenize events into a factored vocabulary,
and emit a binary token stream for autoregressive training (miniGPT,
separate repo). Later phases analyze the trained model with surprisal and
probing methods. `SPEC.md` is the source of truth for goals, tokenization,
and the data-sourcing decisions summarized below.

C++17, CMake, no dependencies beyond the standard library for pipeline
code. Every stage is a standalone binary that reads stdin-or-file and
writes a well-defined format.

## Pipeline

| stage | code | status |
|-------|------|--------|
| LOBSTER message parser | `src/parser/` | done |
| Order book reconstruction | `src/book/` | done, validated |
| Factored tokenizer `[TYPE][SIDE][PRICE_OFF][SIZE][DT]` | `src/tokenizer/` | done (v2, level-indexed price) |
| BPE over the factored stream | `src/bpe/` | next |
| Corpus statistics (Zipf, bigrams) | `analysis/` | next |

## Build and test

```sh
cmake -B build && cmake --build build -j
ctest --test-dir build
```

## Data

Free LOBSTER academic samples, 2012-06-21, 09:30-10:30, 50 levels, for
AAPL, MSFT, and SPY. Fetch them (with confirmation; orderbook CSVs total
~1.7 GB) into `data/lobster/`:

```sh
scripts/fetch_lobster_sample.sh          # or --mirror for the HF mirror
```

Do not add the 30-level samples: they cover the identical ticker-day-window
at lower depth, so pooling duplicates events and leaks across train/eval
splits (SPEC Decision B).

## Reproduce the token corpus

```sh
# 1. Validate book reconstruction against LOBSTER's orderbook file.
build/validate_book data/lobster/MSFT_2012-06-21_34200000_37800000_message_50.csv \
                    data/lobster/MSFT_2012-06-21_34200000_37800000_orderbook_50.csv

# 2. Fit per-ticker size/dt bins on the training split only (frozen in the
#    manifest with the split boundary, so eval data can never leak in).
build/tokenizer_fit data/lobster/SPY_2012-06-21_34200000_37800000_message_50.csv \
                    --ticker SPY --manifest data/tokens/manifest.json --train-frac 0.8

# 3. Tokenize (pure application of the manifest; this tool cannot fit).
build/tokenize data/lobster/SPY_2012-06-21_34200000_37800000_message_50.csv \
               data/lobster/SPY_2012-06-21_34200000_37800000_orderbook_50.csv \
               --ticker SPY --manifest data/tokens/manifest.json \
               -o data/tokens/SPY_50.tokens.bin

# 4. Round-trip validation: tokens -> approximate event -> same tokens.
build/token_roundtrip data/tokens/SPY_50.tokens.bin data/tokens/manifest.json
```

Repeat 2-4 for AAPL and MSFT.

## Tokenization (v2)

Each event becomes five tokens from a 52-id vocabulary: type (6), side (2),
price offset (13), size (8 per-ticker quantile bins), and inter-event time
(log-bucketed, 14 buckets plus zero and tail). PRICE_OFF is the signed
occupied-level index on the event's side of the pre-event book: 0 = at the
same-side best, +k = k occupied levels strictly better than the price,
−1 = inside the spread, window −1..+10 plus one positive tail. The negative
side is a single bucket because, measured across all three tickers, every
inside-spread event sits at −1, none reach or cross the opposite best.

The training corpus is SPY only (1.15M events, ~5.8M tokens); there is no
ticker token. AAPL and MSFT are held-out generalization checks
(SPEC Decision C).

## Data-quality findings

LOBSTER message files only report events inside the sample's requested
price window, so a book reconstructed from messages alone drifts from the
companion orderbook file, worst at the window edge, catastrophic for
narrow windows (the 10-level full-day AAPL file is 99.2% wrong at level 1).
Consequences, measured and specced in `SPEC.md`:

- Book-state features (price offsets, spread, imbalance) come from the
  orderbook file, not our reconstruction (Decision A). Reconstruction is
  retained for ITCH, where no orderbook file exists.
- Censoring severity is ticker-dependent on the identical window: MSFT's
  top 11 levels are exactly clean (first mismatch anywhere is at level 12),
  AAPL is mildly contaminated, SPY is ~10% wrong by depth 10. Safe depth
  must be measured per ticker.
- Phantom reductions (deletes/cancels against orders the message stream
  never showed us) on the 50-level hour samples: MSFT 5, SPY 36, AAPL 41 -
  versus 5,423 on the 10-level full-day file.
- AAPL's depth distribution reaches level 49, the deepest a 50-level file
  can express; its deep tail is truncated by the file's depth and is
  unobservable in this data.

## Known v1 limitations

- Order reference identity is dropped: cancels/executions cannot be linked
  to the add they consume.
- The model only ever observes events inside the sample's price window;
  surprisal and probing conclusions are bounded accordingly.

## License

MIT. See [LICENSE](LICENSE).
