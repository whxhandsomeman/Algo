# Private Relational State for Hidden-Role Multi-Agent Games

This anonymous repository contains the scenario rules, evaluation metrics, algorithm reference, and prompt templates for PRS-based hidden-role multi-agent language interaction experiments.

## Contents

1. [Game Rules](#1-game-rules)
   - [Avalon Scenario Rules](#11-avalon-scenario-rules)
   - [Who Is the Undercover Scenario Rules](#12-who-is-the-undercover-scenario-rules)
2. [Evaluation Metrics](#2-evaluation-metrics)
3. [Prompt Templates](#4-prompt-templates)

## 1. Game Rules

### 1.1 Avalon Scenario Rules

#### Overall Rules

This environment implements a standard multi-round Avalon game. Players are divided into the Good camp and the Evil camp. Good players aim to pass three quests, while Evil players aim to make three quests fail or to win by assassinating Merlin after the Good camp has passed three quests.

The game contains five quests. In each round, the current leader proposes a team of a predefined size. All players then vote on whether to approve the proposed team. If the proposal receives a strict majority of approval votes, the team is sent on the quest. Otherwise, the proposal is rejected, the leadership passes to the next player, and a new proposal begins.

The environment also keeps a rejection track. If four consecutive proposals are rejected, then the fifth proposal is executed directly without an approval vote. This mechanism prevents the game from stalling through repeated rejection.

Once a team is approved, only the selected team members participate in the quest. Good players can only submit a success card. Evil players may submit either a success card or a fail card. A quest succeeds unless the number of fail cards reaches the failure threshold for that round. Most rounds fail with one fail card, but some larger-player configurations require two fail cards on a specific round.

The game ends immediately if either camp wins three quests. If the Evil camp wins three failed quests first, Evil wins directly. If the Good camp wins three successful quests first, the game enters the assassination stage. In that final stage, the Assassin chooses one Good player as the assassination target. If the target is Merlin, Evil wins by reversal; otherwise, Good wins.

#### 7-Player Experimental Configuration

The experiments use a 7-player Avalon setup with the following roles:

- 1 Merlin
- 1 Percival
- 2 Loyal Servants of Arthur
- 1 Morgana
- 1 Assassin
- 1 Oberon

This configuration contains four Good players and three Evil players.

The five quest sizes are:

- Quest 1: 2 players
- Quest 2: 3 players
- Quest 3: 3 players
- Quest 4: 4 players
- Quest 5: 4 players

In the 7-player setting, Quests 1, 2, 3, and 5 fail if at least one fail card is submitted. Quest 4 is the only exception: it requires two fail cards to fail. Therefore, a single fail card on Quest 4 is not enough to make the Evil camp score that round.

The role information structure in this setup is as follows:

- Merlin knows the identities of Evil players except Mordred. Since Mordred is not included in the 7-player setup, Merlin can observe Morgana, Assassin, and Oberon.
- Percival sees two possible Merlin candidates, namely Merlin and Morgana, but cannot distinguish which one is the real Merlin.
- Morgana and Assassin know each other as Evil teammates.
- Oberon belongs to the Evil camp but is isolated: Oberon does not know the other Evil players, and the other Evil players do not know Oberon.

This 7-player board is strategically useful because it combines asymmetric hidden information with a nontrivial fourth-quest failure rule. As a result, players must jointly reason about public discussion, team composition, voting patterns, and quest outcomes under uncertainty.

### 1.2 Who Is the Undercover Scenario Rules

#### Overall Rules

This environment implements a multi-round "Who Is Undercover" game. At the beginning of each game, players are assigned to two camps: civilians and undercovers. Civilians share one secret word, while undercovers receive a different but semantically related word. The objective of the civilians is to identify and eliminate all undercovers. The objective of the undercovers is to avoid elimination and survive long enough to reach parity with the civilians.

Unlike many tabletop versions of the game, players in this environment only know their own secret word. They do not explicitly know whether they are a civilian or an undercover player. Therefore, each player must infer their likely identity from the relationship between their own word, other players' clues, and the observed voting behavior over time.

The game proceeds in repeated rounds. Each round contains two phases.

- Speech phase: every alive player gives one short clue sentence about their secret word.
- Voting phase: every alive player votes for one alive player to eliminate.

Players are not allowed to reveal their exact secret word directly. The clue should therefore be informative enough to help same-side players recognize semantic similarity, while still avoiding explicit disclosure.

After all votes are cast, the player with the highest number of votes is eliminated from the game. If multiple players are tied for the highest vote count, the environment breaks the tie uniformly at random among the tied candidates. The eliminated player is removed from all future rounds.

The game ends under any of the following conditions:

- Civilian win: all undercovers have been eliminated.
- Undercover win: the number of alive undercovers is greater than or equal to the number of alive civilians.
- Maximum-round termination: if the game reaches the predefined round limit and at least one undercover is still alive, the final result is recorded as undercover survival at max rounds.

This setup makes the task a repeated social deduction problem with partial observability. Players must reason jointly over semantic consistency in clues, changes in public suspicion, and the strategic consequences of each elimination.

#### Experimental Configuration

The experiments use the following configuration:

- 6 total players
- 1 undercover
- 5 civilians
- Maximum 20 rounds per game

At the start of each game, one civilian word and one undercover word are sampled as a semantically related pair. Each civilian receives the civilian word, and the undercover receives the undercover word. Example pairs used in the environment include `milk` vs. `soy milk`, `phone` vs. `tablet`, and `pizza` vs. `burger`.

Under this 6-player, 1-undercover setting, the game dynamics are straightforward but still strategically nontrivial. Because there is only one undercover, the civilian side wins immediately once that player is eliminated. However, the undercover can still win if repeated civilian mistakes reduce the number of alive civilians until parity is reached. For example, after several incorrect eliminations, a state with 1 undercover and 1 civilian alive is already an undercover win.

This configuration is useful for evaluation because it isolates a single hidden adversary while preserving multi-round public interaction. As a result, performance depends less on coalition coordination among multiple hidden agents and more on clue generation, semantic discrimination, vote targeting, and adaptive reasoning across rounds.

## 2. Evaluation Metrics

Please see [`Metrics.pdf`](./Metrics.pdf) for more details.Since GitHub doesn't natively support complex formula rendering, the PDF contains the complete mathematical specifications.


## 3. Prompt Templates

This section collects the core prompt templates for the two PRS environments:

1. **Graph Update Prompt**: updates the private relation graph from the latest observations.
2. **Final Generation Prompt**: generates the final action from the updated relation graph, graph-based decision hints, and the current task.

The core design of PRS separates **persistent social-state tracking** from **final language/action generation**. The former is handled by the structured private relation graph, while the latter is completed by the LLM under a compact prompt.

---

### 1. Avalon Prompt Templates

The Avalon setting requires long-horizon tracking of camp beliefs, team proposals, voting behavior, quest outcomes, and alliance relations. Therefore, the Avalon graph update prompt updates both explicit game-state fields and node/edge relation states.

#### 1.1 Graph Update Prompt for Avalon

##### Purpose

This prompt is called whenever new observations arrive. Before the call, temporal decay is applied to the private relation graph. The LLM then outputs structured graph updates from the latest observations, including:

- `state_updates`: explicit game-state updates, such as the current leader, most recent team, vote results, and score.
- `node_deltas`: additive updates to player node states.
- `edge_deltas`: additive updates to directed relations between players.

##### System Prompt Template

```text
You update a private Avalon relation graph.
You are given the current graph state after decay and the latest observations.
Your job is to output direct node/edge deltas plus explicit state-field updates.
Return exactly one JSON object and nothing else.

Output schema:
{
  "state_updates": {
    "current_leader": 2,
    "last_team": [1, 2, 3],
    "last_votes": {"1": true, "2": false},
    "good_score": 1,
    "evil_score": 0
  },
  "node_deltas": {
    "2": {"p_good": -0.10, "trust": -0.08, "impact": 0.03},
    "4": {"impact": 0.02}
  },
  "edge_deltas": {
    "2->3": {"support": -0.05, "attack": 0.10, "follow": 0.00},
    "4->1": {"support": 0.06, "follow": 0.04}
  }
}

Rules:
1. Deltas are additive adjustments applied to the decayed graph state.
2. Output only players and edges that should change materially.
3. Use explicit observations to justify deltas, but you may combine multiple observations into one net delta.
4. Keep deltas small and local. Typical absolute delta should usually stay <= 0.20.
5. `p_good`, `trust`, `impact`, `support`, `attack`, `follow` deltas may be positive or negative.
6. `current_leader`, `last_team`, `last_votes`, `good_score`, `evil_score` should be set only when the observations explicitly update them.
7. Preserve prior graph semantics: supportive speech raises support, accusatory speech raises attack, agreement raises follow, failed missions reduce team members' p_good/trust and may raise internal attack, successful missions increase team members' p_good/trust and may raise internal support.
8. Do not explain your reasoning. Output JSON only.
```

##### User Prompt Template

```text
Agent player_id={player_id}, role={role}
{graph_update_state_hint}

Current graph snapshot after decay will be updated by your deltas:
{current_graph_snapshot_json}

Observations:
{cleaned_observations}
```

##### Output Field Explanation

| Field | Description |
|---|---|
| `state_updates.current_leader` | Current quest leader. |
| `state_updates.last_team` | Most recently proposed quest team. |
| `state_updates.last_votes` | Most recent vote record. |
| `state_updates.good_score` | Current score of the Good camp. |
| `state_updates.evil_score` | Current score of the Evil camp. |
| `node_deltas.p_good` | Additive belief update that the player belongs to the Good camp. |
| `node_deltas.trust` | Additive update to the player's trustworthiness. |
| `node_deltas.impact` | Additive update to the player's influence or centrality. |
| `edge_deltas.support` | Support or defense relation from the source player to the target player. |
| `edge_deltas.attack` | Attack, suspicion, or opposition relation from the source player to the target player. |
| `edge_deltas.follow` | The source player's tendency to follow or align with the target player. |

---

#### 1.2 Final Generation Prompt for Avalon

##### Purpose

This prompt is called after graph update to generate the final action. Instead of feeding the full history back into the model, it combines the latest observations, private graph summary, graph decision hints, current task instruction, and output-format constraints into a compact prompt.

The Avalon final generation prompt is organized dynamically according to the player's camp:

- If the player belongs to the Evil camp, an `Evil Strategy Directive` is included.
- If summary memory is enabled, `Summarized Evidence` is used; otherwise, raw observations are provided as `Information you missed/observed`.
- If a private graph is available, `Private Relation Graph Snapshot` and `Graph Decision Hints` are included.

##### Final Generation Prompt Template

```text
== Evil Strategy Directive ==
{evil_phase_directive}
Keep statements natural and avoid deterministic vote patterns.

== Information you missed/observed ==
{observations}

== Private Relation Graph Snapshot ==
{graph_summary}

== Graph Decision Hints ==
{graph_decision_hints}

== Current Task Instruction ==
{phase_instruction}

== Output Format Requirements ==
Return exactly one JSON object and nothing else (no markdown/code fence/explanations).
Required format for this phase: {phase_json_hint}
```

> Note: `Evil Strategy Directive` appears only for Evil-side roles. If summary memory is enabled, `Information you missed/observed` is replaced by `Summarized Evidence`.

##### Phase-Specific Output Formats

```text
speech:
{"statement": "your speech"}

proposal:
{"team": [id1, id2, ...]}
## exactly {team_size} unique player ids

voting:
{"vote": true} or {"vote": false}

execution:
{"success": true} or {"success": false}

assassination:
{"target": player_id}
```

##### Graph Decision Hints

Avalon graph decision hints provide task-specific graph reasoning results for different phases. For example:

```text
Top team candidates from graph scoring:
- team=[...] score=... avg_p_good=... cohesion=... conflict=... hard_bad=...

Vote recommendation for current team [...]: YES/NO
(score=..., threshold=..., avg_p_good=..., cohesion=..., conflict=..., hard_bad=...)

Current discussion anchor team=[...], score=..., conflict=..., hard_bad=...
Speech focus: prioritize hard-role evidence, then relationship structure evidence.

Assassination candidates ranked by impact: [...]
```

These hints convert the private relation graph into actionable recommendations for proposal, voting, speech, execution, and assassination phases.

---

### 2. Who Is the Undercover Prompt Templates

The Who Is the Undercover (WITU) setting emphasizes short-term linguistic clues, semantic outliers, voting trends, and elimination results. Therefore, the WITU prompts focus on whether player descriptions are compatible with the group's semantic pattern and whether voting behavior reveals abnormal relations.

#### 2.1 Graph Update Prompt for WITU

##### Purpose

This prompt is called after each speech, vote, or elimination observation to update the private relation graph. Unlike Avalon, WITU uses `p_undercover` as its node belief field, representing the probability that a player is the undercover.

The graph update combines:

- the latest observation text;
- the current graph snapshot;
- structured vote pairs for the current round;
- elimination information;
- the current agent identity: civilian or undercover.

##### System Prompt Template

```text
You update a private relation graph for the game Who Is Undercover.
You are given the current graph after decay and the latest observations.
Output only direct graph deltas and explicit state updates.
Return exactly one JSON object and nothing else.

Output schema:
{
  "state_updates": {
    "last_votes": {"1": 3, "2": 3, "3": 5}
  },
  "node_deltas": {
    "3": {"p_undercover": 0.12, "trust": -0.08, "impact": 0.01},
    "5": {"trust": 0.05}
  },
  "edge_deltas": {
    "1->3": {"support": -0.03, "attack": 0.08, "follow": 0.00},
    "2->1": {"follow": 0.05}
  }
}

Rules:
1. Deltas are additive and are applied after decay.
2. Use only these fields: `p_undercover`, `trust`, `impact`, `support`, `attack`, `follow`.
3. Keep deltas small and local. Typical absolute delta should usually stay <= 0.20.
4. Speech can change speaker impact and relationships implied by endorsement, accusation, or following.
5. Vote rounds can update `last_votes`, increase suspicion / reduce trust for voted targets, and update voter-voter follow or attack based on whether they voted the same target.
6. Elimination can change the eliminated player's `p_undercover`, and also adjust voters who supported that elimination depending on whether the eliminated role was `undercover` or `civilian`.
7. Do not output explanations.
8. {role_specific_rule}
```

##### User Prompt Template

```text
Agent player_id={player_id}, role={role}

Current graph snapshot after decay:
{current_graph_snapshot_json}

Observations:
{cleaned_observations}

Structured vote pairs for this round:
{vote_pairs_json}

Structured elimination update:
{elimination_json_or_None}
```

##### Output Field Explanation

| Field | Description |
|---|---|
| `state_updates.last_votes` | Most recent vote map in the format `voter_id -> target_id`. |
| `node_deltas.p_undercover` | Additive belief update that the player is the undercover. |
| `node_deltas.trust` | Additive update to the player's trustworthiness. |
| `node_deltas.impact` | Additive update to the player's influence. |
| `edge_deltas.support` | Support or agreement relation between players. |
| `edge_deltas.attack` | Attack, suspicion, or opposition relation between players. |
| `edge_deltas.follow` | Following or vote-alignment tendency between players. |

---

#### 2.2 Final Generation Prompt for WITU

##### Purpose

This prompt is called after the WITU graph update to generate the final speech or vote. It combines the latest observations, graph summary, graph decision hints, strategy directive, current task, and JSON output format.

##### Final Generation Prompt Template

```text
== Observations ==
{observations}

== Private Relation Graph Snapshot ==
{graph_summary}

== Graph Decision Hints ==
{graph_decision_hints}

== Strategy Directive ==
{strategy_directive}

== Current Task ==
{phase_instruction}

== Output Format ==
Return exactly one JSON object, no markdown or extra text.
Required format: {phase_json_hint}
```

##### Phase-Specific Instructions

For speech:

```text
Round {round_id} speech. Give one short clue sentence about your word.
Return JSON: {"statement": "..."}
```

For vote:

```text
Round {round_id} voting. Choose one suspicious alive player from: {alive_ids}.
Return JSON: {"vote_target": <int>, "reason": "..."}
```

##### Phase-Specific Output Formats

```text
speech:
{"statement": "..."}

vote:
{"vote_target": <int>, "reason": "..."}

other:
{"message": "..."}
```



### 3. Summary

| Environment | Graph Update Focus | Final Generation Focus |
|---|---|---|
| Avalon | Camp beliefs, team proposals, vote results, quest scores, and support/attack/follow relations | Graph-scored team selection, voting, speech, mission execution, and assassination decisions |
| WITU | Undercover probability, semantic clue outliers, voting momentum, elimination feedback, and player relations | Speech or vote generation based on semantic clusters, suspicion rankings, and voting trends |

Overall, the two environments share the same PRS pattern:

```text
New observations
    -> temporal decay
    -> graph update prompt
    -> updated private relation graph
    -> graph-supported decision hints
    -> final generation prompt
    -> JSON action
```
nder anonymity constraints.
