<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Freedom VPN</title>
<style>
*{box-sizing:border-box}
body{margin:0;background:#07111f;color:#f1f7fc;font-family:Arial,sans-serif;padding:24px}
main{max-width:620px;margin:30px auto;background:#102237;border:1px solid #244157;border-radius:22px;padding:26px}
header{display:flex;align-items:center;gap:14px}
.logo{background:#163d4b;padding:14px;border-radius:14px;font-size:28px}
h1{font-size:25px;margin:0}
.muted{color:#9bb0c3;font-size:14px}
.panel{background:#0a1929;border:1px solid #294358;border-radius:16px;padding:22px;margin-top:24px}
.center{text-align:center}
.shield{font-size:64px;color:#5eead4;margin:12px}
label{display:block;margin:18px 0 8px;color:#b8cbd9;font-size:14px}
input,select,button{width:100%;padding:14px;border-radius:10px;font-size:15px}
input,select{background:#0a1929;border:1px solid #355168;color:white}
button{margin-top:14px;background:#5eead4;color:#08202c;border:0;font-weight:bold;cursor:pointer}
button.secondary{background:#20384d;color:white}
button:disabled{opacity:.6}
#status{font-size:14px;line-height:1.6;color:#facc75}
#viewer{width:100%;margin-top:16px;border-radius:10px}
footer{color:#8096aa;font-size:12px;line-height:1.6;margin-top:20px}
</style>
</head>
<body>
<main>
<header>
<div class="logo">🛡️</div>
<div>
<h1>Freedom VPN</h1>
<div class="muted">Your browser. Your choice.</div>
</div>
</header>

<section class="panel center">
<div class="shield">♢</div>
<h2 id="heading">Browse with freedom</h2>
<p class="muted">Select your virtual location to begin.</p>
<label for="country" style="text-align:left">Virtual location</label>
<select id="country">
<option value="gb">🇬🇧 United Kingdom</option>
<option value="us">🇺🇸 United States</option>
<option value="ca">🇨🇦 Canada</option>
<option value="de">🇩🇪 Germany</option>
<option value="jp">🇯🇵 Japan</option>
</select>
<button id="connect">Connect</button>
<p id="status" role="status" aria-live="polite">
Not connected. Backend setup is required.
</p>
</section>

<section class="panel">
<h2>Website viewer</h2>
<p class="muted">Open a website through your remote browser session.</p>
<label for="destination">Website address</label>
<input id="destination" type="url"
value="https://ppt32mn322jkool.com"
placeholder="https://example.com">
<button id="open">Open through Freedom VPN</button>
<button id="disconnect" class="secondary" disabled>
End session
</button>
<div id="viewerArea" hidden>
<p class="muted">Remote browser preview</p>
<img id="viewer" alt="Remote browser screenshot">
</div>
</section>

<footer>
Website preview only. A deployed and secured backend is required
to create remote browser sessions. This page does not itself create
a VPN tunnel, change your IP address, or bypass website restrictions.
Keep provider credentials on the server, never in this file.
</footer>
</main>

<script>
const API = "/api";
let sessionId = null;
let refreshTimer = null;
let imageUrl = null;

const connect = document.querySelector("#connect");
const open = document.querySelector("#open");
const disconnect = document.querySelector("#disconnect");
const status = document.querySelector("#status");
const country = document.querySelector("#country");
const destination = document.querySelector("#destination");
const viewerArea = document.querySelector("#viewerArea");
const viewer = document.querySelector("#viewer");

country.addEventListener("change", () => {
  document.querySelector("#heading").textContent =
    "Location: " + country.options[country.selectedIndex].text;
});

async function request(path, options = {}) {
  return fetch(API + path, {
    ...options,
    credentials: "same-origin",
    headers: {
      "Content-Type": "application/json",
      ...options.headers
    }
  });
}

connect.addEventListener("click", () => {
  status.textContent =
    "Not connected: deploy and configure the backend first.";
});

open.addEventListener("click", async () => {
  let url;
  try {
    url = new URL(destination.value);
    if (url.protocol !== "https:") throw Error();
  } catch {
    status.textContent = "Enter a valid HTTPS website address.";
    return;
  }

  open.disabled = true;
  status.textContent = "Requesting a remote browser session…";

  try {
    const response = await request("/sessions", {
      method: "POST",
      body: JSON.stringify({
        url: url.href,
        country: country.value
      })
    });
    const data = await response.json();

    if (!response.ok) throw Error(data.error || "Session unavailable");
    sessionId = data.id;
    disconnect.disabled = false;
    viewerArea.hidden = false;
    status.textContent =
      "Session created. Loading remote browser…";

    async function refresh() {
      if (!sessionId) return;
      const result = await request(
        "/sessions/" + encodeURIComponent(sessionId) + "/screenshot"
      );
      if (!result.ok) throw Error("Viewer disconnected");

      const blob = await result.blob();
      const next = URL.createObjectURL(blob);
      viewer.src = next;
      if (imageUrl) URL.revokeObjectURL(imageUrl);
      imageUrl = next;
    }

    await refresh();
    status.textContent =
      "Remote session active. Screenshot preview only.";
    refreshTimer = setInterval(() => {
      refresh().catch(() => {
        clearInterval(refreshTimer);
        status.textContent = "Viewer unavailable. Reconnect to try again.";
      });
    }, 2000);
  } catch {
    status.textContent =
      "Could not connect. The backend must be deployed and authenticated.";
  } finally {
    open.disabled = false;
  }
});

disconnect.addEventListener("click", async () => {
  clearInterval(refreshTimer);
  const id = sessionId;
  sessionId = null;

  if (id) {
    try {
      await request("/sessions/" + encodeURIComponent(id), {
        method: "DELETE"
      });
    } catch {}
  }

  if (imageUrl) URL.revokeObjectURL(imageUrl);
  imageUrl = null;
  viewer.removeAttribute("src");
  viewerArea.hidden = true;
  disconnect.disabled = true;
  status.textContent = "Session ended.";
});
</script>
</body>
</html>
