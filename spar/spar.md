# /spar — Discuss with a sub-agent

Launch an Opus sub-agent (`model: "opus"`) to get an independent perspective
on the current topic. Describe the *situation* — deliberately minimal and
factual: the system, the symptom, the constraints. Do NOT present your
analysis, position, or preferred framing. Less context forces the sub-agent
to build its own causal model rather than absorbing yours.

Do NOT tell the sub-agent to "challenge" or play devil's advocate. Do NOT
pre-supply counterarguments or angles to consider. Ask for its genuine
assessment — specifically, the 2-3 most different ways to think about the
situation and which it finds most productive. The sub-agent should think
independently, not perform opposition.

Instruct the sub-agent to open with a one-line prediction: "I'd guess the
obvious move here is X." — then give its actual assessment. This surfaces
implicit framing and signals early when a spar won't produce novel insight.

Both sides: dense, direct, agent-to-agent register. No prose, no hedging.

Go 2-3 rounds (resume the same agent ID). In follow-up rounds, explore where
your frames diverge — test both, don't just defend yours. Concede where the
sub-agent's frame is better. Note: the first response is the most independent;
rounds converge, so don't extend beyond 3.

After the discussion, synthesise for the user: what each frame was, where they
diverged, and what that divergence reveals.

The user's arguments (optional focus): $ARGUMENTS
