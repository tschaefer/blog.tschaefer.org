---
title: "Beets Store - A plugin for the music geek's media organizer"
description: "Beets Store turns your beets music library into a web frontend.
A single-page app feel powered by HTMX keeps the player alive across every
view, native OS controls work out of the box, and WebSocket toasts tell you
the moment an album zip is ready."
author: "Tobias Schäfer"
date: 2026-03-30T12:00:00+02:00
draft: false
toc: false
images:
  - "https://blog.tschaefer.org/images/beets-view-albums.png"
  - "https://blog.tschaefer.org/images/beets-view-download.png"
tags:
  - beets
  - music
  - python
  - web-frontend
  - htmx
  - flask
  - websocket
---

If you organise your music with [beets](http://beets.radbox.org/) you already
have a well-tagged, neatly structured library on disk. *Beets Store* turns that
library into a browser-based music player — albums, artists, tracks, playback,
and downloads - without standing up a separate media server.

![Beets Store Albums View](/images/beets-view-albums.png)

## A single-page app without a JavaScript framework

Every navigation link and search form in the app is carried by HTMX. Clicking
*Albums*, *Artists*, or *Tracks*, or submitting a search, swaps only the page
content and simultaneously patches the navbar search form via an out-of-band
swap. The address bar updates, the browser history is intact, but the page
never fully reloads.

```html
<body hx-boost="true"
      hx-select="#page-content"
      hx-target="#page-content"
      hx-select-oob="#navbar-form"
      hx-push-url="true"
      hx-swap="innerHTML">
```

A slim progress bar sits fixed at the very top of the viewport, above the
navigation bar. It animates from left to right on every navigation request and
fades out on completion, giving the same visual feedback you would expect from
a native app transition.

The approach means there is no client-side router and no virtual DOM. Just HTML
rendered on the server and partial swaps in the browser.

## The player survives every page navigation

The player bar is part of DOM and is therefore never torn down. It holds a
single `<audio>` element and its controls. When HTMX swaps a new page
in, javascript parses the incoming response for an embedded album data JSON
block and loads the info to register the new track list - without touching the
currently playing audio.

The result is that you can browse from album to album, search for tracks,
or jump to the artists view, and whatever is playing keeps playing. The
album artwork and track title in the player bar stay accurate, and if the
currently playing album happens to be the page you are on, a small animated
heart-pulse marker appears next to the active track in the tracklist.

Player state — track index, position, album metadata — is persisted in the
browser store every ten seconds and on pause, so a page reload or a cold
browser start resumes exactly where you left off.

## The player progress bar

Directly above the player controls, at the very top of the fixed bottom bar,
sits another progress bar. It fills in real time as the track plays,
driven by the audio timeupdate event. Clicking anywhere on it seeks to the
corresponding position in the track. Together with the cover art thumbnail,
track title, and artist/album label right next to it, the bar makes the player
a compact but complete widget that is always visible no matter which page you
are on.

## Native OS audio controls

The player registers a full [Media Session API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Session_API)
handler set. Title, artist, album name, and cover art are pushed into the
browser's media metadata on every track load.

```javascript
navigator.mediaSession.metadata = new MediaMetadata({
  title: t.title,
  artist: playerAlbum.albumartist,
  album: playerAlbum.album,
  artwork: playerAlbum.artUrl ? [{ src: playerAlbum.artUrl }] : [],
});
```

Play, pause, previous track, and next track are all wired to the native
handlers. That means the lock screen controls on a phone, a Bluetooth headset,
or the media keys on a keyboard all work without any extra configuration.

## Cross-tab awareness

A browser broadcast channel is used to coordinate between open tabs. When
playback starts in one tab, every other tab receives a `stop` message and
pauses immediately. There is no fighting and audio glitch between two instances
of the app.

## Album downloads with WebSocket notifications

Zipping an entire album with cover art can take a moment. Rather than blocking
the UI, the download flow is fully asynchronous:

1. Clicking the download button fires a JSON request to the server.
2. The server enqueues a Redis Queue job and returns `202 Accepted` together
   with the job ID.
3. The client emits a `watch_job` event over the WebSocket connection.
4. The server's `Dispatcher` listens on Redis pub/sub for `rq:job:succeeded`
   and `rq:job:failed` events. When one arrives it emits a `download_ready`
   or `download_failed` event to the matching WebSocket room.
5. The browser receives the event and shows a toast in the bottom-right corner.
   A success toast contains the album art, the artist · album label, and a
   direct download link. A failure toast surfaces in red.

![Beets Store Download View](/images/beets-view-download.png)

If the zip was already built within the last twelve hours the server returns
`200` with the URL directly and the download starts immediately without
involving the queue at all.

The WebSocket connection state is reflected in the navbar brand icon: a solid
heart while connected, a cracked heart when the connection is lost.

## Last.fm scrobbling

If Last.fm credentials are configured, an icon appears in the player bar. A
dimmed icon signals that no session exists yet; clicking it starts the OAuth
flow. Once authenticated, `now_playing` is sent when a track starts and
`scrobble` is sent when it finishes - both as fire-and-forget XHR requests in
the background.

## Browsing the library

Three views cover the library from different angles.

- **Albums** - a cover art grid, searchable by album title.
- **Artists** - each artist's albums grouped together, searchable by name.
- **Tracks** - a flat table of every track, searchable by title.

All three respond to both direct GET requests and HTMX-boosted POST searches
without a page reload. Each view is also available as a JSON API by sending
an `Accept: application/json` header, so the same endpoints serve both the
browser and any external tooling.

## Conclusion

Beets Store is deliberately small. HTMX does the heavy lifting for navigation,
the Media Session API hands off to the operating system, and WebSockets close
the feedback loop on downloads. The result feels more complete than a simple
static frontend but costs almost nothing in framework weight.

Source: [github.com/tschaefer/beets-store](https://github.com/tschaefer/beets-store)
