# Bixberry app reconnaissance (public evidence + static Android analysis)

**Checked:** 2026-10-31  
**Purpose:** Separate vendor claims from static evidence and identify the next safe experiment. This is preliminary static APK analysis plus a public-documentation review; no code has been executed.

## Executive summary

Bixberry presents itself as an opt-in residential connection-proxy network: a participant app stays active in the background; a partner's current-web lookup is routed through a participant's connection; the participant earns a balance from requests, connected time, and referrals; redemptions are gift cards. In practical terms, this is closer to a consented bandwidth/proxy-sharing product with an account/rewards layer than an AI model running locally on a phone.

That description is **the vendor's published explanation**, not an independently verified account of production traffic. The Android APK split set is present on the updated `origin/main` as `bixberry_apk/`; I fetched those blobs from Git history into `/tmp` for inspection without switching branches or adding binaries to this experiment branch. There is still no app source, backend access, test account/device, or traffic capture.

## Evidence reviewed

- [Bixberry home page](https://www.bixberry.com/)
- [How Bixberry works](https://www.bixberry.com/how-bixberry-works/)
- [Download page](https://www.bixberry.com/download/)
- [Privacy policy](https://www.bixberry.com/privacy/), reviewed 27 May 2026 per the document
- [Terms of Use](https://www.bixberry.com/terms-of-service/), updated 3 August 2026 per the document
- [Google Play listing](https://play.google.com/store/apps/details?id=ai.bixberry.app), package ID `ai.bixberry.app`
- [Apple App Store listing](https://apps.apple.com/gb/app/bixberry/id6767462542)
- [Microsoft Store listing](https://apps.microsoft.com/detail/xp88vkb45055h5?hl=en-GB&gl=GB)

The listings and policy are first-party/developer-provided descriptions and declarations. They are useful leads, not proof of app behavior. Store privacy disclosures are self-reported.

## Preliminary Android APK findings (static inspection only)

The current `origin/main` contains four install-time splits under `bixberry_apk/`. I fetched these Git blobs for analysis without switching from the experiment branch or copying the APKs into it. Each APK is a readable ZIP and passes its archive CRC check:

| File | Size | SHA-256 |
|---|---:|---|
| `base.apk` | 9,130,618 bytes | `fe19e7685702546f3a0fe6b5ef8bd4fcd282c24fa787b2eadc6be12e70022c04` |
| `split_config.arm64_v8a.apk` | 57,672 bytes | `f249db74fe1074f4f038599e3dbe1c74573ba86f67d7d8d213f4edbf01946b72` |
| `split_config.en.apk` | 41,170 bytes | `50506ba4f574bc6a816d427055a3be96f896b51d112a813d34225779cd435de0` |
| `split_config.xxhdpi.apk` | 5,036,004 bytes | `2a835a4e52809228e1adcba7661225f14c8416d14c5a838f15557c7ee497c5da` |

Manifest and DEX observations from `base.apk`:

- Package `ai.bixberry.app`, version `1.3.0` (version code `17`), minimum Android API 29, target API 37.
- The app has a named `AgentService` foreground service and a `DeviceBootReceiver` that handles `BOOT_COMPLETED`. The service creates an “Earning in action” notification and accepts `start`/`stop` actions. This establishes the app has an explicit background agent lifecycle; it does **not** by itself establish when the agent auto-starts or what traffic it carries.
- Manifest permissions include Internet/network-state access, foreground service with `specialUse`, notifications, boot completed, wake lock, and a request to ignore battery optimizations. **No Android location permission is declared** in this build. That does not mean the service never learns region (e.g. from IP/backend), only that Android OS location permission is not requested by this manifest.
- I also found no Android `VpnService`, Accessibility service, or broad media/storage permission in the manifest. Combined with the code's per-job sockets, the inspected build shows no standard mechanism for capturing all other apps' browsing traffic. It can still inspect the host/port/protocol and bytes of the work it relays itself.
- The background agent is not just UI: `AgentService` is a foreground service, `WsAgent` maintains the remote session, resolves configured hostnames with retries, starts inbound-message and traffic-handling coroutines, and has reconnect/cleanup paths. The service creates an “Earning in action” notification and supports explicit `start`/`stop` actions. A boot receiver handles `BOOT_COMPLETED`; its downstream condition for restarting activity still needs tracing.
- The agent's decoded WebSocket configuration has hostnames `zawspin.com`, `dawstrix.com`, and `powslet.com`. Its feature enum names are `IP_VERSION`, `BLACKLIST`, `RECONNECT`, `UDP`, and `TCP`. The actual DNS client also includes Cloudflare resolver hostnames (`cloudflare-dns.com`, `security.cloudflare-dns.com`, `family.cloudflare-dns.com`). Domain ownership, endpoint certificates, and which fallback is used at runtime remain unverified; these strings are noteworthy, not by themselves evidence of abuse.
- **Host-device relay path (strong static evidence):** a decoded traffic message type carries a hostname, port, protocol enum (`TCP`/`UDP`), byte data, and associated metadata. The traffic distributor resolves the requested hostname, applies configured target-address/range checks (it has an explicit `Connection to target forbidden` failure), and opens a TCP connection or UDP datagram channel from the device. It then handles byte messages against the WebSocket session. This supports the interpretation that an opted-in device acts as a controlled egress/relay node for partner jobs—not merely that AI code runs locally. We still need runtime capture to verify exact request/response framing, destinations, payload visibility, and remote-side routing.
- A separate REST client targets `api.bixberry.com`; code constructs `X-platform`, `X-uid`, `X-api-request`, and User-Agent headers, and some flows attach a Bearer token. Literal route strings include `/api/v1/referrals`, `/api/v1/users/tokens`, `/api/v1/users/tokens/google`, and `/api/v1/vids`. These are examples, not a complete API inventory; other routes may be assembled dynamically.
- Device/network metadata is represented in the code: `Build.MANUFACTURER`, `Build.MODEL`, Android release, app package/version, process details, current network transport (mobile/Wi-Fi/Ethernet/unknown), and network quality/state. The agent's pong/status builder reads Android `NetworkCapabilities` to classify the active transport. Android location permission is absent, so region can plausibly be derived server-side from IP; we have not observed the exact method.
- Reward/account types in the APK include user/token APIs, `GiftCards`, `Payouts`, `Referrals`, balance-credit updates, request-count updates, online-time standard/premium categories, quality, and a booster. These identify the client-side reward dimensions; they do not expose partner prices or the backend's margin.
- Bundled/initialized SDKs include Firebase Analytics, Crashlytics, Cloud Messaging and Remote Config, Microsoft Clarity, and Singular markers. Application startup initializes Microsoft Clarity and conditionally pauses it; Firebase Cloud Messaging is also initialized. Inclusion or initialization does not establish the exact events/screens/data transmitted, which needs configuration and traffic review.

This is meaningful evidence about app structure, but it remains **static**: the APKs have not been installed or executed, and we have not observed a live job, network payload, actual egress IP, consent transition, or reward calculation.

## Published product flow

1. **Install / account:** Website says sign up or log in, then opt in. The iOS listing describes Google or Apple sign-in; the privacy policy says registration data includes the Google/Apple account email, depending on sign-up method.
2. **Consent and configuration:** User approves connection sharing and chooses Wi-Fi-only or mobile-data use. Sharing is stated to be off until enabled and pausable from the app/settings.
3. **Background work:** While sharing is active, small web lookups from vetted AI companies are said to be sent through the participant's connection. The user-facing examples are pages, prices, listings, current/local results. Website copy calls this “AI traffic only.”
4. **Metering/rewards:** Published earning factors are request volume, time online/connected, region/connection quality, and referrals. The app is said to show live activity and rewards.
5. **Redemption:** Gift cards are the stated payout mechanism. The home FAQ says first payout at $10 and later payouts at $20; “instant” delivery is claimed. These are current marketing statements and may vary by country/account or change over time.

### Monetization: what is evidenced vs. unknown

- **Participant compensation:** The APK contains balance-credit, request-count, standard/premium online-time, referral, quality, and booster concepts, as well as gift-card and payout models. This aligns with the public claim that participants accrue rewards and redeem gift cards. The app binary does not reveal the actual rate card, how the premium/booster multiplier is calculated, country-specific thresholds, redemption fulfillment cost, or who funds each credit.
- **Company-side customer use (explicit in Terms):** Bixberry defines “Clients” as business clients that use Device Resources to accelerate AI requests or gather publicly available web data for business purposes, explicitly including pricing, brand/market analysis, and ad verification. The Terms also mention possible LLM training/search enhancement and say clients may request resources from a particular country or region. This is more specific—and broader—than the homepage's AI-currentness pitch. The commercial product is therefore a region-targeted residential egress/data-access network. The Terms imply client demand underwrites the business and user rewards, but exact client pricing, billing units, gross margin, or how much content Bixberry itself stores are not public in the APK or Terms.
- **Ads/analytics are not the same as core monetization:** The package includes advertising-ID/attribution-related permissions and analytics SDKs (including Singular), but that alone does not show ads are served or that ads fund rewards.

### Likely logical components (inference, not verified architecture)

- **Participant client / data plane (partly code-confirmed):** foreground Android service, long-lived WebSocket agent, destination DNS resolution, target checks, and device-originated TCP/UDP sockets for relay traffic.
- **Control plane (inference):** account identity, device/region/network status, partner eligibility/configuration, session state, abuse/fraud controls, and remote settings/feature updates.
- **Metering and rewards ledger (client models + vendor claim):** request/time/referral event categories aggregate into a balance; payout and gift-card catalog types are present.
- **Analytics/support:** crash and usage telemetry, customer-support tooling, account/payout records.

Static code supports an outbound TCP/UDP relay interpretation, but we have not confirmed the full wire-level job protocol, whether all production jobs are public-web retrieval, the complete endpoint set, which party can inspect request contents, backend vendors beyond bundled SDKs, or the billing/reward formula.

## Data and permission disclosures to pay attention to

The Privacy Policy says the app can collect:

- Signup email (Google/Apple account, based on the method used).
- Payout information: name, surname, email, amount, request/disbursement dates, transaction/earning history.
- Device information: IP address, OS version, model, last-seen time, city/country location.
- Whether Wi-Fi is on or off; aggregated analytics and crash/error reports.
- It names Google/Facebook analytics, Firebase for Android crash/performance reporting, Hetzner storage, Freshdesk support, and email delivery providers.
- Most importantly for the experiment, it says business partners/clients may receive **location and IP data** to access public internet resources through users' connections. The policy describes this disclosure as necessary for the service contract.

The iOS App Store privacy section separately lists precise location, email, user/device IDs, and product interaction as data that may be linked to the user, plus diagnostics not linked. The listing says location may be used while the app is not open. Its text says location is used to establish the region from which the connection is sharing.

The Google Play listing says data is encrypted in transit, users can request deletion, and declares no data shared with third parties while noting personal info, app/performance info, and device/other IDs may be collected. This should be reconciled with the Privacy Policy's partner disclosure; store form categories and legal-policy wording are not directly equivalent, but this is an important question to validate.

## Claims vs. unknowns / tensions worth testing

- **“Nothing runs on your device” vs resource language:** iOS copy says nothing runs on the device, while Google Play says limited network and compute resources and the website says device resources. We need to inspect CPU, memory, battery, and network use rather than infer from slogans.
- **“Only AI traffic” vs actual request scope:** The APK contains a relay-task protocol and target checks, but no partner roster, production policy, or real sample job. Verify which destinations and protocols are allowed, what URL/request content is carried, and whether user-specific requests or credentials could ever be included.
- **“Doesn't see browsing” vs proxy mechanics:** The inspected code shows the agent can open outbound TCP/UDP sockets for server-supplied host/port work; this is separate from reading the user's browser history, but means request/response data traverses the device. Determine exactly what the app, Bixberry backend, relay customer, and destination site can see/log.
- **Consent boundaries:** Verify no traffic is relayed before explicit opt-in, behavior after pause/stop/logout/revoke, and whether background execution/location permissions are required for each OS.
- **Region/IP disclosure:** The policy says partners/clients may receive participant IP/location. The Terms state client requests reach the web from the participant's IP and warn of ISP action, data charges, blocklists, CAPTCHAs, service access issues, and complaints. Determine which parties see which network metadata and what is logged/retained.
- **Economics:** Published thresholds and claimed request/time/referral reward factors do not disclose rates, demand distribution, payout availability by country, or expected earnings. Do not extrapolate “passive income” from the marketing copy.
- **Privacy policy is broad:** It states service data may be retained five years after relationship/last activity; security data one month; inquiry/marketing data two years; legal records ten years. This is worth reconciling with in-app controls and account-deletion behavior.

## What we have not done

- No proprietary app source was found.
- The Android split APKs from `origin/main` were extracted to `/tmp` and statically inspected (manifest, DEX metadata/strings, and selected decompiled methods). They were **not installed or executed**. No iOS/Windows binary was acquired.
- No account was created and no live traffic was generated, intercepted, or observed.
- Static evidence does not establish safety, policy compliance, server-side behavior, or actual payload handling.

The sandbox could fetch the APK blobs through GitHub, but direct shell access to Bixberry's download host failed. Do not run the APKs on a general-purpose host; any dynamic testing should use a dedicated test device/emulator with an authorized test account.

## Authorization / bounty status

As of 2026-10-31, targeted public searches did not find an official Bixberry bug-bounty or vulnerability-disclosure program, and `https://www.bixberry.com/.well-known/security.txt` returns a page-not-found response. This does not rule out a private/invitation-only program.

The [Terms of Use](https://www.bixberry.com/terms-of-service/) (updated 3 August 2026) prohibit reverse engineering or source extraction without prior written authorization, and prohibit using the app for competitive monitoring if developing a competing product. They also prohibit interference, accessing another user's account, and manipulating earnings. Before any live test—especially given the competitive experiment—obtain written scope and safe-harbor authorization from Bixberry. The Terms list `support@bixberry.com` for complaints; ask there whether a security program exists and request written permission, permitted targets/techniques, test accounts, and bounty eligibility. No bounty should be assumed.

## Recommended experiment (authorized test device/account only)

1. Verify the origin/version/signature of the APK splits already in `origin/main`; compare against the current Play Store build if possible. Prefer a spare Android device/emulator and test account; do not use personal browsing or credentials.
2. **Complete static triage before execution:** map the remaining obfuscated app code and host-string call sites, enumerate all manifest intent filters/SDKs/native libraries, and inspect network security configuration. Preserve hashes and tool versions.
3. **Consent matrix:** fresh install, before opt-in, opt-in on Wi-Fi, mobile-data enabled, paused, stopped, logged out, permission revoked, and app uninstalled. Record permission prompts, background service/notification state, network activity and UI state at each step.
4. **Traffic/resource observation:** use a controlled Wi-Fi network and instrumented test device. Capture DNS, destination IP/hostname, connection timing/volume, and CPU/battery/network counters. Avoid TLS interception or certificate-pinning bypass unless explicitly authorized and scoped; no probing of third-party endpoints.
5. **Request validation:** If a legitimate in-app test task or vendor-provided test partner is available, document task source, request type, payload visibility, response path, egress IP, and whether pause takes effect immediately. Do not send arbitrary requests through another person's connection.
6. **Ledger/payout verification:** Record displayed balance changes against request/time/referral events; check threshold, fees, country/brand availability, cancellation and account deletion. Never spend real money or redeem personal data without explicit approval.
7. Report only reproducible observations; maintain a clear split between vendor statements, static evidence, and dynamic evidence.

### Inputs needed to go beyond this baseline

- Official install artifact (APK/MSIX/EXE) or a test device with the app installed; and
- explicit authorization for a controlled account/device/network experiment, including which analysis techniques are in scope.
