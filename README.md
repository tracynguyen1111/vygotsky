Soca's ZPD Learning Algorithm

1. Who is Vygotky
Lev Vygotsky (1896–1934) was a Soviet psychologist who did most of his famous work in about ten years, then died of tuberculosis at 37. He didn’t start in psychology. He studied law and literature, then wandered into psychology and mostly rewrote it.
His problem was IQ tests. They measured what a kid could do alone, and that number was treated as the whole story. Vygotsky noticed the flaw. Take two kids with the same score and give both a hint. One jumps way ahead, the other barely moves. Same score, very different kids. The test missed the part that matters: what you can do with help.
He called that gap the Zone of Proximal Development (ZPD). “Proximal” just means nearby: not what you can already do, not what’s impossible, but the next thing.
Bad timing, by the way. He died young, his work was banned in the Soviet Union soon after, and the West didn’t really read it until decades later.
Soca is built on his idea. Along with many other ideas we have in mind. Our tutor never hands over the answer. It asks the question that closes the gap. The algorithm’s job is to pick problems that sit inside that gap. We model this as a control problem. Think of it as a thermostat for difficulty:

State: the learner’s recent accuracy.
Control: the difficulty of the next question.
Target: accuracy inside a fixed band, [0.70, 0.85].

2. Notation
Symbol	Meaning
u	a learner
s	a skill (e.g. SAT.RW.1.1)
d_i ∈ {1,2,3,4,5}	difficulty of the i-th question attempted
c_i ∈ {0,1}	correctness of the i-th attempt (1 = correct)
W	window size (number of most recent attempts), currently W = 20
a	windowed accuracy
d̄	windowed mean difficulty
L = 0.70, H = 0.85	lower and upper accuracy thresholds
3. Zone classification

For the last W attempts, ordered from oldest to newest, indexed i = n−W+1, …, n:

a = (1/W) · Σ c_i
d̄ = (1/W) · Σ d_i

The zone is a function of a:

          ⎧ INSUFFICIENT_DATA   if n < W
zone(a) = ⎨ TOO_HARD            if a < L
          ⎪ OPTIMAL             if L ≤ a ≤ H
          ⎩ TOO_EASY            if a > H
<!-- TODO: confirm minimum n before leaving INSUFFICIENT_DATA (n < W is assumed here). -->
4. Difficulty update rule

Define the step δ by zone:

δ(zone) = −1   if TOO_HARD
           0   if OPTIMAL
          +1   if TOO_EASY

The next question's target difficulty is:

d_next = clamp( round(d̄ + δ), 1, 5 )

where clamp(x, lo, hi) = min(max(x, lo), hi).

<!-- TODO: confirm rounding (round / floor / ceil) when d̄ is not an integer, and whether the rule is per-skill or global. -->
Worked example

A learner's last 20 attempts have mean difficulty d̄ = 3.0.

Correct out of 20	a	Zone	δ	d_next
13	0.65	TOO_HARD	−1	2
15	0.75	OPTIMAL	0	3
18	0.90	TOO_EASY	+1	4
5. Properties
Bounded. d_next ∈ [1, 5] always, due to the clamp.
Dead band. The OPTIMAL zone has width H − L = 0.15. Inside it, δ = 0, so difficulty doesn't oscillate on small fluctuations in accuracy.
Memory. Each attempt influences the next W decisions and then drops out. Smaller W reacts faster but is noisier. The standard error of a for a learner with true success rate p is √(p(1−p)/W); at p = 0.78, W = 20 that is about 0.093, so the band is only about 1.6 standard errors wide. This is the main reason W is a tuning knob worth revisiting.
Monotone. For fixed d̄, higher accuracy never produces a lower d_next.
6. Why 70–85%? (motivation, not implemented)

Suppose a learner has ability θ and a question has difficulty b on a common scale, with success probability under a logistic (1PL / Rasch) model:

P(correct) = σ(θ − b),   σ(x) = 1 / (1 + e^(−x))

Inverting for the band edges:

θ − b = ln( p / (1 − p) )

p = 0.70  →  θ − b = ln(7/3)   ≈ 0.85
p = 0.85  →  θ − b = ln(17/3)  ≈ 1.73

So the OPTIMAL zone corresponds to questions that sit roughly 0.85 to 1.73 logits below the learner's ability: challenging, but with a high chance of success. Soca doesn't estimate θ or b today. Difficulty labels are static and admin-assigned, and the thresholds act as a model-free proxy for this condition.

7. Lesson mode

Each lesson conversation has three stages with rising difficulty:

Stage	Difficulty
Hook	Easy
Explore	Easy–Medium
Confirm	Medium (SAT-style MCQ)

A wrong answer triggers a Socratic loop instead of revealing the answer. Confirm gates completion. The lesson ends only when the Confirm stage is passed and required concepts are marked covered.

<!-- TODO: describe how the zone status feeds into the tutor in lesson mode once confirmed in code. -->
8. Tutor constraints (orthogonal to ZPD)

ZPD controls how hard. The tutor controls how to help:

Never give the direct answer.
Every turn ends in a question.
Concept coverage is tracked via hidden [CONCEPT_DONE:] tags.
max_exchanges per lesson is tiered by complexity and enforced server-side.
9. Data model
Table / function	Role
practice_questions	Question bank with skill_id, difficulty ∈ {1..5}
user_question_attempts	One row per answer; supplies (c_i, d_i)
lesson_config	Lesson structure and prompts
get_user_zpd_status	Computes zone(a) for a learner
10. Design decisions
Thresholds over Elo. No bidirectional Elo and no per-skill rating table. With a small question bank, rating estimates would be too noisy to beat a simple controller.
Static difficulty. d_i is assigned by admins, not fit from data.
No "teach it back" mechanic. Rejected for lower-confidence learners because it can feel exposing. Cross-lesson patterns use static topic tags instead.
11. Limitations and roadmap
W = 20 gives a noisy accuracy estimate (see §5).
Difficulty labels are not calibrated from outcomes. A future step is to estimate b per question and θ per learner (Rasch / 2PL) and select questions to hit a target P(correct) directly.
No pre/post diagnostic yet, so learning gains can't be measured directly.
Thresholds L, H are chosen from the literature and product judgment, not tuned on real learner data.
