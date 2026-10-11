# Setting up Midflight

There are three ways to set Midflight up. Most people only need the first.

| You want to | Go to | You need |
| --- | --- | --- |
| **Use Midflight with your team** | [Part 1](#part-1-use-the-hosted-midflight) | A GitHub account and an AI client. Nothing to install, about 5 minutes. |
| **Run Midflight on your own laptop** | [Part 2](#part-2-run-midflight-on-your-laptop) | Python 3.12 and uv |
| **Deploy your own copy to AWS** | [Part 3](#part-3-deploy-your-own-copy-to-aws) | An AWS account and your own GitHub App |

## Part 1: use the hosted Midflight

Midflight is one hosted service, run by Team Yoga. Nobody on your team clones this
repository or runs a server. One person is the **lead**; everyone else is a **teammate**.

The connector URL, for everyone:

```
https://5hwub7vaxiyz6oezhxrs3qivaa0cvjvy.lambda-url.us-east-1.on.aws/mcp
```

It works from Claude Code, Claude, ChatGPT, and Codex.

### Everyone: connect your AI client

**Claude Code**

```bash
claude mcp add --transport http --scope user midflight https://5hwub7vaxiyz6oezhxrs3qivaa0cvjvy.lambda-url.us-east-1.on.aws/mcp
```

Start `claude`, type `/mcp`, choose **midflight**, and sign in with GitHub in the browser
that opens.

**Codex**

```bash
codex mcp add midflight --url https://5hwub7vaxiyz6oezhxrs3qivaa0cvjvy.lambda-url.us-east-1.on.aws/mcp
```

```bash
codex mcp login midflight
```

The second command opens a browser to sign in with GitHub.

**Claude or ChatGPT (the apps):** choose **Add custom connector**, paste the connector
URL, and sign in with GitHub when asked.

✅ Ask your agent "list my Midflight projects". It answers that you aren't in a project
yet.

### Lead: create the project (once per repository)

1. **Install the Midflight GitHub App** on the repository your team builds in:
   https://github.com/apps/midflightt/installations/new. Choose **Only select
   repositories** and pick that repository. You must be an admin of it.
2. **Create the project.** Ask your agent:

   > Create a Midflight project for `owner/repo`.

   It answers with a project id and a join code like `MF-7K2Q-9XPA`.
3. **Send the join code to your teammates** privately. Anyone who has it can join.
4. **Load the plan**, after your teammates have joined (a task's owner must be a member).
   Describe the plan to your agent, or point it at a plan file:

   > Read PLAN.md. Propose it to Midflight: the requirements, the tasks with their
   > owners' GitHub logins and branches, and the contracts with their fields and types.
   > Then approve the plan with the reason "First plan".

✅ "Show the Midflight project status" lists every task with the right owner.

### Teammate: join

1. Connect your AI client (above).
2. Ask your agent, with the code your lead sent:

   > Join Midflight project MF-XXXX-XXXX.

✅ "List my Midflight projects" shows the project.

### Everyone: work on a task

Make a branch for your task, then tell your agent:

> My Midflight task is T1. Check in with Midflight, submit your claim, and wait for the
> verdict. Then build it.

The agent claims before it writes code, checks in at the points it planned, and answers
any directive that is waiting for it. How this works is in the
[README](README.md#how-an-agent-stays-in-sync-checkpoints-and-check-in).

Claude Code is sent Midflight's working rules when it connects. Other clients may not
be, so spell the steps out in the prompt: check in, claim with every assumption, wait
for the verdict, answer directives, and stop on anything that needs the lead.

### Optional: the pre-push hook

The hook stops a `git push` while your claim isn't approved or a directive is unanswered.
In your clone of the team's repository, ask your agent:

> Set up the Midflight pre-push hook for my task.

Run the commands it gives you in the repository root. They stay in `.git` and are never
committed.

✅ A push before your claim is approved prints "midflight: push blocked".

### Optional: the `midflight/verify` check on pull requests

After a push, Midflight checks the real commit and posts `midflight/verify` on the pull
request. It starts when a GitHub Actions workflow in a file named `contract.yml`
finishes, so the repository needs its tests in a workflow with that file name. The
branch must belong to an approved claim or have an open pull request.

### A full walk-through

[docs/guides/pigeonpost-with-a-friend.md](docs/guides/pigeonpost-with-a-friend.md) takes
two people through a whole project, with every prompt written out.

## Part 2: run Midflight on your laptop

This is for reading the code, running the tests, or trying the connector without the
hosted service. You need Python 3.12 and [uv](https://docs.astral.sh/uv/).

```bash
git clone https://github.com/MidFlightt/midflight-submission.git
```

```bash
cd midflight-submission
```

```bash
uv sync
```

```bash
cp .env.example .env
```

(In PowerShell: `Copy-Item .env.example .env`.)

```bash
uv run midflight-server
```

The server is at http://127.0.0.1:8000, with the REST documentation at
http://127.0.0.1:8000/docs.

✅ http://127.0.0.1:8000/healthz answers `{"status":"ok"}`.

Connect an AI client to the local server:

```bash
claude mcp add --transport http midflight-local http://127.0.0.1:8000/mcp
```

`.env.example` sets `MIDFLIGHT_DEV_LOGIN=1`, so sign-in is a local form where you type
any GitHub login. No GitHub secrets are needed, and no GitHub App. Data is held in
memory and resets when the server stops. The AI reviewer is off unless you set
`MIDFLIGHT_REVIEWER_MODEL` and have AWS credentials that can call Amazon Bedrock; without
it, claims are checked by the explicit rules only.

Run the tests and the linter:

```bash
uv run pytest
```

```bash
uv run ruff check .
```

✅ `433 passed`, and `All checks passed!`. The tests use fakes for GitHub, DynamoDB, and
Bedrock, so they need no accounts.

## Part 3: deploy your own copy to AWS

Team Yoga deploys Midflight once and every team uses that deployment, so you only need
this to run a separate copy. The full steps are in [infra/README.md](infra/README.md).
In short:

1. **Tools:** AWS CLI v2 and AWS SAM CLI, signed in to an account, region `us-east-1`.
2. **Your own GitHub App**, with a client secret, a private key, and a webhook secret in
   your `.env`. Permissions: Checks read and write; Actions, Contents, Pull requests,
   and Metadata read. Event: Workflow run.
3. **Package and deploy** the SAM template (`infra/template.yaml`). It creates two
   Lambdas, a Function URL, a DynamoDB table, a Secrets Manager secret, and an SQS
   dead-letter queue.
4. **Store the secrets** with `infra/put_secrets.py`, then deploy again with `PublicUrl`
   set to the Function URL.
5. **Point the GitHub App** at the deployment's callback and webhook URLs.
6. **Turn on the AI reviewer** by setting `ReviewerModel` to a Bedrock model your
   account can call.

## If something goes wrong

| Symptom | What to do |
| --- | --- |
| Your agent has no Midflight tools | In Claude Code type `/mcp` and check **midflight** is connected and signed in. Sign in again if it says it needs authentication. |
| "the Midflight GitHub App isn't installed on `owner/repo`" | Install it on that repository: https://github.com/apps/midflightt/installations/new |
| "isn't an admin of `owner/repo`" | Only a repository admin can create its project. Ask an admin to be the lead. |
| A teammate can't be given a task | They haven't joined yet. A task's owner must be a member. |
| The verdict is `human_review_required` | The lead has one question to decide. The lead asks their agent to "show the Midflight project status" and answers it. |
| A push is blocked | Ask your agent to check in with Midflight. It lists what is stopping the push. |
| No `midflight/verify` on a pull request | The workflow file must be named `contract.yml`, and the branch must belong to an approved claim. |
| Locally, sign-in opens GitHub and fails | `.env` is missing `MIDFLIGHT_DEV_LOGIN=1`. Copy `.env.example` again. |

More, for the AWS deployment: [infra/README.md](infra/README.md#if-something-goes-wrong).
