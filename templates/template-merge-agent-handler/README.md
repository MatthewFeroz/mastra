# Your product's own @bot on X, with Merge Agent Handler

Someone hits a bug, tags your bot in the thread on X, and asks it to file an issue. The bot reads the original post and the reply chain, opens a GitHub issue through [Merge Agent Handler](https://www.merge.dev/agent-handler), and replies under the mention with the link. Ask it to summarize the thread instead and it replies with the main points.

## Why we built this

Public bots like @grok made "tag the bot in the thread" a normal way to get things done. Any product can have one, but a bot that anyone on the internet can trigger needs a narrow, governed action surface.

- **Mastra** runs the agent, the X channel, the signed webhook, storage and Studio.
- **Merge Agent Handler** holds the GitHub credentials and exposes one tool, `github__create_issue`, through a Tool Pack. Every call is recorded in Tool Call Logs, and the bot never sees the GitHub token.
- **Merge Gateway** routes model calls, so model spend and traffic are in one place too. Any Mastra model string works if you'd rather use your own provider key.

The template demonstrates Mastra agents with tools, channels with the X adapter, custom API routes, LibSQL storage, and tool calls brokered over MCP.

## Demo

`npm run demo` plays two X threads through the real agent. One asks for an issue and one asks for a summary. Only the threads are scripted, and the agent decides what to do. Issue creation is a dry run unless you opt in.

Open `/demo` on a running server for a live view of each request as it moves through the system. It shows the mention, thread read, model call, Agent Handler call, issue and reply, with elapsed time for each step.

## Prerequisites

- [Merge Gateway API key](https://www.merge.dev/gateway) for the default `merge-gateway/anthropic/claude-sonnet-4-6` model. Set `MODEL` to swap in any Mastra model string, such as `openai/gpt-5-mini` with `OPENAI_API_KEY`.
- [Merge Agent Handler](https://ah.merge.dev) access for real issues. Connect GitHub for a Registered User and create a Tool Pack with only `github__create_issue`.
- An [X developer app](https://developer.x.com) and a bot account for real mentions. X API reads and writes need available credits.

## Quickstart 🚀

1. **Clone the template**
   - Run `npx create-mastra@latest --template template-merge-agent-handler` to scaffold the project locally.
2. **Add your API keys**
   - Copy `.env.example` to `.env` and add `MERGE_GATEWAY_API_KEY`. Everything else is optional for the local demo.
3. **Run the demo**
   - Run `npm run demo` to watch the agent handle both threads in your terminal.
4. **Start the dev server**
   - Run `npm run dev` and open [localhost:4111](http://localhost:4111) to try the agent in Studio.

`npm test` checks the whole loop against local fakes of X and Agent Handler, so it needs no credentials and posts nothing.

## Opening real issues

1. In Merge Agent Handler, connect GitHub for a Registered User and restrict the connection to your repo.
2. Create a Tool Pack with `github__create_issue` and nothing else.
3. Fill in `AGENT_HANDLER_API_KEY`, `AGENT_HANDLER_TOOL_PACK_ID`, `AGENT_HANDLER_REGISTERED_USER_ID`, `GITHUB_OWNER` and `GITHUB_REPO`. The repo needs the labels `from-x`, `bug`, `regression`, `docs` and `feature request`.

The terminal demo and Studio stay dry runs even with credentials. Set `DEMO_LIVE_ACTIONS=true` only for an intentional live terminal demo, and keep it false on a public server.

`npm run test:live` checks the real Agent Handler tool catalog, the X identity, and model inference. It creates no issues and sends no posts.

## Answering real mentions on X

1. Set `X_BOT_USERNAME`, `X_CONSUMER_KEY`, `X_CONSUMER_SECRET`, `X_CLIENT_ID` and `X_CLIENT_SECRET` from your X app. The app needs `tweet.read`, `tweet.write`, `users.read` and `offline.access`.
2. Generate `X_ENCRYPTION_KEY` with `openssl rand -base64 32`. It encrypts rotated OAuth tokens in the database, so keep it across deployments.
3. Run `npm run x-auth`, sign in as the bot account and click Authorize. The script saves the bot's tokens to `.env`.
4. Deploy, register `https://<your-host>/webhooks/x` as an X webhook, and subscribe the bot to `post.mention.create`.

The `/webhooks/x` route answers X's GET CRC challenge and verifies signed POST deliveries before handing them to the Mastra channel.

## How the bot stays safe in public

- **One action.** The agent has one tool, and the server fixes the owner and repo. The model fills in only a title, body and labels.
- **Only the latest mention can ask.** The server checks the newest post for an explicit request to open, file or report an issue. Summary requests disable tools, and instructions in earlier posts can't grant write access.
- **One issue per mention.** Each mention is claimed in the database before the model runs, so a redelivery or restart can't file it twice.
- **Spam limits.** By default each person can open 3 issues an hour, everyone together 25 a day, and each person gets 10 mentions an hour before the bot goes quiet. Change them with the `TAGBOT_*` variables.
- **No made-up product claims.** Asked whether Merge connects to a tool, the bot looks it up in Merge's public connector catalog with `checkConnector` instead of answering from memory.
- **Prompt injection is blocked.** Merge Gateway screens each model call. A blocked mention never reaches the model, and the bot replies with a fixed refusal.
- **Short, plain replies.** Replies are trimmed to 240 characters and never include provider error text.

## Deploying

Use a persistent LibSQL database such as Turso for `DATABASE_URL` and `DATABASE_AUTH_TOKEN`. A local file on ephemeral hosting loses mention claims, spam counts and rotated tokens.

By default the bot keeps X thread locks and queues in memory, so run one instance. To run more, set `REDIS_URL` to a Redis database, such as Upstash, that every instance can reach. Instances then share locks and queues, and the database still makes sure each mention files at most one issue.

Set `TAGBOT_API_TOKEN` to protect Studio's APIs and the live request feed. The X webhook stays reachable because it checks X's signature itself.

## Making it yours

- **Change the action.** Swap `github__create_issue` for any of Agent Handler's connectors, such as a Linear ticket, a Jira issue or a Zendesk ticket. Update the Tool Pack, `src/mastra/tools.ts` and the agent's instructions.
- **Change the voice.** Edit the instructions in `src/mastra/agents/tagbot.ts`.
- **Change what counts as a request.** Edit `asksForIssue` in `src/mastra/x-handler.ts`.
- **Change the demo threads.** Edit `scripts/scenario.ts`.

## About Mastra templates

[Mastra templates](https://mastra.ai/templates) are ready-to-use projects that show off what you can build — clone one, poke around, and make it yours. They live in the [Mastra monorepo](https://github.com/mastra-ai/mastra) and are automatically synced to standalone repositories for easier cloning.

Want to contribute? See [CONTRIBUTING.md](./CONTRIBUTING.md).
