# Postman Headless: The Agentic Era

The CLI is to agents what the UI is to humans. This repo runs the whole
argument, live, in three parts:

1. **Act 1 — build & test, headless.** An agent lints `openapi.yaml`,
   generates a collection and a mock from it, and runs the collection
   against the mock. One binary, no hand-written test code, no UI.
2. **Act 2 — the question no UI can answer either.** The agent asks the
   Postman CLI's `context-graph` command who else depends on the endpoint
   you're about to change, before you change it.
3. **Act 3 — nobody prompts the agent this time.** You open a real PR that
   changes `openapi.yaml`. A GitHub Action triggers a headless agent that
   judges the diff, decides on its own whether it's worth asking the graph,
   and posts the verdict as a PR comment — before any human reviews it.

Acts 1 and 2 are things you drive from your terminal, with Claude Code.
Act 3 is the same CLI, the same reasoning, running without you.

---

## 0. One-time setup (repo owner only)

Skip this if you're just cloning to watch — it's already done. If you're
setting this repo up fresh:

```bash
npm install -g postman-cli@latest      # known-good: 1.62.0
postman login                          # decides which team's Context Graph answers
gh secret set POSTMAN_API_KEY          # paste the key when prompted — needed by Act 3's job
gh secret set ANTHROPIC_API_KEY        # from console.anthropic.com/settings/keys — lets Act 3 run Claude Code headless
```

Act 3's workflow (`.github/workflows/headless-agent.yml`) runs on
`pull_request` against `openapi.yaml`. GitHub does not expose secrets to a
PR opened from a fork, so **open PRs from a branch on this repo**, not from
a fork, or Act 3's job will fail on the Postman login step.

---

## 1. Clone and get your bearings

```bash
git clone https://github.com/avdev4j/postman-cli-headless-agents.git
cd postman-cli-headless-agents
claude
```

`openapi.yaml` documents `GET /api/patients/{id}` on `patients-service` — a
real service in a real estate the Postman DevRel team's Context Graph has
already ingested. Nothing else exists yet: no collection you have to trust,
no mock server already running.

## 2. Act 1 — build and test it, headless

Paste this into Claude Code:

> Using nothing but the Postman CLI, validate openapi.yaml, generate a
> Postman collection from it, generate a mock from it, and run the
> collection against the mock to prove the API behaves as documented. Don't
> write any test code by hand — every step should be a Postman CLI command.
> If any generated request has a placeholder path variable instead of a
> real example value, fill it in before running. Show me the final result.

Payoff: a real `postman spec lint` (with a real warning — this contract is
missing a documented 5xx response, on purpose), a generated collection and
mock, and a **200 OK** — all without opening Postman once.

## 3. Act 2 — ask the thing no UI can answer either

Paste this next:

> I'm about to change the GET /api/patients/{id} endpoint on
> patients-service. Before I do, I need to know who else depends on it. Use
> the Postman CLI's context-graph command to ask which services or APIs call
> this endpoint and which teams own them, wait for the answer, and tell me
> what it says.

20–40 seconds — the graph is reasoning over the estate, not doing a lookup.
It comes back with real service names, real owning teams, and the evidence
for each.

## 4. Act 3 — make the change for real, and let the headless agent judge it

Now do the thing Act 2 just warned you about:

```bash
git checkout -b remove-blood-type
```

Edit `openapi.yaml`: remove the `blood_type` property from the `Patient`
schema (or rename it — either is a real, breaking change to the published
contract). Then:

```bash
git add openapi.yaml
git commit -m "Remove blood_type from the patient contract"
git push -u origin remove-blood-type
gh pr create --fill
```

Open the **Actions** tab, or just wait — within a minute or two, the
**Headless Reviewer** job runs and posts a comment on your PR. Nobody typed
a prompt for that run; the trigger was the PR itself. Read what it decided:
whether it judged the change risky, whether it asked the graph, and what the
graph's evidence said about who's actually affected.

---

## What each piece is

| File | Role |
|---|---|
| [`openapi.yaml`](openapi.yaml) | The one committed contract. |
| [`postman/collections/Patients Service/`](postman/collections/Patients%20Service/) | The checked-in test suite — Act 3's agent runs this against a fresh mock, it never regenerates it itself. |
| [`postman/mocks/patients-service/`](postman/mocks/patients-service/) | The mock, regenerated live in Act 1 and again by Act 3's agent. |
| [`agents/headless-pr-agent.md`](agents/headless-pr-agent.md) | What the headless agent in Act 3 actually reasons through — not a fixed script, a set of judgment calls. |
| [`.github/workflows/headless-agent.yml`](.github/workflows/headless-agent.yml) | The trigger: on every PR touching `openapi.yaml`, run the agent, headless. |

## Troubleshooting

| Issue | Fix |
|---|---|
| Act 2 gets a refusal ("this API does not exist in the graph") | Your `postman login` account isn't on a team with the estate ingested. Sign in as one that is, or swap the question for a real API you own. |
| Act 3's job fails on `postman login` | Almost always a fork PR — secrets aren't passed to those. Push the branch to this repo instead. |
| Act 3's job fails on `claude -p` with an auth error | `ANTHROPIC_API_KEY` isn't set (or is wrong) as a repo secret — see [section 0](#0-one-time-setup-repo-owner-only). |
| No comment shows up on the PR | Check the Actions tab for the run's logs — the agent still executed, it just may have decided (correctly) that nothing about your change was risky, and said so in one line instead of a long comment. |
