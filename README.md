# Strategic AI Agents in a Simulated Courtroom

**CS F407 Artificial Intelligence, BITS Pilani, Hyderabad Campus**
**Team 22, Saul Goodman** | Track: **Demo** | Topic: **Agents & Game Theory** (the judge also uses rule-based reasoning)

Two rational agents, a prosecution and a defense, play a budgeted game over structured, synthetic evidence. They decide what to present, what to object to and when to rest. A deterministic, rule-based judge rules on every objection and delivers a verdict, naming the rule behind each decision. A game-theory layer then plays the strategies against each other, builds payoff matrices and solves them for Nash equilibria. The whole trial is visualised live.

> **Status:** project in progress. Sections marked *(planned)* describe the intended design from our proposal; they are not results. This README is updated as each package lands.

> Every case, person and organisation in this repository is **invented**. Nothing models real court procedure, real law or a guilt predictor. All thresholds are game parameters, not legal probabilities.

## Project question

Against a fixed and fully transparent judge, how do budgets, penalties and noisy perception decide whether an aggressive or a conservative strategy wins, and for which cases does the game have a dominant strategy rather than a mixed equilibrium?

## What the Demo track requires, and where we meet it

| Requirement | How |
| --- | --- |
| Deterministic, well-defined verdict rules | The judge is a pure function of the admitted record, uses exact `Fraction` arithmetic, has a written tie rule, and every ruling cites a rule ID. See `docs/judge_rules.md` *(planned)*. |
| Robust to weak, missing and contradictory evidence | Six edge fixtures with the expected verdict fixed in advance, a Chaos agent, and 2,500 fuzzed and replayed trials. |
| Synthetic cases only | Three invented cases, a seeded generator, six edge fixtures. No real data. |

## Course concepts used

Rational agents and expected utility, Bayes' rule under noisy signals (Beta-Bernoulli opponent model), game formulation as a finite state machine, normal-form games with pure and mixed Nash equilibria (support enumeration, dominance elimination, best-response check), empirical game-theoretic analysis, replicator dynamics, parameter design, and rule-based reasoning for the judge.

## Architecture

```
viz/        terminal dashboard and browser UI (reads TrialResult and PayoffTable only)
engine/ api/ cli     runs a trial, catches agent faults, hash-chained event log, FastAPI, replay and verify
gametheory/ payoff Monte Carlo, Nash solver, bootstrap, replicator dynamics
cases/      case library, edge fixtures, seeded generator
procedure/  phase state machine, action validator, ledgers, turn caps
judge/      admissibility, exact scoring, directed verdict, rule trace
agents/     perception, read-only Oracle, strategies
contracts/  frozen Pydantic v2 models: CaseFile, EvidenceItem, Action, TrialEvent, TrialResult
```

Imports point downward. The judge never imports the agents; agents see the judge only through a read-only Oracle that scores "what if this item were admitted?" from the public record and their own beliefs, never the true defect flags.

**Stack:** Python 3.11, Pydantic v2, FastAPI, Rich (terminal UI), plain JavaScript (browser UI). No model training and no paid API.

## How a trial runs

Opening statements, prosecution case, directed-verdict review, defense case, rebuttal, deliberation, closed. After every `PRESENT_EVIDENCE` the opponent chooses `OBJECT(ground)` or `PASS`; the judge rules and the window closes. Every phase has a turn cap and a global event cap, so every trial ends.

## The judge in brief

For each legal element `e`, over admitted items only:

```
S_i = r_i * kappa_i
d_i = min(9/10, sum over contradicting j of (4/5) * S_j / (S_i + S_j))
c_i = 1 + (1/2) * (1 - 2^(-k_i))
v_ie = s_ie * S_i * (1 - d_i) * c_i
L_e  = pi + 3 * sum over facts f of ( max positive v_ie - max negative v_ie )
plaintiff wins  <=>  L_e > theta for every element e
```

`L_e` is in log-odds. Standards `theta`: 11/5 (beyond reasonable doubt, about 0.90), 11/10 (clear and convincing, about 0.75), 0 (preponderance). The prior `pi` is -1 for criminal and 0 for civil cases. An exact tie goes to the defendant. A directed verdict is granted if any element is already at or below its standard when the prosecution rests (DV-1), and always when nothing is admitted (DV-2). Full definitions and rule IDs are in the proposal and in `docs/judge_rules.md` *(planned)*.

## Strategies

| Strategy | Behaviour |
| --- | --- |
| Aggressive | Presents even known-defective items (up to two), objects at belief >= 0.35, keeps no reserve |
| Conservative | Presents only clean items, objects at belief >= 0.70 and impact >= 0.25 log-odds, keeps 25% reserve |
| Adaptive | Best expected-utility action under current beliefs, learns the opponent's objection rate |
| Mixed(p) | Per decision, Aggressive with probability p, else Conservative |
| Chaos | Random and often illegal; robustness testing only, never in the payoff matrix |

## Cases and edge fixtures

Three full synthetic cases (cyber-espionage prosecution, autonomous-vehicle liability suit, DeFi exploit dispute) and six edge fixtures: empty docket, two equally credible contradictory witnesses, every exhibit defective, defense-only evidence, malformed case file, and illegal or out-of-turn actions. Each fixture's expected outcome is written down and checked by a test.

## Evaluation plan

| Check | Done when |
| --- | --- |
| Correctness: hand-computed judge fixtures, property tests, solver on Prisoner's Dilemma, Bach or Stravinsky, matching pennies | All fixtures match and every known equilibrium is found exactly |
| Robustness: six edge fixtures plus 100 generated cases x 25 pairings = 2,500 trials, each replayed | Expected outcomes hold, no crash or broken invariant, replay hashes identical |
| Analysis: 4x4 payoff matrices over 200 common seeds per cell, 200 bootstrap resamples | Each case labelled pure, mixed or dominance-solvable with a stability figure |
| Sensitivity: sanctions, costs, scale and standards varied by +/-25% | We report which conclusions survive |
| Live demo: any case, pairing and seed can be stepped and replayed; examiners can load a case | A trial runs in under 1 s and all edge fixtures run cleanly offline |

## Getting started *(planned; commands will be filled in as the packages land)*

```bash
git clone https://github.com/BinaryCoder3012/T22_SAUL_GOODMAN.git
cd T22_SAUL_GOODMAN
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# run one trial and print the judge trace
python -m cli run --case cases/library/<case>.json --pros aggressive --def conservative --seed 1

# replay and verify a recorded trial
python -m cli replay <trial_file> --verify

# payoff matrix and equilibrium analysis
python -m cli analyse --case cases/library/<case>.json --seeds 200

# live demo (browser) and tests
python -m api            # then open the printed local URL
pytest
```

Everything is seeded: the same case, strategy pair and seed always give the same trial and the same event-log hash.

## Timeline

Six weeks from 3 Oct to 13 Nov 2026 (the official deadline is still to be announced).

| Gate | End of | Goal |
| --- | --- | --- |
| G1 | Week 2 | One case runs end to end |
| G2 | Week 3 | Integrated demo on all three cases |
| G3 | Week 5 | Parameters frozen, results reproduced |

If a gate slips we cut in this order: replicator dynamics, stretch expectimax agent, hash-chained log (plain seeded replay instead), then the web UI (the terminal replay stays). The core demo, judge and 4x4 analysis are never cut.

## Team

| Member | Student ID | Package |
| --- | --- | --- |
| Divvij Yogesh Chichra | 2024AAPS0298H | `procedure/` trial procedure and objection rules |
| Lakshya Sachdeva | 2024A8PS0468H | `judge/` deterministic judge and scoring |
| Mofasser Arafat Midda | 2023B1A80965H | `agents/` strategic agents and perception model |
| Pramit Khandelwal | 2024A3PS0443H | `engine/`, `api/`, `cli` trial engine, audit trail, API |
| Rakshit Goel | 2023A3PS0373H | `viz/` live visualisation |
| Aditya Singh | 2024A8PS0491H | `contracts/`, `cases/` data contracts and case library |
| Rohan Baban Pagire | 2024A7PS1160H | `gametheory/` payoff Monte Carlo, Nash solver, analysis |

Each member owns one package, writes its tests, defends it in the viva and reviews the next person's pull requests in a ring.

## Contributing workflow

Feature branches, pull requests with one reviewer from the ring, and **weekly commits from every member** so the history reflects steady work across the semester. Progress notes go in `docs/PROGRESS.md` *(planned)*.

## AI-usage disclosure

The course allows AI tools but requires disclosure and verification. We record each use in [`AI_USAGE.md`](AI_USAGE.md): the tool, what it was used for, and how we verified the output. Every member must be able to explain any code, math or writing they are associated with. *(Fill this in as the project proceeds; a README line is not a substitute for the log.)*

## Limitations

This is a stylised game, not a model of real courts. Equilibria are estimated from Monte Carlo payoff matrices, so labels carry sampling noise (we report bootstrap stability). Scores depend on game parameters we chose and vary in a sensitivity sweep.

## References

1. S. Russell and P. Norvig. *Artificial Intelligence: A Modern Approach*, 4th ed., Pearson, 2021.
2. Y. Shoham and K. Leyton-Brown. *Multiagent Systems*. Cambridge University Press, 2008.
3. J. Nash. Non-cooperative games. *Annals of Mathematics*, 54(2), 1951.
4. R. Porter, E. Nudelman and Y. Shoham. Simple search methods for finding a Nash equilibrium. *Games and Economic Behavior*, 63(2), 2008.
5. M. P. Wellman. Methods for empirical game-theoretic analysis. *AAAI 2006*.
6. P. Taylor and L. Jonker. Evolutionarily stable strategies and game dynamics. *Mathematical Biosciences*, 40, 1978.
7. Z. He et al. AgentsCourt. *Findings of EMNLP 2024*. doi:10.18653/v1/2024.findings-emnlp.549
8. P. D. Siedler. Strategic persuasion with trait-conditioned multi-agent systems for iterative legal argumentation. arXiv:2604.07028, 2026.
9. L. Zhang and K. D. Ashley. Mitigating manipulation and enhancing persuasion. arXiv:2506.02992, 2025.

## License and course note

Academic coursework for CS F407. Do not reuse without permission from the team.
