# Laravel Reverb from a Flutter app (Windows desktop and Android), the 2026 way

How an offline-first Flutter app hears "something changed at your site" from Laravel Reverb, so devices sync at once
instead of polling. Built for a clinic records app (Flutter on Windows PCs and Android phones, drift/SQLite, Laravel
API with Reverb behind Apache), Nov 2026. Sync design: [../architecture/offline-first-sync-at-scale.md](../architecture/offline-first-sync-at-scale.md).

## Why bother: polling at scale

Every device asking "anything new?" every 2 seconds is about 80 requests a second for 160 idle devices. Each answer
is one indexed read, but each request still boots Laravel (20 to 40 ms of CPU): roughly two cores busy all day saying
"nothing". With a push signal plus a one-minute safety poll it is about 3 requests a second. Push, don't poll.

## Which client (checked on pub.dev, 28 Nov 2026)

Reverb speaks the **Pusher protocol**; Laravel documents only its JavaScript client (Echo + pusher-js). For Flutter:

| Package | Verdict |
| --- | --- |
| `laravel_echo` (0.2.9, 5 years old), `flutter_laravel_echo` | **Don't.** Dart 3 incompatible or Dart 2 only, and their examples use socket.io, which Reverb does not speak. Tutorials from 2020 to 2022 use these. |
| `pusher_channels_flutter` (Pusher's own), `pusher_client` | Native wrappers: Android, iOS, macOS, web. **No Windows desktop.** Fine for a phone-only app. |
| **`dart_pusher_channels`** (1.3.1) | **Chosen.** Pure Dart, all platforms including Windows, private channels with an HTTP auth endpoint and custom headers, mature (40 releases). Reconnects through a `connectionErrorHandler` that calls `refresh()`. Needs `http` 1.x (we had to move the app off `http` 0.13: zero analyzer errors across 67 files, but run the whole suite). |
| `laravel_reverb` (0.7.1), `pusher_reverb_flutter` (0.0.x) | Pure Dart, Reverb-specific, auto-reconnect; newer and small. Worth a look later. |

## The server side

- An event per site on a **private channel**: `new PrivateChannel('facility.'.$id)`, `broadcastAs()` =
  `'facility.changed'`, a tiny payload (which tables, the server time). It is a "sync now" hint, never the records.
- **Channel auth with the device's own token**: `withBroadcasting(... ['middleware' => ['auth:sanctum']])` gives
  `POST /api/broadcasting/auth`; `Broadcast::channel('facility.{id}', fn ($user, $id) => $user instanceof Device &&
  ! $user->isRevoked() && $user->facility_id === $id)`. A device can only join its own site, never a revoked one.
- **Tell the app where to listen** in an endpoint it already calls (`GET devices/me`): `realtime: {key, host, port:
  443, scheme: 'wss'}`. The key is public by the protocol's design (it names the app); never send the secret. Use the
  request's own host, so test and production each point at themselves.
- Behind Apache/nginx, Reverb listens on localhost and the proxy forwards `/app` (WebSocket upgrade). **Check it with
  a real handshake**, not a plain GET (a plain GET answered 500 and looked broken):

  ```bash
  curl -s -o out.txt --max-time 4 -w "%{http_code}" --http1.1 -H "Connection: Upgrade" -H "Upgrade: websocket" \
    -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
    "https://api.example.com/app/$KEY?protocol=7&client=dart&version=1.3.1"
  # 101, and out.txt starts with {"event":"pusher:connection_established",...}
  ```

  (`--max-time` lets curl print the status; killing it with `timeout` prints nothing, because a good socket stays
  open.)

## The client side

```dart
final client = PusherChannelsClient.websocket(
  options: PusherChannelsOptions.fromHost(scheme: 'wss', host: host, key: key, port: 443,
      shouldSupplyMetadataQueries: true, metadata: PusherChannelsOptionsMetadata.byDefault()),
  connectionErrorHandler: (e, trace, refresh) { live.value = false; refresh(); },  // retry, the delay below apart
  minimumReconnectDelayDuration: const Duration(seconds: 10),
);
final channel = client.privateChannel('private-facility.$facilityId',
  authorizationDelegate: EndpointAuthorizableChannelTokenAuthorizationDelegate.forPrivateChannel(
    authorizationEndpoint: Uri.parse('${api}broadcasting/auth'),
    headers: {'Authorization': 'Bearer $deviceToken', 'Accept': 'application/json'}));
client.onConnectionEstablished.listen((_) { channel.subscribeIfNotUnsubscribed(); live.value = true; syncSoon(); });
channel.bind('facility.changed').listen((_) => syncSoon());   // the broadcastAs name, no leading dot
client.connect();
```

- **Sync on every (re)connection**: anything that changed while the socket was down is caught then.
- **Pace the timer by the socket**: listening, sync when signalled, after a local save (drift's `db.tableUpdates()`,
  ignored while a sync pass is writing), and once a minute as a safety net. Not listening: poll as before. The socket
  can only make things better, never worse.
- One connection per site; reconnect when the site, token or server changes (key the running connection by
  `facility|host|key|token`); close it when the device has no site.
- The socket id (`X-Socket-ID`, so the server's `toOthers()` skips the sender) is internal in this package; without it
  the sender gets its own signal and runs one cheap extra pass. Harmless.

## Every situation, and what happens (go through this list for any realtime feature)

| Situation | What happens | Data safe? |
| --- | --- | --- |
| Internet lost, airplane mode | The socket errors; `refresh()` retries every 10 s; the timer falls back to polling (each pass fails fast, the chip says Offline). Work goes on locally. When the network returns: reconnect, sync. | Yes: saves are local first, sent later. |
| App closed, PC shut down, phone killed | The socket closes with the app. Next start: `devices/me`, connect, sync. | Yes. |
| Phone in the background or asleep | Timers pause; the OS may drop the socket. On resume: the error handler reconnects and the connection sync runs. | Yes. |
| Sign out | The device stays a member of its site (its token is the device's, not the person's), so it keeps listening and syncing for the next person. | Yes. |
| Switch to another site on the device | The app restarts on that site; the running connection is keyed by site and token, so it closes and the new site's opens. | Yes, each site its own file. |
| Remove a site from the device | Background sync pauses during the removal; with no site left the connection is closed. | Only offered when nothing is unsent. |
| Device revoked in the console | Channel auth refuses it (403); the next sync gets 401 and the app forgets that site's token only. | Local records stay; unsent work goes out after signing in again. |
| App reinstalled or updated | Windows: the data folder and the keys stay. Android uninstall wipes the app's data: the next sign-in is a first download. | Server holds everything synced; anything unsent on a wiped phone is lost, so sync often. |
| App data deleted, or the database key lost | The file is unreadable (encrypted); it is set aside and the site downloads again (first download). | Same as above. |
| Server or Reverb down | No socket: polling as before, or "Offline" if the API is down too. | Yes. |
| Proxy blocks WebSockets (a facility network) | `live` stays false: polling as before. | Yes. |
| Two signals in a row, or the sender's own signal | The pass is shared (one in flight); an extra pass finds nothing. | Yes. |
| Wrong clock on the device | Signals are not time-based; the sync cursor is the server's clock. | Yes. |

## Checklist

- [ ] Private channel per site, auth with the device token, revoked devices refused (server test).
- [ ] `devices/me` gives key, host, port, scheme; never the secret (server test).
- [ ] WebSocket handshake checked through the proxy (101 + `connection_established`).
- [ ] Client reconnects; syncs on every connection; falls back to polling when not live.
- [ ] Load measured: an idle device's requests per minute in the access log, before and after (ours: 30 a minute
      before, 1 a minute after; a server-side change reached the laptop in 3 s, 28 s after its previous pass).
- [ ] The situations table walked on a real phone and PC (airplane mode, kill, background, sign out, switch, remove).
