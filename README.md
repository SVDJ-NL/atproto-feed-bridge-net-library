# ATProto Bridge .NET library 

AtProtoBridge is a lightweight ASP.NET Core service that bridges the [AT Protocol](https://docs.bsky.app/docs/api/at-protocol-xrpc-api) / [Bluesky](https://bsky.social/) Jetstream firehose into your own applications. It listens for posts and reposts from a configurable set of accounts, enriches them with author and media data from the Bluesky AppView, and makes the resulting feed available both as a simple REST snapshot and as a real-time SignalR stream.

A working example of displaying the live feed over `wss`/SignalR can be found in [wwwroot/index.html](wwwroot/index.html): it loads the initial snapshot from `GET /feed` and then connects to the `/feedHub` SignalR hub to receive and render new posts (and removals) in real time. It is served automatically when you run the project and open the site's root URL.

## Background and Purpose

AtProto Feed Bridge is the backend component of an editorial feed that automatically integrates posts published by editors on AT Protocol-based social media platforms, such as oso.social and Bluesky. The editorial feed allows news organizations to introduce their audiences to alternative social media platforms.

Users do not need an account on the underlying social media platform to read editors' posts in the news organization's app or on its website. An account is only required if users want to respond to the original posts.

The project originated within the Incubator 2026, developed by Team 6 (De Gieters) in collaboration with RTVSoftware.

The software is intended for news organizations and other interested parties who want to reduce their dependence on Big Tech platforms in their relationship with their audiences. It provides a foundation for developing an independent editorial feed for an existing app or website.

AtProto Feed Bridge retrieves posts from AT Protocol repositories and makes them available to other applications through a REST API and a real-time WebSocket connection using SignalR.

## Overview

* Connects to a [Jetstream](https://github.com/bluesky-social/jetstream) instance and subscribes to `app.bsky.feed.post` and `app.bsky.feed.repost` events for a configurable list of account DIDs.
* On startup, backfills the most recent posts and reposts for those accounts directly from their PDS, so the feed is populated immediately.
* Resolves author profiles and parses link previews, images and video embeds through the public Bluesky AppView API, with in-memory caching to limit outgoing requests.
* Tracks reply threads: each post carries its `rootUri`, `parentUri` and a `threadIndex`, so replies can be shown in order after the root post they belong to.
* Keeps the latest feed items in memory and exposes them through a REST endpoint and a SignalR hub.
* Intended as a reusable building block for running your own curated Bluesky feed (for a community, organization or region) without having to build a full feed generator or Jetstream consumer from scratch.

## Frontend

This repository contains backend software only. No frontend is included. Organizations wishing to use the software must develop their own frontend, tailored to their specific requirements and visual identity.

> The [wwwroot/index.html](wwwroot/index.html) page referenced above is a minimal working example for demonstration and testing purposes only, and is not intended as a production-ready frontend.

## Example Implementation

An example of the software in use is [1Almere Dichtbij](https://1almere.nl/), the editorial feed integrated into the 1Almere broadcaster's app. This feed was inspired by the editorial feed (*redactiefeed*) from NU.nl.

## Project Status

AtProto Feed Bridge is in **production** use, as the backend for the 1Almere Dichtbij editorial feed, and is actively maintained.

## Project Structure

| Path | Role |
|---|---|
| [Program.cs](Program.cs) | Application entry point: wires up dependency injection, configuration options, CORS, SignalR and the minimal API endpoints (`/echo`, `/feed`, `/feedHub`). |
| [BackgroundWorker.cs](BackgroundWorker.cs) | Hosted background service that backfills posts on startup, runs the Jetstream subscription, and forwards feed notifications to connected SignalR clients. |
| [AtProtoCollections.cs](AtProtoCollections.cs) | Constants for the AT Protocol collection types (`app.bsky.feed.post`, `app.bsky.feed.repost`) the service tracks. |
| [Configuration/](Configuration) | Strongly typed options classes (`ATProtoOptions`, `JetstreamOptions`, `WebserverOptions`) bound from `appsettings.json`. |
| [Hubs/FeedHub.cs](Hubs/FeedHub.cs) | SignalR hub exposed at `/feedHub` for real-time feed updates. |
| [Parsers/](Parsers) | Helpers that parse external link previews (`ExternalParser.cs`) and image/video embeds (`MediaEmbedParser.cs`) out of raw AT Protocol post records. |
| [Records/](Records) | Immutable record types (DTOs) representing feed items, posts, authors, media and internal notifications. |
| [Services/JetstreamServices.cs](Services/JetstreamServices.cs) | Connects to the configured Jetstream instance and subscribes to post/repost events for the configured account DIDs. |
| [Services/AppViewServices.cs](Services/AppViewServices.cs) | Resolves author profiles through the public Bluesky AppView API, with in-memory caching. |
| [Services/PostServices.cs](Services/PostServices.cs) | Builds enriched `FeedPost` objects from raw records and handles the initial backfill from the PDS. |
| [Services/FeedServices.cs](Services/FeedServices.cs) | Reads the current feed snapshot from the store for the `GET /feed` endpoint. |
| [Stores/FeedItemStore.cs](Stores/FeedItemStore.cs) | In-memory store holding the most recent feed items and managing reply-thread indices. |
| [wwwroot/index.html](wwwroot/index.html) | Example page demonstrating the REST snapshot and the live SignalR stream (demo/testing only, see [Frontend](#frontend)). |
| [appsettings.json](appsettings.json) | Application configuration: CORS origins, AT Protocol account DIDs/agent URL, Jetstream URL — see [Configuration](#configuration). |

## Technology Stack

* **Backend**: [ASP.NET Core](https://dotnet.microsoft.com/apps/aspnet) minimal APIs on [.NET 10](https://dotnet.microsoft.com/) (`net10.0`), with [SignalR](https://dotnet.microsoft.com/apps/aspnet/signalr) for the real-time `/feedHub` stream.
* **Frontend**: none included; see [Frontend](#frontend). The [wwwroot/index.html](wwwroot/index.html) page is a demo only, built with plain HTML/JS and the `@microsoft/signalr` client.
* **Data storage**: none — feed items are kept in memory ([Stores/FeedItemStore.cs](Stores/FeedItemStore.cs)) for the lifetime of the process; nothing is persisted to disk or to a database.
* **Infrastructure**: no infrastructure-as-code, container image or deployment pipeline is included in this repository; the service is a self-contained ASP.NET Core app that can be run with `dotnet run`/`dotnet publish` or hosted in any standard .NET 10 hosting environment.
* **Key dependency**: [idunno.AtProto](https://github.com/blowdart/idunno.Bluesky) (NuGet) for AT Protocol/Bluesky client functionality — see [External dependencies](#external-dependencies).
* **AI models**: none. See [AI Transparency](#ai-transparency).

## Datasets and Data Sources

AtProto Feed Bridge does not ship with any bundled dataset. At runtime it retrieves data live from the following public AT Protocol / Bluesky services, as configured in [appsettings.json](appsettings.json):

* The configured **AT Protocol PDS/agent** (`Application:ATProto:AgentUrl`, e.g. `https://pds.oso.social` or `https://bsky.social`) — used to resolve accounts and backfill the most recent posts/reposts on startup.
* The public **Bluesky AppView API** — used to resolve author profiles and to parse link previews, images and video embeds.
* A **Jetstream** instance (`Application:Jetstream:Url`, e.g. `wss://jetstream1.eurosky.network/receive`) — the real-time firehose of AT Protocol repository events that the service subscribes to. See the [Jetstream repository](https://github.com/bluesky-social/jetstream) for available public instances.

Only posts and reposts from the explicitly configured account DIDs (`Application:ATProto:Dids`) are retrieved and re-exposed through `/feed` and `/feedHub`; the service does not crawl or redistribute the wider AT Protocol/Bluesky network.

## Getting Started

### Prerequisites

* [.NET 10 SDK](https://dotnet.microsoft.com/download) (the project targets `net10.0`).
* No database, Docker or other external services are required to run the project locally — NuGet packages are restored automatically via `dotnet run`/`dotnet restore`.

### Local development

1. Edit `appsettings.json` with your own DIDs, PDS/agent URL and allowed CORS origins — see [Configuration](#configuration) below. There is no `.env` file; all configuration is read from `appsettings.json` (and, if set, the matching ASP.NET Core environment variables/`appsettings.{Environment}.json` overrides).
2. Restore and run the project:
   ```bash
   dotnet restore
   dotnet run
   ```
3. The service starts consuming Jetstream immediately and serves:
   * `GET /feed` — the current feed as JSON
   * `/feedHub` — a SignalR hub (`wss://`) for real-time updates
   * `GET /echo` — a simple connectivity check

### Production

Publish a self-contained or framework-dependent build and run the resulting binary/DLL on any host with the .NET 10 runtime installed, e.g.:

```bash
dotnet publish -c Release -o ./publish
dotnet ./publish/AtProtoBridge.dll
```

Set `ASPNETCORE_ENVIRONMENT=Production` (and provide a corresponding `appsettings.Production.json`, if needed) to use production configuration overrides. No container image or Docker Compose setup is provided in this repository at this time.

## Configuration

All settings live in `appsettings.json`:

| Section | Key | Description |
|---|---|---|
| `Webserver` | `CorsAllowed` | Array of allowed CORS origins — add the URL(s) of the website(s)/app(s) that will call `/feed` or connect to `/feedHub`. |
| `Application:ATProto` | `AgentUrl` | Base URL of the AT Protocol agent/PDS used to resolve accounts and backfill posts on startup (e.g. `https://pds.oso.social` or `https://bsky.social`). |
| `Application:ATProto` | `Dids` | The account DIDs (`did:plc:...`) whose posts and reposts are included in the feed. This is the main setting to change when reusing this project for a different community or set of accounts. |
| `Application:Jetstream` | `Url` | WebSocket URL of the Jetstream instance to consume, e.g. `wss://jetstream1.eurosky.network/receive`. See the [Jetstream repository](https://github.com/bluesky-social/jetstream) for public instances. |

> The sample `appsettings.json` also contains `Application:ATProto:Handle` and `Application:ATProto:Collections` keys. These are not currently read by the application — the tracked collections are hard-coded to `app.bsky.feed.post` and `app.bsky.feed.repost` — so they can be ignored or removed.

## Usage

### REST: `GET /feed`

Returns the latest feed items (default 50), newest first, as a JSON array:

```http
GET /feed
```

```json
[
  {
    "id": "at://did:plc:example/app.bsky.feed.post/abc123",
    "post": {
      "uri": "at://did:plc:example/app.bsky.feed.post/abc123",
      "postedBy": {
        "did": "did:plc:example",
        "handle": "example.bsky.social",
        "displayName": "Example",
        "avatarUrl": "https://..."
      },
      "text": "Hello from Bluesky!",
      "rootUri": "at://did:plc:example/app.bsky.feed.post/abc123",
      "parentUri": null,
      "threadIndex": 0,
      "createdAt": "2026-01-01T12:00:00Z",
      "external": null,
      "images": [],
      "videos": []
    },
    "repostedBy": null,
    "feedTimestamp": "2026-01-01T12:00:00Z"
  }
]
```

Each post includes `rootUri`, `parentUri` and `threadIndex`, which together describe its position in a reply thread:

* `rootUri` is the `uri` of the first (root) post of the thread — equal to the post's own `uri` for root posts.
* `parentUri` is the `uri` of the post it directly replies to, or `null` for root posts.
* `threadIndex` is `0` for root posts, and `1`, `2`, `3`, ... for subsequent replies on the same `rootUri`, ordered by `createdAt`.

This lets consumers group feed items by `rootUri` and sort them by `threadIndex` to display a thread with the root post first, followed by its replies in the order they were posted.

### Real-time: SignalR hub `/feedHub`

Connect over WebSockets (`wss://`) using a [SignalR client](https://learn.microsoft.com/aspnet/core/signalr/client-features) and listen for:

* `ReceivePost` — invoked with the post payload whenever a new post or repost is received from Jetstream.
* `RemovePost` — invoked with the post `uri` (`string`) whenever a post is deleted.

Example using `@microsoft/signalr`:

```js
const connection = new signalR.HubConnectionBuilder()
  .withUrl("https://your-host/feedHub")
  .withAutomaticReconnect()
  .build();

connection.on("ReceivePost", (post) => {
  console.log("New post", post);
});

connection.on("RemovePost", (uri) => {
  console.log("Post removed", uri);
});

await connection.start();
```

## AI Transparency

No generative AI tools were used during the development of this software.

## License

This package is available under the MIT license,
see the [LICENSE](LICENSE) file for more information.

## Support Policy

Only the latest version of the libraries are supported. All bug fixes, including security fixes, will be made in the latest version.
Older versions will not receive any updates.

The library is supported on all in-support versions of .NET.

### External dependencies

* [idunno.AtProto](https://github.com/blowdart/idunno.Bluesky)