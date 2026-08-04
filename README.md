# Browser PDF Popup & Window Behavior Guide

This guide documents browser security policies (Chrome, Safari, iOS Safari, Edge, Firefox) regarding long-running asynchronous PDF generation and tab opening triggered by user click events.

---

## 1. Why `window.open()` after `await` gets blocked

Modern browsers enforce strict **User Activation / Transient User Gesture** policies:

1. **User Gesture Expiration:**
   When a user clicks a button, a temporary user gesture token is granted. If the event listener performs an asynchronous network fetch (`await fetch(...)`) that takes more than 1–2 seconds, the gesture context expires.
2. **Delayed `window.open()` Blocked:**
   If frontend code attempts `window.open(url)` **after** an expired `await` call, modern browsers catch and block the popup window entirely.

---

## 2. Summary of Options & Solutions

| Option | Approach | Popup Blocker Risk | Inline PDF Viewer? | Recommended Scenario |
| :--- | :--- | :---: | :---: | :--- |
| **Option A** | Pre-open `about:blank` + `location.href` redirect | ⚠️ High (Safari) | Yes | Desktop Chrome only |
| **Option B** | Pre-open `about:blank` + Loader Spinner + `<iframe>` Viewer | 🟢 Low | Yes | **POST APIs / Payload-based PDF requests** |
| **Option C** | Native `<a href="URL" target="_blank">` link | 🟢 0% Risk | Yes | **Gold Standard (Direct GET URLs)** |

---

## 3. Implementation Details

### Option B (Pre-open tab + Animated Loader + `<iframe>` Viewer)
Best pattern when PDF generation requires a POST request with payload:

```javascript
onViewPdfClick: async function() {
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

  // 3. Perform long backend API call
  const pdfUrl = await fetchBoardingPassPdfApi();

  // 4. Inject full-page iframe to render PDF inline
  pdfWindow.document.body.style.cssText = 'margin:0;padding:0;overflow:hidden;';
  pdfWindow.document.body.innerHTML = `<iframe src="${pdfUrl}" style="width:100vw;height:100vh;border:none;"></iframe>`;
}
```

### Option C (Native GET Link — Gold Standard)
If your API allows fetching via tokenized URL parameters:

```html
<a href="/api/BoardingPass/PDFFile?token=XYZ" target="_blank" class="btn">
  View Boarding Pass PDF
</a>
```
* Backend HTTP Header: `Content-Disposition: inline; filename="BoardingPass.pdf"`.
* Native browser PDF viewer opens seamlessly in a new tab with 0% popup blocker risk.

---

## 4. GitHub Actions & GitHub Pages Deployment

This repository includes automated deployment to GitHub Pages via GitHub Actions:

* **Workflow File:** [.github/workflows/deploy.yml](.github/workflows/deploy.yml)
* **Deployment Setup:**
  1. In GitHub Repository Settings, go to **Pages**.
  2. Under **Build and deployment** $\rightarrow$ **Source**, select **GitHub Actions**.
  3. Push changes to `main` or `master` to trigger automatic deployment.
