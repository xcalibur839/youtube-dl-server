![Docker Stars Shield](https://img.shields.io/docker/stars/nbr23/youtube-dl-server.svg?style=flat-square)
![Docker Pulls Shield](https://img.shields.io/docker/pulls/nbr23/youtube-dl-server.svg?style=flat-square)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](https://raw.githubusercontent.com/nbr23/youtube-dl-server/master/LICENSE)

# youtube-dl-server

Simple Web and REST interface for downloading youtube videos onto a server.
[`starlette`](https://www.starlette.io/) +
[yt-dlp](https://github.com/yt-dlp/yt-dlp) / [`youtube-dl`](https://github.com/rg3/youtube-dl)

Forked from [manbearwiz/youtube-dl-server](https://github.com/manbearwiz/youtube-dl-server).

![screenshot][1]


![screenshot][2]

## Running

For easier deployment, a docker image is available on
[dockerhub](https://hub.docker.com/r/nbr23/youtube-dl-server):

- `nbr23/youtube-dl-server:yt-dlp` or simply `nbr23/youtube-dl-server` to use `yt-dlp`
- `nbr23/youtube-dl-server:youtube-dl` to use `youtube-dl`. Note that the latest release of `youtube-dl` is pretty [outdated](https://github.com/ytdl-org/youtube-dl/releases/tag/2021.12.17).

### Docker CLI

This example uses the docker run command to create the container to run the
app. Note the `-v` argument to specify the volume and its binding on the host.
This directory will be used to output the resulting videos.

```shell
docker run -d --name youtube-dl -p 8080:8080 -v $HOME/youtube-dl:/youtube-dl nbr23/youtube-dl-server:latest
```

OR for yt-dlp:

```shell
docker run -d --name youtube-dl -p 8080:8080 -v $HOME/youtube-dl:/youtube-dl nbr23/youtube-dl-server:yt-dlp
```

### Docker Compose

This is an example service definition that could be put in `docker-compose.yml`.

```yml
  youtube-dl:
    image: "nbr23/youtube-dl-server:latest"
    volumes:
      - $HOME/youtube-dl:/youtube-dl
      - ./config.yml:/app_config/config.yml:ro # Overwrite the container's config file with your own configuration
    ports:
      - 8080:8080
    restart: always
```

### VPN (Private Internet Access)

Some extractors (adult sites in particular) redirect or block requests from
certain regions/IP ranges, independent of anything yt-dlp itself can work
around. To route the container's traffic through a VPN, run it alongside a
[gluetun](https://github.com/qdm12/gluetun) sidecar, which has native PIA
support - youtube-dl-server joins gluetun's network namespace instead of
having one of its own, and every request yt-dlp makes goes out through the
tunnel. This is Docker-only and needs `docker compose` (not plain
`docker run`), since it requires two containers sharing a network stack.

A full example is in [`docker-compose.pia.yml`](docker-compose.pia.yml).
The shape of it:

```yml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    cap_add: [NET_ADMIN]
    devices: ["/dev/net/tun:/dev/net/tun"]
    environment:
      - VPN_SERVICE_PROVIDER=private internet access
      - VPN_TYPE=wireguard
      - OPENVPN_USER=${PIA_USER}
      - OPENVPN_PASSWORD=${PIA_PASS}
      - WIREGUARD_PRIVATE_KEY= # derived from PIA_USER/PIA_PASS, see gluetun's PIA wiki page
      - SERVER_REGIONS=US East
    ports:
      - 8080:8080 # published on gluetun, not on youtube-dl-server

  youtube-dl-server:
    image: nbr23/youtube-dl-server:yt-dlp
    network_mode: "service:gluetun" # shares gluetun's network instead of having its own
    depends_on: [gluetun]
    volumes:
      - $HOME/youtube-dl:/youtube-dl
      - ./config.yml:/app_config/config.yml
```

Because `youtube-dl-server` has no network of its own in this setup, its
`ports:` must move to the `gluetun` service, and it reaches the outside world
(and gets reached) entirely through gluetun's tunnel. See
[gluetun's PIA docs](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/private-internet-access.md)
for the WireGuard key derivation step and the full list of `SERVER_REGIONS`
names. Verify the tunnel is actually up before relying on it:

```shell
docker compose exec gluetun wget -qO- https://ipinfo.io/ip
```

That should print a PIA exit IP, not your own. If you're already running
something like `binhex/arch-delugevpn` for other downloads, note that it
bundles its own VPN client rather than exposing a shared network namespace,
so it isn't something youtube-dl-server can attach to directly - the gluetun
sidecar above gives youtube-dl-server its own independent PIA connection
instead.

Only yt-dlp's own network traffic needs tunneling for most of these cases -
if you just want to hand yt-dlp a proxy without rerouting the whole
container, PIA's SOCKS5 proxy works through the plain [`ydl_options`
passthrough](#ydl_options), no compose changes needed:

```yaml
ydl_options:
  proxy: 'socks5://username:password@proxy-nl.privateinternetaccess.com:1080'
```

(PIA's proxy username/password are separate from your account login - see
PIA's own SOCKS5 documentation for how to generate them, and which proxy
hostnames are currently available.)

### This fork's image

This fork publishes its own image to GHCR on every push to `master`
(`.github/workflows/ci.yml`), built for `linux/amd64`, `linux/arm64`, and
`linux/arm/v7` (matching the platforms the Dockerfile itself already
special-cases, e.g. skipping the `deno` install on 32-bit ARM):

```shell
docker pull ghcr.io/krakupkiwi/youtube-dl-server:latest
```

Use it anywhere the examples above reference `nbr23/youtube-dl-server` -
same volumes, same config, same tags (`:latest`/`:yt-dlp`).

### Updating

There's no in-app auto-update; pull the new image and recreate the
container:

```shell
docker pull ghcr.io/krakupkiwi/youtube-dl-server:latest
docker compose up -d   # or: docker stop/rm + docker run again for the CLI form
```

For unattended updates, a tool like
[Watchtower](https://containrrr.dev/watchtower/) can poll and recreate the
container automatically. On Unraid, the Docker tab's "Check for Updates" /
"Update" action does the same thing for containers added as templates.

## Configuration

Configuration is done through the config.yml file at the root of the project.

An alternate configuration path or file path can be forced by setting the environment
variable `YDL_CONFIG_PATH`:

```shell
export YDL_CONFIG_PATH=/var/local/youtube-dl-server/config.yml
```

In the above case, if `/var/local/youtube-dl-server/config.yml` does not exist, it will be created with the default options.

```shell
export YDL_CONFIG_PATH=/var/local/youtube-dl-server/
```

In the above case, if `/var/local/youtube-dl-server/config.yml` does not exist, it will be created with the default options as well.

### ydl_server options

| Key | Default | Description |
|-----|---------|-------------|
| `port` | `8080` | Port to listen on |
| `host` | `0.0.0.0` | IP to bind to |
| `site_title` | `null` | Optional browser tab title, navbar brand, and main-page heading; unset or empty preserves each original label. Restart the server after changing it |
| `metadata_db_path` | `/youtube-dl/.ydl-metadata.db` | Path to the SQLite job database |
| `output_playlist` | `/youtube-dl/%(playlist_title)s [%(playlist_id)s]/%(title)s.%(ext)s` | Output template for playlists and multi-URL jobs |
| `max_log_entries` | `100` | Maximum number of job history entries to keep |
| `default_format` | `video/best` | Default format pre-selected in the UI |
| `download_workers_count` | `2` | Number of parallel download worker threads |
| `forwarded_allow_ips` | `None` | Comma-separated list of IPs to trust proxy headers from (passed to uvicorn) |
| `proxy_headers` | `True` | Trust `X-Forwarded-Proto`, `X-Forwarded-For`, `X-Forwarded-Port` headers (passed to uvicorn) |
| `debug` | `False` | Enable debug mode |
| `api_key` | unset | Require this key on every `/api/` request (see [API key protection](#api-key-protection)) |

Minimum required configuration:

```yaml
ydl_server:
  port: 8080
  host: 0.0.0.0
  metadata_db_path: '/youtube-dl/.ydl-metadata.db'

ydl_options:
  output: '/youtube-dl/%(title)s [%(id)s].%(ext)s'
  cache-dir: '/youtube-dl/.cache'
```

### API key protection

By default, every `/api/` route is open to anyone who can reach the server -
the intended deployment is behind a reverse proxy that already handles access
control (see [HTTPS](#https) below). For deployments that aren't proxied,
set `api_key` to require a matching key on every API request:

```yaml
ydl_server:
  api_key: 'some-long-random-string'
```

With this set, the bundled web UI still works: visit the site once with
`?api_key=some-long-random-string` in the URL, and the frontend stores the
key in `localStorage`, strips it from the visible address bar, and attaches
it to every subsequent request (as an `X-API-Key` header for regular calls,
and as an `api_key` query parameter for the live-updates connection and file
download links, since browsers don't let those set custom headers). Calling
the API directly (scripts, curl) needs the same header or query parameter:

```shell
curl -H 'X-API-Key: some-long-random-string' http://{{host}}:8080/api/info
```

### Adult content filtering

`ydl_options.age-limit` filters both the extractor list (`/api/extractors`) and
actual downloads down to content rated at or below the given age:

```yaml
ydl_options:
  age-limit: 18
```

Leave it unset for no restriction. This is easiest to manage from the in-app
**Settings** page (`#/settings`) rather than editing `config.yml` directly -
it edits the file in place (comments and formatting elsewhere survive) and
applies immediately, no restart required.

### ydl_options

Additional yt-dlp/youtube-dl parameters can be set in the `ydl_options` section. Add
parameters by removing the leading `--` from the flag name.

For example, to write subtitles in spanish, the yt-dlp command would be:

`yt-dlp --write-sub --sub-lang es URL`

Which translates to:

```yaml
ydl_options:
  output: '/youtube-dl/%(title)s [%(id)s].%(ext)s'
  cache-dir: '/youtube-dl/.cache'
  write-sub: True
  sub-lang: es
```

The download directory listed in the Finished tab is derived from the static part of
`output`, before the first template variable. If `output` is relative, it is resolved
against `paths` (yt-dlp's `--paths`, `home:` prefix supported):

```yaml
ydl_options:
  paths: '/youtube-dl'
  output: '%(title)s [%(id)s].%(ext)s'
```

`output` must resolve to a directory other than the filesystem root, otherwise the server
refuses to start.

### Authentication / cookies for gated videos

Since any yt-dlp flag can be set via `ydl_options` (see above), gated or
private videos that require a logged-in session can be downloaded by pointing
yt-dlp at a cookies file:

```yaml
ydl_options:
  cookies: '/app_config/cookies.txt'
```

Export the cookies file from your browser (e.g. with a "Get cookies.txt"
extension) and mount it alongside your `config.yml` in the `/app_config`
volume. `cookies-from-browser` also works the same way if the server has
access to a real browser profile, though this is uncommon in a container:

```yaml
ydl_options:
  cookies-from-browser: chrome
```

This is a credential for the *video sources* yt-dlp downloads from, not a
login for the youtube-dl-server web UI itself — put youtube-dl-server behind
a reverse proxy (see [HTTPS](#https) below) if you need to restrict who can
reach it.

### Profiles

Profiles are named configuration sets selectable in the UI. Each profile can
override any `ydl_options`.

```yaml
profiles:
  podcast:
      name: 'Audio Podcasts'
      ydl_options:
        output: '/youtube-dl/Podcast/%(title)s [%(id)s].%(ext)s'
        format: bestaudio/best
        write-thumbnail: True
        embed-thumbnail: True
        add-metadata: True
        audio-quality: 0
        extract-audio: True
        audio-format: mp3
  philosophy_lectures:
      name: 'Philosophy Lectures'
      ydl_options:
        output: '/youtube-dl/Lectures/Philosophy/%(title)s [%(id)s].%(ext)s'
        write-thumbnail: True
        embed-thumbnail: True
        add-metadata: True
        verbose: True
```

![screenshot][3]

### Option groups (aliases)

Aliases are named `ydl_options` sets that can be shared between profiles, avoiding
duplication. A profile pulls them in with `use:`, and they can also be ticked
individually in the Advanced Options section of the download form.

```yaml
aliases:
  mp3:
      name: 'MP3 audio'      # optional label shown in the UI, defaults to the key
      ydl_options:
        format: bestaudio/best
        extract-audio: True
        audio-format: mp3
  thumbnails:
      name: 'Thumbnails'
      ydl_options:
        write-thumbnail: True
        embed-thumbnail: True
  podcast_base:
      ui: False              # building block: usable with `use:`, hidden from the UI
      use: [mp3, thumbnails]
      ydl_options:
        add-metadata: True

profiles:
  podcast:
      name: 'Audio Podcasts'
      use: [podcast_base]
      ydl_options:
        output: '/youtube-dl/Podcast/%(title)s [%(id)s].%(ext)s'
```

Aliases may reference other aliases through their own `use:` list. Options are merged
in this order, the last one winning: `ydl_options`, then the profile (its aliases in
listed order, then its own `ydl_options`), then the aliases selected in the form.
Unknown aliases and recursive `use:` chains are rejected when the configuration is
loaded, at server startup.

### Download folders

Named subfolders selectable as the download destination from the download form
(and the Quick Download modal), so a video can be sorted into place at queue time
instead of being moved by hand afterward:

```yaml
download_folders:
  - Movies
  - TV Shows
  - Music
  - Kids
```

Picking one nests the file under whatever `output` (global, or a profile's
override) would otherwise have used - e.g. with the default `output` above,
choosing "Movies" downloads to `/youtube-dl/Movies/%(title)s [%(id)s].%(ext)s`.
It composes with profiles, playlists, and title overrides: whichever output
template ends up selected, the chosen folder is nested into it as the last
step. Folder names can't contain `/`, `\`, or `,`.

An entry can also give itself its own absolute path instead of nesting under
the default output - useful on Unraid/Docker setups where each media type is
bind-mounted to its own separate container path rather than everything
living under one shared directory:

```yaml
download_folders:
  - Movies                # nests under the default output, as above
  - name: Concerts
    path: /concerts        # a separately mounted path - downloads go straight there
```

Picking "Concerts" downloads directly to `/concerts/%(title)s [%(id)s].%(ext)s`,
not underneath the default output directory at all. `path` must be absolute
and contain no `..` segments; it isn't cross-checked against `ydl_options`,
so make sure it's a real, currently-mounted directory or the download will
fail. Note that files sent to a folder with an explicit `path` currently
don't show up in the **Finished Files** page (which only browses the default
output directory) - only the download destination is affected.

Rather than typing (and mistyping) container paths from memory, the Settings
page's "Browse…" button lets you navigate the server's real filesystem and
pick an existing directory - backed by `GET /api/browse-dirs`, which lists
subdirectories of a given path (defaulting to the filesystem root). In a
container this only ever sees what's actually mounted in, nothing on the
host beyond that; it sits behind the same access control as every other
`/api/` route (see [API key protection](#api-key-protection)).

This list - and the [age restriction](#adult-content-filtering) above - can
also be managed from the in-app **Settings** page (`#/settings`) instead of
editing `config.yml` by hand; changes there take effect immediately, no
restart required.

### Per-extractor options

Different sites sometimes need different yt-dlp options - subtitles only make sense
on some extractors, a site might need a specific format fallback, etc. `extractor_options`
sets `ydl_options` that only apply when yt-dlp's matched extractor for the URL equals
the given key (case-insensitive; this is yt-dlp's own `extractor` field, e.g. `youtube`,
`twitter`, `generic`):

```yaml
extractor_options:
  youtube:
    ydl_options:
      write-thumbnail: True
  twitter:
    ydl_options:
      format: best
```

These act as defaults: they're resolved after the video's metadata is fetched (so the
server knows which extractor matched), and only fill in keys not already set by the
selected format, profile, or aliases - an explicit choice made elsewhere always wins.

### Recipes

Common `ydl_options` combinations, added as profiles or aliases (see above) so they're
selectable from the UI:

**Force H264 video + AAC audio in an MP4 container** - useful for players/devices
(older smart TVs, some NAS media players) that can't handle VP9/AV1 video or Opus
audio, which yt-dlp otherwise prefers by default:

```yaml
aliases:
  compatible_mp4:
    name: 'Compatible MP4 (H264/AAC)'
    ydl_options:
      format: 'bestvideo[vcodec^=avc1]+bestaudio[acodec^=mp4a]/best[vcodec^=avc1][ext=mp4]'
      merge-output-format: mp4
```

**Cap resolution** - avoid pulling a 4K stream when 1080p is plenty:

```yaml
aliases:
  max_1080p:
    name: 'Max 1080p'
    ydl_options:
      format: 'bestvideo[height<=1080]+bestaudio/best[height<=1080]'
```

**Always write and embed subtitles:**

```yaml
aliases:
  subtitles:
    name: 'Subtitles'
    ydl_options:
      write-sub: True
      write-auto-sub: True
      embed-subs: True
      sub-lang: en
```

**Throttle download bandwidth** - handy on a connection shared with other traffic:

```yaml
ydl_options:
  limit-rate: 5M   # bytes/sec; accepts yt-dlp's usual K/M/G suffixes
```

**Extract audio only, as a specific format** - the built-in `mp3` alias in the
default config already covers this; the same shape works for other formats:

```yaml
aliases:
  flac:
    name: 'FLAC audio'
    ydl_options:
      format: bestaudio/best
      extract-audio: True
      audio-format: flac
```

**Plex-friendly layout** - the built-in `plex` alias in the default config puts
each video in its own folder named after the video (Plex's expected layout for
local movies), and writes the thumbnail as a same-named `.jpg` next to it,
which Plex picks up automatically as poster art (needs ffmpeg, already bundled
in this project's Docker image):

```yaml
aliases:
  plex:
    name: 'Plex (movie folder + poster)'
    ydl_options:
      output: '/youtube-dl/%(title)s [%(id)s]/%(title)s [%(id)s].%(ext)s'
      write-thumbnail: True
      convert-thumbnails: jpg
```

Match the `/youtube-dl/` prefix to your own `ydl_options.output` base path if
you've changed it - like the `podcast` profile above, this fully replaces
`output` rather than appending to it. It composes with
[download folders](#download-folders): picking both `plex` and, say, a
"Movies" folder nests as `Movies/Title [id]/Title [id].mp4` +
`Movies/Title [id]/Title [id].jpg`.

## Python

Requires Python ^3.8.

Install dependencies:

```shell
pip install -r requirements.txt
```

Build the frontend:

```shell
cd front && npm install && npm run build
```

Run the server:

```shell
python3 -u ./youtube-dl-server.py
```

To force a specific yt-dlp/youtube-dl fork, set the `YOUTUBE_DL` environment variable:

```shell
YOUTUBE_DL=yt-dlp python3 -u ./youtube-dl-server.py
```

The following environment variables are used for version display in the UI:

| Variable | Description |
|----------|-------------|
| `YOUTUBE_DL` | The yt-dlp/youtube-dl module to use (`yt-dlp`, `youtube-dl`) |
| `YDLS_VERSION` | Version string shown in the server info endpoint |
| `YDLS_RELEASE_DATE` | Release date string shown in the server info endpoint |

## Usage

### Web UI

Navigate to `http://{{host}}:8080/` and enter the URL to download.

### REST API

#### Queue a download

```shell
curl -X POST \
  -H 'Content-Type: application/json' \
  --data-raw '{"url": "{{URL}}", "format": "video/best"}' \
  http://{{host}}:8080/api/downloads
```

Accepted body fields:

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | Single URL to download |
| `urls` | array | Multiple URLs to download as one job |
| `format` | string | Format string (see formats below) |
| `profile` | string | Profile name from config |
| `aliases` | array | Alias names from config to apply on top of the profile |
| `audio_format` | string | Audio format (e.g. `mp3`, `aac`) |
| `force_generic_extractor` | bool | Force use of the generic extractor |
| `extra_params` | object | Extra parameters; `title` key overrides the output filename |

Available format values:

| Value | Description |
|-------|-------------|
| `video/best` | Best video+audio (default) |
| `video/bestvideo` | Best video quality |
| `video/mp4` | MP4 |
| `video/webm` | WebM |
| `video/mkv` | Matroska |
| `video/avi` | AVI |
| `video/flv` | Flash Video |
| `video/ogg` | Ogg |
| `bestaudio/best` | Best audio |
| `audio/mp3` | MP3 |
| `audio/aac` | AAC |
| `audio/flac` | FLAC |
| `audio/m4a` | M4A |
| `audio/opus` | Opus |
| `audio/vorbis` | Vorbis |
| `audio/wav` | WAV |

#### List jobs

```shell
curl http://{{host}}:8080/api/downloads
```

Query parameters:

| Parameter | Description |
|-----------|-------------|
| `status` | Filter by status: `Running`, `Completed`, `Failed`, `Pending`, `Aborted` |
| `show_logs` | Include log output in response, `1` (default) or `0` |

#### Queue statistics

```shell
curl http://{{host}}:8080/api/downloads/stats
```

#### Clean old job entries

Removes completed and failed entries beyond `max_log_entries`:

```shell
curl -X POST http://{{host}}:8080/api/downloads/clean
```

#### Purge all job history

```shell
curl -X DELETE http://{{host}}:8080/api/downloads
```

#### Fetch metadata without downloading

```shell
curl -X POST \
  -H 'Content-Type: application/json' \
  --data-raw '{"url": "{{URL}}"}' \
  http://{{host}}:8080/api/metadata
```

#### Check a video's availability and quality

A cheap alternative to `/api/metadata` for scripts that just need to know
whether a link still resolves and what quality is available, without the
full formats/thumbnails/subtitles payload - handy for e.g. periodically
verifying a cookies file hasn't expired.

```shell
curl -X POST \
  -H 'Content-Type: application/json' \
  --data-raw '{"url": "{{URL}}"}' \
  http://{{host}}:8080/api/check
```

```json
{
  "success": true,
  "is_playlist": false,
  "id": "...",
  "title": "...",
  "uploader": "...",
  "duration": 2341,
  "is_live": false,
  "availability": "public",
  "extractor": "youtube",
  "best_format": {"format_id": "399", "ext": "mp4", "resolution": "1920x1080", "filesize": 40184962}
}
```

#### Stop a job

```shell
curl -X POST http://{{host}}:8080/api/jobs/{{job_id}}/stop
```

#### Retry a job

```shell
curl -X POST http://{{host}}:8080/api/jobs/{{job_id}}/retry
```

#### Delete a job entry

```shell
curl -X DELETE http://{{host}}:8080/api/jobs/{{job_id}}
```

#### List downloaded files

```shell
curl http://{{host}}:8080/api/finished
```

#### Delete a downloaded file

```shell
curl -X DELETE http://{{host}}:8080/api/finished/{{filename}}
```

#### Server info

```shell
curl http://{{host}}:8080/api/info
```

#### Available formats

```shell
curl http://{{host}}:8080/api/formats
```

#### Supported extractors

```shell
curl http://{{host}}:8080/api/extractors
```

### Bookmarklet

Add the following bookmarklet to your bookmark bar to send the current page URL
to your youtube-dl-server instance.

#### HTTPS

If your youtube-dl-server is served through HTTPS (behind a reverse proxy
handling HTTPS for example), you can use the following bookmarklet:

```javascript
javascript:fetch("https://${host}/api/downloads",{body:JSON.stringify({url:window.location.href,format:"video/best"}),method:"POST",headers:{'Content-Type':'application/json'}});
```

#### Plain HTTP

If you are hosting it without HTTPS, the previous bookmarklet will likely be
blocked by your browser (mixed content when used on HTTPS sites).

Instead, you can use the following bookmarklet:

```javascript
javascript:(function(){document.body.innerHTML += '<form name="ydl_form" method="POST" action="http://${host}/api/downloads"><input name="url" type="url" value="'+window.location.href+'"/></form>';document.ydl_form.submit()})();
```

## Notes

### Extra format support

`ffmpeg` is required for format conversion and audio extraction in some
scenarios.

## Additional references

* [ansible-role-youtubedl-server](https://github.com/nbr23/ansible-role-youtubedl-server)
* [ytdl-k8s](https://github.com/droopy4096/ytdl-k8s) - `youtube-dl-server` Helm chart (uses `youtube-dl-server` image for kubernetes deployment)
* [starlette](https://www.starlette.io/)
* [youtube-dl](https://github.com/rg3/youtube-dl)
* [yt-dlp](https://github.com/yt-dlp/yt-dlp)

[1]:youtube-dl-server.png
[2]:youtube-dl-server-logs.png
[3]:youtube-dl-server-profiles.png
