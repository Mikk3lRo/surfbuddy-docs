# Feature Documentation

- **Position sharing** — two separate-looking mechanisms exist and their relationship isn't confirmed yet: a frontend-only `/:base64position` route (map position encoded directly in the URL) in `surfbuddy-vue`, and a backend `POST /share` + `GET /share/{token}` (+ OG image variants) in `surfbuddy-v3` used to build `https://surfbuddy.dk/s-{token}` links. Confirm in code before relying on how these interact.
