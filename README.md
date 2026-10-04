# Chor Sipahi: real-world Hide & Seek for Android

Friends join a room. Everyone shakes their phone to flip a full-screen card: one player becomes
**👮 Sipahi (police)** and everyone else is a **🦹 Chor (thief)**. Thieves hide within 100 m of the
police phone. The police hunts them down for 20 minutes, guided by vibration when a thief is close.
Entry fees go into a pot: the police wins it all by catching every thief, otherwise the surviving
thieves split it equally.

```
Hide&Seek/
├── server/   Node.js game server: Express REST API + Socket.IO real-time, SQLite (built into Node)
└── mobile/   Android app: Expo SDK 57 / React Native, Expo Router
```

## Run it (development)

Requirements: Node 22.13+ (tested on Node 24). An Android phone with **Expo Go** from the Play Store.
The phone and the PC must be on the same Wi-Fi.

```bash
# 1. Server
cd server
npm install
npm start                 # prints the LAN address, e.g. http://192.168.1.22:4000
npm test                  # 20 tests: rules, payouts, zone and proximity logic, refunds

# 2. App
cd ../mobile
npm install
npm start                 # scan the QR code with Expo Go
```

If the server's address differs from `mobile/.env`, either edit `.env` or tap **⚙ Server** on the login
screen. In dev mode the OTP appears on screen and wallet top-ups are credited instantly.

**Short test games:** start the server with shorter timers, e.g.
`GAME_DURATION_SEC=180 HIDE_DURATION_SEC=20 RADAR_PULSE_EVERY_SEC=30 npm start`
(PowerShell: `$env:GAME_DURATION_SEC=180; npm start`). Every rule in `server/src/config.js` can be
overridden this way.

> **Windows note:** the `&` in the folder name `Hide&Seek` breaks npm's `.cmd` shims, so commands like
> `npx expo ...` fail. The npm scripts call the tools through `node` directly to work around this.
> Renaming the folder to `HideSeek` also fixes it.

### Build an installable APK

```bash
cd mobile
npx eas-cli@latest login
npx eas-cli@latest build -p android --profile preview    # cloud build, gives you an .apk link
```

Set the server address for the build in `eas.json` → `build.preview.env.EXPO_PUBLIC_SERVER_URL`.
No Android Studio or SDK is needed.

## Game rules as implemented

| Your requirement | Implementation |
|---|---|
| Rooms, invites, friends | 6-letter room codes, share via WhatsApp/SMS, a friends list (add by phone number), live in-app invites to online friends |
| Full-screen cards, shake to flip | The server deals; your card is only revealed to you when you shake (with a tap fallback; auto-flip after 45 s) |
| 1 police, rest thieves | Picked by a provably-fair draw (see below) |
| 100 m range, 30 s alert | Out of zone → red full-screen warning with a 30 s countdown and a vibration; still out after 30 s → disqualified |
| Police within 1 m → police phone vibrates, thief sees a green dot | Server compares GPS fixes every second; repeating vibration pattern on the police phone; large pulsing green dot for the thief |
| Catch by touch / 0 m / manual button | The police **holds the CAUGHT button on the thief's phone** for 1.5 s, or **scans the thief's QR catch code** |
| Police catches all → wins the whole pot | ✓ |
| Otherwise surviving thieves split it equally | ✓ split in paise; any leftover paise go one each to winners so the full pot is always paid out |
| 20 min, max 10 players | ✓ (min 2) |
| ₹10–₹100 entry, everyone pays equally | Host picks ₹10/20/30/50/75/100. Fees are taken **all at once when the host starts**, or from nobody if anyone is short |
| 1 free game per day, used up for everyone who played it | Free game = no fee, no cash prize, XP only. Tracked per player per day (IST) |
| Max ₹500 deposit per day | Enforced in the wallet ledger |

### Additions for fairness and fun

- **Provably fair police draw.** When the room is created, the server commits to `sha256(secret seed)` and
  shows it as the "fairness seal". Each player adds their own random seed when they ready up. The police
  is derived from all of these. The secret is revealed after the game, so anyone can verify the draw and
  the server can't rig it.
- **No police twice in a row.** In a rematch, the previous police can't be drawn again.
- **Hiding phase.** Thieves get a 90 s head start while the police screen is blacked out ("close your eyes
  and count").
- **Radar pulse.** Every 3 minutes the police sees how many thieves are within 25 / 50 / 100 m, with no
  directions.
- **Thief heat meter.** Thieves see cold / warm / hot / burning bands. The host can switch it off.
- **Final hunt.** In the last 2 minutes the police vibration range widens from 1 m to 10 m.
- **Three-strike zone rule.** Repeatedly stepping out of the zone to shake off the police disqualifies
  you, even if you come back each time.
- **Stealth screen for night games.** A black screen with dim red text, so the glow doesn't give away a
  hiding spot. Thief phones never vibrate when the police is near.
- **Rotating QR catch codes.** The code changes every 30 s, so screenshots can't be reused.
- **Anti-cheat and fail-safes.** Fake-GPS (mock location) is detected and disqualifies the player. A thief
  who loses GPS or connection for 90 s is out. If the police is offline for 3 minutes, the game is
  cancelled and **everyone is refunded**. A server crash or restart also refunds every unfinished game.
  Leaving mid-game counts as forfeiting.
- **Zone mode.** "Follows police" (your spec) or "fixed arena" (centred where the police stood when the
  hunt began).
- **XP, levels, leaderboard, history.** XP for playing, catching and winning; a friends leaderboard; game
  history.

## About GPS and "1 metre"

Phone GPS is typically accurate to **3–10 m** (worse indoors), so it cannot reliably measure 1 m or 0 m.
The app is honest about this:

- "Within 1 m" is evaluated as "**could** be within 1 m given both phones' reported accuracy" (uncertainty
  capped at 5 m). In practice, the vibration and green dot fire within roughly 1–6 m.
- Zone checks give the thief the benefit of the doubt, so nobody is disqualified for GPS jitter.
- A catch is **never** automatic from GPS, because false catches would cost real money. It always needs
  a physical action: holding the button on the thief's phone, or scanning their QR code.

True 1 m accuracy needs Bluetooth (BLE RSSI) or UWB. That is a good v2: it requires a development
build with `react-native-ble-plx`, which Expo Go doesn't include.

## ⚠ Before going live with real money

1. **Legal.** India's *Promotion and Regulation of Online Gaming Act, 2025* prohibits offering
   "online money games", meaning games where users pay or deposit money expecting winnings, whether
   based on skill or chance. It also bars banks and payment gateways from processing payments for them.
   This app's paid mode very likely falls under that. **Get advice from a lawyer before taking real money.**
   The free mode (XP only) and every other feature work without payments.
2. **Payments.** `POST /api/wallet/deposit` credits instantly **only in dev mode**. In production
   (`NODE_ENV=production`) it returns 501 until you integrate a gateway (Razorpay / Cashfree / PhonePe).
   Credit the wallet only from the gateway's verified webhook. Withdrawals are recorded as pending and
   need a payout integration.
3. **OTP.** Connect an SMS provider (MSG91, Twilio, Firebase Auth) in `server/src/auth.js`.
4. **HTTPS.** Deploy the server behind HTTPS/WSS, then remove `usesCleartextTraffic` from `mobile/app.json`.
5. **KYC / age checks** as required for payouts. The app currently uses an 18+ self-declaration.
6. **Scaling.** Rooms live in one server process's memory. For multiple instances, add the Socket.IO
   Redis adapter with sticky sessions.

## Architecture

- `server/src/rules.js`: pure rule functions (distance, zone, proximity, pot split, winner, fair police draw).
- `server/src/game/Room.js`: the room state machine `LOBBY → DEALING → HIDING → HUNTING → ENDED/VOID`,
  ticked once per second.
- `server/src/wallet.js`: integer-paise, append-only ledger with an all-or-nothing entry-fee collection.
- `server/src/socket.js`: real-time events (`room:*`, `card:flip`, `loc`, `catch:tap`, `catch:scan`,
  `invite:send`). The server pushes a **personalised** snapshot to each player. Locations are never sent
  to other players.
- `mobile/src/app/*`: screens (Expo Router). `mobile/src/components/room/*`: one view per game phase.
- `mobile/src/hooks`: shake detection (accelerometer), GPS streaming, server-synced clock.
