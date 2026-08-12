# PreludeTM

Deals a Terraforming Mars **Prelude** draft: pick a player count, tap Roll, and
each player gets 4 cards dealt from the 35-card Prelude deck with no repeats
across players — mirroring the real setup rule.

Tap any card to zoom in.

## Running it

No build step. Either:

- Open `index.html` directly in a browser, or
- Serve the folder with any static file server (e.g. `npx serve .`) and open
  it on your phone.

For reliable access from your phone, host it as a static site — e.g. GitHub
Pages or Cloudflare Pages pointed at this folder, same as the `substract`
project's frontend. Once hosted, add it to your phone's home screen for a
one-tap launch.

## Data

`index.html` embeds the 35 base Prelude cards (P01–P35) inline; images live
in `assets/cards/`. Card metadata and images sourced from
[hadronikle/Complete-Terraforming-Mars-Card-Database](https://github.com/hadronikle/Complete-Terraforming-Mars-Card-Database).
