---
layout: post
title: "Authenticating MCPs: three ways we do it at Jollyes"
categories: [ai]
tags: [mcp, authentication, oauth, claude, jollyes, kaggle]
---
*I tried to use the Kaggle MCP, fell back to the CLI - and it made me write down the three ways we authenticate MCPs at Jollyes.*

<div class="tldr" markdown="1">
**TL;DR**

- Claude.ai connectors take only a URL, so they can't complete Kaggle's OAuth handshake. Every unauthenticated endpoint worked; the moment I asked to *download the competition data*, I got a 403.
- At Jollyes (and on my personal MCP) we authenticate Claude.ai three ways: **OAuth 2.0 over SSO** (the main one), **no auth** (when we're happy to share), and **a token - and identity hash - in the URL** (the neat, low-ceremony one).
- Knowing *who* is calling isn't just security. It unlocks my favourite MCP trick: **dynamic tool registration** - a different set of tools, and personalised tool descriptions, per user.
- My suggestion to Kaggle: only advertise the tools my current auth state can actually use, and let me pass a token as a query param.
</div>

I've subscribed to Kaggle's emails for over ten years and never once entered a competition. Yesterday one caught my eye - [cell tracking during development](https://www.kaggle.com/competitions/biohub-cell-tracking-during-development/leaderboard) - because it's closely aligned to a postdoc friend's work. She's spent the last several years counting and measuring plant cells in [MorphoGraphX](https://www.mpipz.mpg.de/MorphoGraphX), and in the last month has made huge strides building custom viewers and human-in-the-loop tools with Claude Code (more on that idea in another post). I thought I'd point the same approach at this Kaggle competition.

Naturally, the first job - after being amazed I'd bagged the `biologist` username years ago - was downloading the competition data. I was very excited to see Kaggle ships an [MCP](https://www.kaggle.com/docs/mcp).

![The Kaggle MCP tools listed in Claude.ai]({{ site.baseurl }}/images/kaggle-mcp-20260701.png)

Installing the MCP into Claude.ai - which nicely syncs connectors and auth across the web app, my phone, and Claude Code - was plain sailing. Until I asked, *"download the competition data"*: **403, no access**. Every non-OAuth endpoint worked; every authenticated one didn't.

To be fair to Kaggle, I hadn't read their docs. When I did:

> If your client is not OAuth 2.0 compliant, you can also use token authentication.

So I need to mint a token, because neither Claude.ai nor Claude Code will drive Kaggle's OAuth for me:

> Some resources or endpoints require authorization. To unlock full access you can authorize using OAuth 2.0.
>
> **Gemini CLI** - simply run: `/mcp auth kaggle`
>
> **Other clients/IDEs** - for clients without a command to initiate auth discovery, you can call the `authorize` tool.

From here I had two choices. Reinstall the MCP locally in Claude Code with the token baked into a header - and lose the cross-platform sync:

```json
{
  "mcpServers": {
    "kaggle": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://www.kaggle.com/mcp",
        "--header",
        "Authorization: Bearer YOUR_TOKEN"
      ]
    }
  }
}
```

Or just let Claude Code shell out to the Kaggle CLI. That was easiest, so that's what I did.

Now, this is really a Claude.ai limitation, not a Kaggle one. But Claude.ai is exactly what we use at Jollyes, and what I use personally to reach my own MCP from web Claude Code (more on that in another post). So the whole episode nudged me to write down the three ways we've given Claude.ai an *authenticated* MCP.

## 1. The main one - OAuth 2.0

This runs exactly opposite to Kaggle. I just said Claude.ai can't OAuth - but it can, *if the server does the work*. You set up the connection with a Client ID and Secret (which, wisely, Kaggle don't hand out) and build the auto-discovery and OAuth machinery on your MCP.

This works fantastically well for Jollyes. We already SSO into Claude.ai through Entra, so the natural flow is to use that same SSO to reach the MCP. The huge benefit, beyond security, is that we can see exactly who is using the MCP and for what.

The catch is non-domain users - subcontractors on other Entra tenants - who still need access. We could have built multi-tenant auth on the backend; more simply, we add their accounts as guest users in our own Entra.

There's a side benefit too. Because the MCP sits behind SSO in Claude.ai, every session a user spawns - whether or not they touch the MCP that session - triggers a `/validate` call. So we can measure overall AI usage across the business as well as MCP usage. This is the same SSO-gated pattern as our [stateful retail-analyst MCP](/stateful-retail-analyst-mcp/).

## 2. The easy one - no auth

Sometimes we're happy to share. No ceremony, no key, open endpoint.

## 3. The neat one - a token, and a hash, in the URL

You might say this is exactly what Kaggle offers by pasting a token into a header at set-up time, and you'd be right. But I hit the same wall Kaggle did: I really wanted my personal MCP available to [web Claude Code](https://claude.ai/code)[^webcc] with no extra work, and Claude.ai connectors take only a URL. So I allow the token as a query parameter:

```
https://mcp.matthew-johnston.com/mcp?token=XXX
```

We follow the same pattern at Jollyes wherever the tools are quick, short-lived, or light-touch and a full SSO set-up isn't worth it.

But I *like* knowing who's calling. So we could add a Kaggle-style `--user` header, or a `?user=` query param (which we do) - except, how do you know they are who they say they are? For that case we ask for both `?user=` and `?hash=`, and the server checks that `sha2(token + user) == hash`. A shared token proves you're allowed in; the hash binds a claimed identity to it.

## Why I care so much who's who

Authentication on its own is easy to justify - we hold commercially sensitive documents on the Jollyes MCP, and on my personal one I don't want people cheating at a game I host (more on that in another post). But plain access control only needs a key.

The real payoff of knowing *who* is calling is my favourite MCP trick: **dynamic tool registration**. Because the agent requests the connector's tool list at run time, from the MCP, we can return a *different* set of tools - and different definitions - per user. The simplest example: only our merchandising team gets write access to change stocking levels. The tool exists in the server, but only an allowlisted set of users ever *see* it.

And the neatest version - my favourite - is personalising the tool descriptions themselves. Our `draft_email` tool comes back with:

> **Description:** Prepare an email draft. Returns a `draft_id` you must pass to `send_email`. Recipients must be `@jollyes.com` / `jollyes.pet`. From: `claude@jollyes.com`. The user you are talking to is `matt@jollyes.com` - use this address for "email me" / "send this to me" / "ping me" style requests unless they name a different recipient.

Naturally, that email address is personalised to each user.

## Closing

My closing thought is that the Kaggle MCP might already be set up correctly - it just needs two things:

1. Only advertise the tools my current authentication state is allowed to call, rather than listing everything and 403-ing me halfway through.
2. Let me pass my token in as a query param, so a URL-only client like Claude.ai can carry it.

Do those, and the MCP would meet me where my client actually is - which, in the end, is the whole point of matching the ceremony to the risk.

[^webcc]: Confusingly distinct from CLI Claude Code, and definitely not desktop Claude.
