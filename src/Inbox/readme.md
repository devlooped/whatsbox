The [`Inbox`](https://www.nuget.org/packages/Inbox) package is the managed
**Inbox Client Protocol (ICP)** client (`InboxClient`) plus non-transitive MSBuild targets
for adapters that ship a native sidecar.

Apps that only want WhatsApp should PackageReference `WhatsBox` instead — these
targets are **not** transitive.

<!-- include ../../readme.md#inbox -->
<!-- #inbox -->
**Inbox Client Protocol (ICP)** is a **JSON-RPC 2.0 pub/sub bus over stdio** for local
companion processes. One binary owns one messaging-product session. Clients
subscribe to chats (and two system topics), send a small set of actions, and
receive live events. It is not an archive, a search engine, or a product CLI.

The [`Inbox`](https://www.nuget.org/packages/Inbox) package is the managed
client (`InboxClient`) for **any** implementation of that protocol. Spawn a
box, exchange newline-delimited JSON, and keep the process alive for the life
of the session. Product differences are opaque topic strings, `$session` auth
payloads, and `product` / `identity` / `capabilities` on `initialize`. There
are no per-product method names.

Wire version `"0.1"`: methods, events (`contents[]` on chat topics), files,
errors, and capabilities — [`docs/INBOX.md`](docs/INBOX.md).

```xml
<PackageReference Include="Inbox" Version="*" />
```

Target framework: `net10.0`. The managed surface is AOT-compatible
(source-generated JSON). Adapters that ship a native sidecar PackageReference
`Inbox` so pointer +
`dotnet pack -r` packaging is shared.

## InboxClient

`InboxClient` turns unary JSON-RPC methods into `Task<T>` and exposes a
single-consumer pull stream of typed `InboxEvent`s. Construct it over an
already-started NDJSON `TextReader`+`TextWriter` pair (the box’s stdout and
stdin). Disposing the client completes `Events`.

Start consuming `Events` **before** (or concurrently with) a connecting
`InitializeAsync`. Auth events (QR, OAuth, device-code, …) arrive on
`$session` while that call is still waiting.

WhatsBox-shaped illustration — the same types work for any box:

```csharp
using Inbox;

await using var box = new InboxClient(stdout, stdin);

var pump = Task.Run(async () =>
{
    await foreach (var ev in box.Events)
    {
        switch (ev)
        {
            case SessionQr qr:
                // Product auth: WhatsBox emits qr; other boxes may emit oauth, …
                Console.WriteLine(qr.Code);
                break;
            case SessionOnline online:
                Console.WriteLine($"online as {online.Me}");
                break;
            case SessionPairError err:
                Console.Error.WriteLine(err.Message);
                break;
            case DirectoryReady:
                var page = await box.ListDirectoryAsync(new DirectoryListOptions { Kind = "user" });
                foreach (var row in page.Items)
                    Console.WriteLine($"{row.Name ?? row.Topic}");
                break;
            case DirectoryUpsert upsert:
                Console.WriteLine($"directory: {upsert.Name ?? upsert.Jid}");
                break;
            case ChatMessage msg:
                Console.WriteLine($"{msg.ByName ?? msg.By}: {msg.Text}");
                if (msg.Id is not null)
                    await box.ReadAsync(msg);
                break;
        }
    }
});

var session = await box.InitializeAsync(new InitializeOptions
{
    Store = store,
    Files = files,
    Subscribe = ["$directory"],
    Connect = true,
});

if (session.Status == "online")
{
    var listed = await box.ListDirectoryAsync(new DirectoryListOptions { Query = "alice" });
    var chat = listed.Items[0].Topic;
    await box.SubscribeAsync([chat]);
    await box.SendAsync(chat, text: "hello from Inbox");
}

await pump;
```

`InitializeAsync(store)` is the short form: no files, no extra subscriptions,
no connect. Pass `InitializeOptions` for blobs, initial topics, or
`Connect = true` (implicit `session.connect`).

| Method | RPC | Result |
|---|---|---|
| `InitializeAsync` | `initialize` | `SessionSnapshot` |
| `ConnectAsync` | `session.connect` | `SessionSnapshot` |
| `PairAsync` | `session.pair` | `SessionSnapshot` |
| `DisconnectAsync` | `session.disconnect` | `SessionSnapshot` |
| `LogoutAsync` | `session.logout` | `SessionSnapshot` (`new`) |
| `StatusAsync` | `session.status` | `SessionSnapshot` |
| `SubscribeAsync` / `UnsubscribeAsync` | `subscribe` / `unsubscribe` | `TopicsResult` (canonical topics) |
| `ListDirectoryAsync` | `directory.list` | `DirectoryListResult` |
| `GetDirectoryAsync` | `directory.get` | `DirectoryRow` |
| `SendAsync` / `ReactAsync` | `messages.send` | `SendResult` (`Id`, canonical `Topic`) |
| `ReadAsync` | `messages.read` | `ReadResult` |

`SessionSnapshot.Status` is `new` (never authenticated), `offline` (keys on
disk, socket down), or `online`. `Me` is the authenticated identity and is
omitted when `new`.

`SubscribeAsync` / `UnsubscribeAsync` take **canonical topics** only. Resolve
names with `ListDirectoryAsync` first. Results and event topics are always
canonical once the box knows them.

Send text (sugar), a file under `files`, a reply, and/or a reaction:

```csharp
await box.SendAsync(chat, text: "hello");

await box.SendAsync(chat, [new ImagePart { Path = "out/photo.jpg" }]);

await box.SendAsync(chat, text: "agreed",
    reply: new MessageReply(id, by));

await box.ReactAsync(chat, target: id, by, "👍");
```

`by` is required on reply, react, and `ReadAsync` — copy it from the inbound
event. Use `"me"` when targeting your own message. Mark-read is never
automatic.

RPC failures throw `InboxRpcException` with the JSON-RPC `Code` and a stable
`Token` (`not_initialized`, `files_required`, `not_found`, …).

stderr is logs only. It is never protocol.

## Events and contents

`Events` is a single-consumer `IAsyncEnumerable<InboxEvent>`. Enumerate it
once. It completes when the child stdout ends or the client is disposed.

| Type | Topic | Kind |
|---|---|---|
| `SessionQr` | `$session` | `qr` — string to render (WhatsBox illustration) |
| `SessionPaired` | `$session` | `paired` |
| `SessionPairError` | `$session` | `pair_error` |
| `SessionOnline` / `SessionOffline` | `$session` | `online` / `offline` |
| `SessionLoggedOut` | `$session` | `logged_out` |
| `SessionRemap` | `$session` | `remap` — subscription moved to a new canonical topic |
| `SessionOverflow` | `$session` | `overflow` — per-topic queue dropped oldest |
| `DirectoryUpsert` / `DirectoryRemove` / `DirectoryReady` | `$directory` | catalog changes |
| `ChatMessage` | chat topic | `message` — `Contents` (text, media, location, unknown); `Text` concatenates text parts |
| `ChatReaction` | chat topic | `reaction` — one reaction part |
| `ChatAck` | chat topic | `ack` — `delivered` / `read` / `played` |
| `ChatMeta` | chat topic | `meta` — join/leave/rename/… |

Chat events share `Id`, `By` (`"me"` or an opaque user id), `Handle`
(`@username` when known), `TopicName`, `ByName`, and `Contents`. Look up `By`
(or a 1:1 `Topic`) with `GetDirectoryAsync` for extra directory fields.

Content parts: `text`, `image`, `video`, `audio`, `document`, `sticker`,
`location`, `unknown`, plus `reaction` / `ack` / `meta` on those kinds.
Blob parts carry a relative `Path` under `initialize.files`.

## Implementing a box

An implementation speaks this envelope on stdio (JSON-RPC 2.0, one object per
line, no batch arrays). Advertise `product`, `identity`, and `capabilities` on
`initialize` / `session.status`. Consumers construct `InboxClient` over that
process — they never type-parse topic / `by` strings.

Suggested binaries (not normative): `whatsbox`, `discordbox`, `slackbox`,
`teamsbox`, `telegrambox`, `matrixbox`. Full method table, event shapes, and
error tokens: [`docs/INBOX.md`](docs/INBOX.md).
<!-- #inbox -->
<!-- ../../readme.md#inbox -->

## Adapter packing

`Inbox.targets` is imported only by a **direct** reference (nupkg `build/`, not
`buildTransitive/`). In this repo, `WhatsBox` ProjectReferences Inbox and
imports the file by hand.

Pointer + RID packing is **opt-in**. Declare `RuntimeIdentifiers` on the adapter
(and set `InboxNativeBinary`). Without that property, Inbox packs as a plain
managed library — no `runtime.json`, no `PackageId` suffix, no native RID assets.

The adapter:

1. Sets `RuntimeIdentifiers` and builds its native binary (`InboxNativeBinary`,
   optionally `InboxNativeName`, `InboxPackNativeDependsOn`,
   `InboxIncludeNativeAfterTargets`).
2. Packs the pointer: `dotnet pack` → adapter DLL + `runtime.json`.
3. Packs each RID: `dotnet pack -r {rid}` → `runtimes/{rid}/native/` only.
