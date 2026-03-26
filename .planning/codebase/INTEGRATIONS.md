# External Integrations

**Analysis Date:** 2026-03-26

## APIs & External Services

**Game Services:**
- OpenRA master services on `master.openra.net` for public server discovery, dedicated-server advertisement, game news, version checks, and AppImage update metadata.
  - Integration method: HTTP GET/POST via `OpenRA.Game/Support/HttpClientFactory.cs`.
  - Auth: no repo-managed runtime secret detected for these endpoints.
  - Endpoints used: `https://master.openra.net/games`, `https://master.openra.net/ping`, `https://master.openra.net/gamenews`, and `https://master.openra.net/versioncheck` in `OpenRA.Mods.Common/WebServices.cs`; `https://master.openra.net/appimagecheck.zsync` in `packaging/linux/buildpackage.sh`.
- OpenRA Resource Center on `resource.openra.net` for map metadata lookup and remote map downloads.
  - Integration method: HTTP GET requests built from `OpenRA.Mods.Common/WebServices.cs`.
  - Auth: none detected.
  - Endpoints used: `https://resource.openra.net/map/` from `OpenRA.Game/Map/MapCache.cs`, `OpenRA.Game/Map/MapPreview.cs`, and `OpenRA.Mods.Common/ServerTraits/LobbyCommands.cs`.
- OpenRA forum profile service on `forum.openra.net` for linked player profiles, badges, and public-key lookup.
  - Integration method: HTTP GET to `https://forum.openra.net/openra/info/{fingerprint}`.
  - Auth: custom fingerprint/public-key flow; the client stores the private key locally and the server verifies signatures using the fetched public key.
  - Files: `OpenRA.Game/PlayerDatabase.cs`, `OpenRA.Game/LocalPlayerProfile.cs`, `OpenRA.Mods.Common/Widgets/Logic/RegisteredProfileTooltipLogic.cs`, `OpenRA.Game/Server/Server.cs`.
- itch.io API for optional player-name lookup in itch-distributed builds.
  - Integration method: bearer-authenticated REST call to `https://itch.io/api/1/jwt/me`.
  - Auth: `ITCHIO_API_KEY` process environment variable.
  - Files: `OpenRA.Mods.Common/ItchIntegration.cs`.

**Desktop Platform Integrations:**
- Discord Rich Presence for official mods.
  - SDK/Client: `DiscordRichPresence` in `OpenRA.Mods.Common/OpenRA.Mods.Common.csproj`.
  - Auth: local Discord client plus installer/launcher URI registration; no OAuth flow is implemented in-repo.
  - App IDs: `mods/ra/mod.yaml`, `mods/cnc/mod.yaml`, `mods/d2k/mod.yaml`, and `mods/ts/mod.yaml`.
  - Files: `OpenRA.Mods.Common/DiscordService.cs`, `packaging/linux/openra.desktop.discord.in`, `packaging/linux/openra-mimeinfo.xml.discord.in`, `packaging/macos/Info.plist.in`.
- Steam install discovery for importing original game assets from local Steam installs.
  - Integration method: local Steam registry/library inspection, not a web API.
  - Auth: none.
  - Content definitions: `mods/cnc-content/installer/steam.yaml`, `mods/ra-content/installer/steam.yaml`, and `mods/ts-content/installer/steam.yaml`.
  - Files: `OpenRA.Mods.Common/Installer/SourceResolvers/SteamSourceResolver.cs`.
- UPnP/NAT-PMP device discovery and LAN game beacons for local-network hosting/discovery.
  - SDK/Client: `Mono.NAT` and `rix0rrr.BeaconLib`.
  - Auth: none.
  - Files: `OpenRA.Game/Network/Nat.cs`, `OpenRA.Mods.Common/ServerTraits/MasterServerPinger.cs`, `OpenRA.Mods.Common/Widgets/Logic/ServerListLogic.cs`.

**Distribution & Build Services:**
- GitHub-hosted downloads are used for the GeoIP database, Linux AppImage tooling, Windows `rcedit`, and release artifact uploads.
  - Integration method: `curl`, `wget`, GitHub CLI `gh`, and GitHub Actions.
  - Auth: `GITHUB_TOKEN` in release workflows where uploads are required.
  - Files: `fetch-geoip.sh`, `make.ps1`, `packaging/linux/buildpackage.sh`, `packaging/windows/buildpackage.sh`, `.github/workflows/packaging.yml`.
- NuGet package restore is required for all `PackageReference` dependencies.
  - Integration method: `dotnet`/NuGet restore.
  - Auth: public feed by default.
  - Feed: `https://api.nuget.org/v3/index.json` is explicitly added in `.github/workflows/ci.yml`.

## Data Storage

**Databases:**
- None detected.
  - Connection: Not applicable.
  - Client: Not applicable.
  - Migrations: Not applicable.

**File Storage:**
- Local filesystem storage under `Platform.SupportDir` for settings, logs, replays, downloaded maps, imported game assets, and auth profiles.
  - Files: `OpenRA.Game/Platform.cs`, `OpenRA.Game/Settings.cs`, `OpenRA.Game/LocalPlayerProfile.cs`, `OpenRA.Game/Map/MapPreview.cs`, `OpenRA.Server/Program.cs`.
- Remote package mirror lists hosted on `www.openra.net` for content installers.
  - Files: `mods/cnc-content/installer/downloads.yaml`, `mods/d2k-content/installer/downloads.yaml`, `mods/ra-content/installer/downloads.yaml`, `mods/ts-content/installer/downloads.yaml`.
- Bundled binary support files are copied into installs locally, including `global mix database.dat` and the downloaded `IP2LOCATION-LITE-DB1.IPV6.BIN.ZIP`.
  - Files: `packaging/functions.sh`, `fetch-geoip.sh`, `make.ps1`.

**Caching:**
- No external cache service is used.
  - Local cache files include `news.yaml` under the support directory from `OpenRA.Mods.Common/Widgets/Logic/MainMenuLogic.cs`.
  - Remote map downloads are stored in user map directories managed through `OpenRA.Game/Map/MapPreview.cs`.

## Authentication & Identity

**Auth Provider:**
- Custom OpenRA player-profile authentication backed by the forum profile endpoint.
  - Implementation: `OpenRA.Game/LocalPlayerProfile.cs` generates/stores RSA keys locally, and `OpenRA.Game/Server/Server.cs` verifies connection signatures against the public key fetched from `forum.openra.net`.
  - Token storage: local auth-profile file, defaulting to `player.oraid` via `OpenRA.Game/Settings.cs`.
  - Session management: per-connection fingerprint + signature handshake on dedicated servers.

**OAuth Integrations:**
- None detected.

**Other Identity Services:**
- itch.io user-name lookup is optional and uses `ITCHIO_API_KEY` in `OpenRA.Mods.Common/ItchIntegration.cs`.
- Discord join requests use local app IDs and URI schemes in `OpenRA.Mods.Common/DiscordService.cs` and the packaging templates under `packaging/linux/` and `packaging/macos/`.

## Monitoring & Observability

**Error Tracking:**
- None detected. No Sentry, Rollbar, Crashpad, or similar hosted error tracker is referenced in the codebase.

**Analytics:**
- Optional anonymous system information is appended to game-news requests when the player opts in.
  - Fields include OS, architecture, runtime, OpenGL version, window metrics, and language in `OpenRA.Mods.Common/Widgets/Logic/SystemInfoPromptLogic.cs`.
  - The data is sent as query parameters to the game-news endpoint from `OpenRA.Mods.Common/Widgets/Logic/MainMenuLogic.cs`.

**Logs:**
- Local file logs and stdout only.
  - Dedicated server log files are configured in `OpenRA.Server/Program.cs`.
  - Client crash dialogs point users to support-directory logs in `launch-game.sh`, `packaging/linux/openra.in`, `packaging/linux/openra.appimage.in`, `packaging/macos/launcher.m`, and `OpenRA.WindowsLauncher/Program.cs`.

## CI/CD & Deployment

**Hosting:**
- GitHub Releases is the primary binary distribution target.
  - Deployment: tag-driven packaging from `.github/workflows/packaging.yml` for `release-*`, `playtest-*`, and `devtest-*`.
  - Environment vars: `GITHUB_TOKEN` plus signing/notarization secrets in GitHub Actions.
- `docs.openra.net` and the OpenRA wiki are updated by pushing generated files to `openra/docs` and `openra/openra.wiki`.
  - Files: `.github/workflows/documentation.yml`, `packaging/format-docs.py`.
- itch.io is a secondary package distribution target.
  - Files: `.github/workflows/itch.yml`, `packaging/.itch.toml`.

**CI Pipeline:**
- GitHub Actions.
  - Workflows: `.github/workflows/ci.yml`, `.github/workflows/packaging.yml`, `.github/workflows/documentation.yml`, `.github/workflows/itch.yml`.
  - Secrets: `GITHUB_TOKEN`, `DOCS_TOKEN`, `BUTLER_CREDENTIALS`, `MACOS_DEVELOPER_IDENTITY`, `MACOS_DEVELOPER_CERTIFICATE_BASE64`, `MACOS_DEVELOPER_CERTIFICATE_PASSWORD`, `MACOS_DEVELOPER_USERNAME`, `MACOS_DEVELOPER_PASSWORD`, `SIGNPATH_API_TOKEN`, and `SIGNPATH_ORGANISATION_ID`.
- Windows signing uses SignPath from `.github/workflows/packaging.yml`.
- macOS notarization uses Apple `notarytool` from `packaging/macos/buildpackage.sh`.

## Environment Configuration

**Development:**
- Required env vars: none for a basic local build/run path.
- Optional env vars: `ENGINE_DIR`, `MOD_SEARCH_PATHS`, `OPENRA_DISPLAY_SCALE`, `OPENRA_DESKTOP_FILENAME`, `XDG_CONFIG_HOME`, `GDK_SCALE`, `ITCHIO_API_KEY`, and `TREAT_WARNINGS_AS_ERRORS`.
- Secrets location: GitHub Actions repository secrets for CI/release workflows; no `.env` files are detected in the repo.
- Mock/stub services: none built in. Online behaviors can be disabled through settings in `OpenRA.Game/Settings.cs`, including `FetchNews`, `CheckVersion`, `AllowDownloading`, `QueryMapRepository`, `AdvertiseOnline`, `AdvertiseOnLocalNetwork`, `EnableGeoIP`, and `EnableDiscordService`.

**Staging:**
- Not detected as a separate hosted environment.
- Release channels are tag-based (`playtest`, `release`, `pkgtest`) rather than backed by a dedicated staging backend; see `packaging/linux/buildpackage.sh` and `.github/workflows/packaging.yml`.

**Production:**
- Secrets management: GitHub Actions secrets and process environment variables only.
- Failover/redundancy: not defined in-repo for `master.openra.net`, `resource.openra.net`, `forum.openra.net`, `itch.io`, or GitHub-hosted package feeds.

## Webhooks & Callbacks

**Incoming:**
- No HTTP webhooks are detected.
- Local callback schemes are registered for Discord join handling in `packaging/linux/openra.desktop.discord.in`, `packaging/linux/openra-mimeinfo.xml.discord.in`, and `packaging/macos/Info.plist.in`, then consumed by `OpenRA.Mods.Common/DiscordService.cs`.

**Outgoing:**
- Dedicated servers advertise via POST to the master server from `OpenRA.Mods.Common/ServerTraits/MasterServerPinger.cs`.
- Clients issue GET requests for server lists, version checks, news, profile data, map metadata/downloads, and itch user data from `OpenRA.Mods.Common/WebServices.cs`, `OpenRA.Mods.Common/Widgets/Logic/ServerListLogic.cs`, `OpenRA.Mods.Common/Widgets/Logic/MainMenuLogic.cs`, `OpenRA.Game/PlayerDatabase.cs`, `OpenRA.Game/LocalPlayerProfile.cs`, `OpenRA.Game/Map/MapCache.cs`, `OpenRA.Game/Map/MapPreview.cs`, and `OpenRA.Mods.Common/ItchIntegration.cs`.
- GitHub release uploads, wiki/docs pushes, SignPath signing requests, and Butler pushes are defined in the workflows under `.github/workflows/`.

---

*Integration audit: 2026-03-26*
