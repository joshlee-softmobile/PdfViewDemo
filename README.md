# Browser PDF Popup & Window Behavior Guide

This guide documents browser security policies (Chrome, Safari, iOS Safari, Android Chrome, Edge, Firefox) regarding long-running asynchronous PDF generation and tab opening triggered by user click events.

---

## 1. Why `window.open()` after `await` gets blocked

Modern browsers enforce strict **User Activation / Transient User Gesture** policies:

1. **User Gesture Expiration:**
   When a user clicks a button, a temporary user gesture token is granted. If the event listener performs an asynchronous network fetch (`await fetch(...)`) that takes more than 1–2 seconds, the gesture context expires.
2. **Delayed `window.open()` Blocked:**
   If frontend code attempts `window.open(url)` **after** an expired `await` call, modern browsers catch and block the popup window entirely.

---

## 2. Production Solutions & Comparison

To support on-the-fly PDF generation across all browsers (including iOS Safari & Android Chrome), use the **GET Ticket Endpoint Pattern** (`/api/BoardingPass/PDFStream?ticket=GUID`).

| Option | Approach | Ticket Generation | Popup Blocker Risk | Compatibility | Recommended Scenario |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **Option A** | Pre-open `about:blank` + Fast Ticket POST + GET Redirect | On Demand (when clicked) | 🟢 Low | 🟢 100% (iOS, Android, Desktop) | **Dynamic PDF generation on button click** |
| **Option B** | Native `<a href="/PDFStream?ticket=GUID" target="_blank">` link | Pre-attached on page load | 🟢 0% Risk | 🟢 100% (iOS, Android, Desktop) | **Gold Standard (Zero Frontend JS)** |

---

## 3. Implementation Details

### Option A (Pre-Open Tab + Dynamic Ticket Fetch)
Best pattern when tickets should only be created when a passenger clicks "View PDF":

```javascript
onViewPdfClick: async function(customerPayload) {
  // 1. Synchronously open blank window inside click handler
  const pdfWindow = window.open('about:blank', '_blank');
  if (!pdfWindow) return alert('Popup blocked!');

  // 2. Inject initial loader UI into the new window
  pdfWindow.document.body.style.cssText = 'display:flex;justify-content:center;align-items:center;height:100vh;margin:0;background:#0f172a;color:#fff;';
  pdfWindow.document.body.innerHTML = `
    <div style="text-align:center;">
      <div class="spinner"></div>
      <h2>Preparing Boarding Pass PDF...</h2>
    </div>
  `;

  // 3. Fetch ticket GUID from backend (fast ~20ms POST call)
  const res = await api.post('/api/BoardingPass/PDFTicket', customerPayload);

  // 4. Navigate window to real server GET URL
  pdfWindow.location.href = `/api/BoardingPass/PDFStream?ticket=${res.ticket}`;
}
```

### Option B (Native Link — Gold Standard)
If ticket GUIDs are pre-attached during passenger list load:

```html
<a href="/api/BoardingPass/PDFStream?ticket=550e8400-e29b-41d4-a716-446655440000" target="_blank" class="btn">
  View Boarding Pass PDF
</a>
```
* Backend HTTP Header: `Content-Disposition: inline; filename="BoardingPass.pdf"`.
* Native browser PDF viewer opens seamlessly in a new tab with 0% popup blocker risk across all mobile & desktop platforms.

---

## 4. GitHub Actions & GitHub Pages Deployment

This repository includes automated deployment to GitHub Pages via GitHub Actions:

* **Workflow File:** [.github/workflows/deploy.yml](.github/workflows/deploy.yml)
* **Deployment Setup:**
  1. In GitHub Repository Settings, go to **Pages**.
  2. Under **Build and deployment** $\rightarrow$ **Source**, select **GitHub Actions**.
  3. Push changes to `main` or `master` to trigger automatic deployment.
