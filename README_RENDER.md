Render Deployment & Local Testing

This file describes quick steps to deploy the app to Render and to test the QR scanner locally.

1) Push to GitHub

- Commit your changes and push to the `main` branch (or a branch you want Render to use):

```bash
git add .
git commit -m "QR scanner: autofocus, torch, overlay, dev seed endpoint, render README"
git push origin main
```

2) Create a Web Service on Render

- Go to https://render.com and log in.
- Click `New` -> `Web Service`.
- Connect your GitHub account and select this repository (`aura-ticket-system-clean`).
- Branch: `main` (or the branch you pushed).
- Build command: leave empty or set if you have build steps. (This app runs using Node; there is no build step in this repo.)
- Start command: `node server.js`
- Environment:
  - `PORT` is handled by Render automatically; do not set it manually unless you have a reason.
  - Set `NODE_ENV=production` for production deployments.
- Click `Create Web Service`. Render will build and deploy and provide an HTTPS URL for your service.

Notes about HTTPS

- Modern browsers only allow camera access on secure contexts (HTTPS) or on `localhost`.
- Render provides HTTPS for your deployed service, so camera access should work on the deployed URL.

3) Local testing (before or instead of deploying)

- Start the server on your machine:

```bash
cd /path/to/aura-ticket-system-clean-main
node server.js
```

- Open in browser: `http://localhost:3000/` (use `localhost` rather than an IP for camera permissions in some browsers).

4) Seed a dev ticket for quick verification

- The server includes a dev-only endpoint at `/dev/seed-ticket` which is enabled when `NODE_ENV !== 'production'`.
- Use `curl` to seed a ticket:

```bash
curl -X POST http://localhost:3000/dev/seed-ticket -H "Content-Type: application/json" -d '{}'
```

- The response will contain the `token` you can use to generate a QR code (or the token may already be the data encoded by your QR generator).

5) Test scanning flow

- Point the device camera at a QR code containing the token returned by the seed endpoint, or modify the scanner code to accept the token directly for testing.
- The scanner will attempt to apply autofocus and show a flash button if the device supports it.
- If you encounter permission issues in your browser, try using `localhost` or deploy to Render to get HTTPS.

6) Troubleshooting

- If the torch button does not appear, your device or browser probably does not expose `capabilities.torch` to the page.
- Autofocus support varies by device and browser. Check the browser console for messages about autofocus attempts.
- Review server logs on Render or local console for `/api/verify-ticket` calls and dev-seed responses.

7) Optional improvements

- Replace `alert()` with in-page modals for better UX.
- Add persistence for seeded dev tickets (this endpoint only stores tickets in memory).
- Add tests for hardware capability detection across browsers.

If you want, I can add a small QR generator page in the repo that creates a QR image from the dev token so you can test scanning directly without printing.
