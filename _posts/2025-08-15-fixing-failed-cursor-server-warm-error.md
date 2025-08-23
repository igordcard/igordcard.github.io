---
layout: post
comments: true
title: Fixing Failed to warm cursor server error
author: Igor DC
tags:
- Cursor
- Programming
- Vibe coding
- AI
- Agents
---

<span style="display: block; text-align: center">![](/assets/cursor-warm-error.png "Cursor server warm error"){:width="80%"}</span><br/>

If you're developing using Cursor and, after clicking "Open in VM", you see `Failed to warm cursor server` while it tries to connect to the remote background agent VM, it's likely because the agent's active model was changed (or you lost access to the previously used model). That causes a model mismatch and the VM won't come up.

<span style="display: block; text-align: center">![](/assets/cursor-open-vm.png "Cursor Open VM button"){:width="80%"}</span><br/>


Try this:

- Update the background agent's model in Cursor to match the model that agent last used. Once they match, the VM should become reachable again.
- If that doesn't work, create a new background agent using the default model listed on the [Cursor dashboard](https://cursor.com/dashboard?tab=background-agents).