---
title: "stdio over http, reluctantly"
date: 2026-05-25
draft: false
categories: ["mcp"]
tags: ["mcp", "stdio", "sse", "cloudflare"]
summary: "A local stdio MCP server became usable from remote Claude by putting a small HTTP/SSE bridge in front of it."
---

Some MCP servers really want to be local.

That is fine until the client is not local. A stdio server is pleasant when the model host and the tool host are the same machine. It is less pleasant when the thing that needs the tool is on the other side of the network and only knows how to talk HTTP.

The bridge shape was simple:

```text
HTTP request in
SSE stream out
one stdio MCP process behind it
```

I put that in front of mcpvault so remote Claude could read the vault through the same tool surface the local clients used. The public edge is just a normal `*.superterran.net` hostname. The private part stays boring: a service, a tunnel, and a little adapter that translates between HTTP/SSE and stdio.

The part worth remembering is the lifecycle. A stdio MCP process is not a daemon in the same way a web API is. It expects a conversation with one client. The bridge needs to respect that instead of pretending there is one immortal backend process everyone can share.

So the adapter treats sessions as sessions. A client connects, the bridge creates the tool process for that session, and the streams stay paired until the conversation ends.

I still prefer native HTTP MCP servers where they exist. Less adapter code, fewer lifecycle questions. But a small bridge is a useful pressure valve when the useful tool is already written for stdio and the client has moved somewhere else.
