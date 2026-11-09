# Google Drive PDF Capture

A personal experimental browser-console utility that captures rendered PDF pages from the Google Drive PDF Viewer and combines them into a downloadable PDF file.

## Overview

Google Drive uses lazy loading and virtualized rendering to display PDF documents. As a result, not all PDF pages are available in the DOM at the same time.

This script is an experimental attempt to work around that behavior by:

1. Detecting the PDF viewer's scrollable container.
2. Detecting the total page count from the viewer when possible.
3. Scrolling through the document progressively.
4. Capturing rendered page images.
5. Tracking captured pages and checking the expected page count.
6. Combining captured images into a PDF using jsPDF.
7. Downloading the generated PDF.

## Project Status

<escape>**Personal experiment — not production-ready.**</escape>

This project was created for personal experimentation, learning, and research purposes.

It is provided as a proof of concept. The script may be incomplete, unreliable, or incompatible with future changes to Google Drive or browser implementations.

There is no guarantee that all pages will be captured correctly.

## Features

- Automatic page-count detection when the viewer exposes a recognizable page counter.
- Manual page-count fallback when automatic detection is unavailable.
- Automatic scrolling through the document.
- Progressive page capture.
- Duplicate-position detection.
- Capture progress logging.
- PDF generation using jsPDF.
- English-only script, comments, and console messages.
- Prevents exporting when the captured page count does not match the expected page count.

## Requirements

- Google Chrome or another Chromium-based browser.
- Access to the Google Drive PDF Viewer.
- Permission to view the source document.
- An internet connection to load jsPDF from the CDN.
- Browser settings that allow the script to load the external library.

## Usage

### Step 1. Open the PDF from Google Drive

1. Open Google Drive.
2. Locate the PDF document you want to capture.
3. Open the document.

### Step 2. Open the PDF Preview

1. If the document opens in the Google Drive file interface, click **Preview**.
2. Wait for the PDF preview to finish loading.

### Step 3. Open the PDF in a new window

1. Locate the **three vertical dots** menu in the top-right corner of the PDF preview.
2. Click the menu.
3. Select **Open in new window**.
4. Wait for the PDF Viewer to load in the new browser tab or window.

### Step 4. Load the document content

Before running the script:

1. Scroll down through the PDF to the end of the document.
2. Allow the viewer to load the document content as you scroll.
3. Wait until the final page has rendered.
4. Return to the top of the document.

This preparation step can help the viewer initialize and load its page content. It does not guarantee that all pages will remain in the DOM because Google Drive may virtualize page elements.

### Step 5. Open Developer Tools

Open the browser Developer Tools.

- **Windows/Linux:** `F12` or `Ctrl + Shift + J`
- **macOS:** `Option + Command + J`

Select the **Console** tab.

### Step 6. Enable pasting in Chrome Console

Chrome may prevent pasting code directly into the Developer Tools Console as a security precaution.

If you see a warning that prevents pasting:

1. Click inside the Console input area.
2. Type the following command manually:

   ```text
   allow pasting
   ```

3. Press `Enter`.
4. You should now be able to paste the script into the Console.

Only enable pasting when you understand and trust the code you are about to execute. Never paste code from an untrusted source.

### Step 7. Run the script

1. Copy the complete script from the section below.
2. Paste it into the Console.
3. Press `Enter` to execute.
4. Keep the PDF Viewer tab open and active while the script runs.

### Step 8. Monitor the capture progress

The Console displays the detected page count and capture progress.

Example:

```text
=== Google Drive PDF Capture ===
Detected total pages: 101
jsPDF loaded successfully.
PDF scroll container detected.
Starting scan 1/3...
Pass 1 | Captured 10/101 pages
Pass 1 | Captured 20/101 pages
...
Total captured: 101/101
Generating PDF...
Downloading PDF...
PDF downloaded successfully.
```

If the script cannot detect the total page count, it asks you to enter the expected number of pages.

### Step 9. Find the generated PDF

After successful completion, the generated PDF is saved to the browser's default download location.

## Configuration

The script contains a configuration object near the beginning:

| Option | Default | Description |
|---|---:|---|
| `expectedPages` | `null` | Expected page count. `null` enables automatic detection with manual fallback. |
| `waitMs` | `1200` | Delay between scrolling operations, in milliseconds. |
| `maxPasses` | `3` | Maximum number of document scanning passes. |
| `imageQuality` | `0.95` | JPEG image quality, from 0 to 1. |
| `outputName` | `GoogleDrive_Capture.pdf` | Output PDF filename. |

Increase `waitMs` if the document is large or the viewer loads pages slowly.

## Complete Script

```javascript
(async function () {
  const CONFIG = {
    expectedPages: null,
    waitMs: 1200,
    maxPasses: 3,
    imageQuality: 0.95,
    outputName: "GoogleDrive_Capture.pdf"
  };

  console.log("=== Google Drive PDF Capture ===");

  if (window.__pdfCaptureRunning) {
    console.warn("A PDF capture process is already running.");
    return;
  }

  window.__pdfCaptureRunning = true;

  try {
    // 1. Detect the total number of pages
    function detectTotalPages() {
      const candidates = [
        ...document.querySelectorAll(
          'input, [aria-label], [title], [role="textbox"], [role="spinbutton"]'
        )
      ];

      const patterns = [
        /^\s*\d+\s*\/\s*(\d+)\s*$/,
        /^\s*Page\s+\d+\s+of\s+(\d+)\s*$/i,
        /^\s*\d+\s+of\s+(\d+)\s*$/
      ];

      for (const element of candidates) {
        const values = [
          element.value,
          element.getAttribute("aria-label"),
          element.getAttribute("title"),
          element.textContent
        ].filter(Boolean);

        for (const value of values) {
          const normalized = String(value).trim();

          for (const pattern of patterns) {
            const match = normalized.match(pattern);

            if (match) {
              const total = Number(match[1]);

              if (Number.isInteger(total) && total > 0) {
                return total;
              }
            }
          }
        }
      }

      return null;
    }

    let expectedPages =
      CONFIG.expectedPages || detectTotalPages();

    if (!expectedPages) {
      const answer = prompt(
        "The total page count could not be detected automatically. " +
        "Please enter the expected number of pages:"
      );

      expectedPages = Number(answer);

      if (
        !Number.isInteger(expectedPages) ||
        expectedPages <= 0
      ) {
        throw new Error("A valid expected page count is required.");
      }
    }

    console.log("Detected total pages:", expectedPages);

    // 2. Load jsPDF with Trusted Types support
    async function loadJsPDF() {
      if (window.jspdf?.jsPDF) {
        return window.jspdf.jsPDF;
      }

      const url =
        "https://unpkg.com/jspdf@2.5.2/dist/jspdf.umd.min.js";

      await new Promise((resolve, reject) => {
        const script = document.createElement("script");

        script.onload = resolve;
        script.onerror = () =>
          reject(new Error("Failed to load jsPDF from the CDN."));

        let trustedURL = url;

        if (window.trustedTypes) {
          let policy;

          try {
            const policyName =
              "pdfCapturePolicy_" +
              Math.random().toString(36).slice(2);

            policy = window.trustedTypes.createPolicy(
              policyName,
              {
                createScriptURL: value => {
                  if (value !== url) {
                    throw new TypeError("Untrusted script URL.");
                  }

                  return value;
                }
              }
            );
          } catch (error) {
            reject(
              new Error(
                "Trusted Types policy creation failed: " +
                error.message
              )
            );
            return;
          }

          trustedURL = policy.createScriptURL(url);
        }

        script.src = trustedURL;
        document.head.appendChild(script);
      });

      if (!window.jspdf?.jsPDF) {
        throw new Error("jsPDF initialization failed.");
      }

      return window.jspdf.jsPDF;
    }

    const jsPDF = await loadJsPDF();

    console.log("jsPDF loaded successfully.");

    // 3. Locate the PDF viewer scroll container
    const getDriveImages = () => [
      ...document.querySelectorAll("img")
    ].filter(img =>
      img.src.startsWith("blob:https://drive.google.com/")
    );

    const scrollCandidates = [
      document.scrollingElement,
      ...document.querySelectorAll("*")
    ].filter(element =>
      element &&
      element.clientHeight > 200 &&
      element.scrollHeight > element.clientHeight + 300
    );

    scrollCandidates.sort((a, b) => {
      const score = element =>
        element.scrollHeight - element.clientHeight +
        Math.min(element.querySelectorAll("img").length, 10) * 1000;

      return score(b) - score(a);
    });

    const scroller = scrollCandidates[0];

    if (!scroller) {
      throw new Error("Unable to locate the PDF scroll container.");
    }

    console.log("PDF scroll container detected:", scroller);

    // 4. Capture rendered pages
    const pages = [];

    function getPagePosition(img) {
      const rect = img.getBoundingClientRect();

      if (scroller === document.scrollingElement) {
        return rect.top + window.scrollY;
      }

      const containerRect = scroller.getBoundingClientRect();

      return (
        rect.top -
        containerRect.top +
        scroller.scrollTop
      );
    }

    async function captureVisible() {
      let added = 0;

      for (const img of getDriveImages()) {
        if (
          !img.complete ||
          img.naturalWidth === 0 ||
          img.naturalHeight === 0
        ) {
          continue;
        }

        const top = getPagePosition(img);

        try {
          const canvas = document.createElement("canvas");

          canvas.width = img.naturalWidth;
          canvas.height = img.naturalHeight;

          const context = canvas.getContext("2d");

          context.drawImage(
            img,
            0,
            0,
            img.naturalWidth,
            img.naturalHeight
          );

          const data = canvas.toDataURL(
            "image/jpeg",
            CONFIG.imageQuality
          );

          const page = {
            top,
            width: img.naturalWidth,
            height: img.naturalHeight,
            data
          };

          const duplicateIndex = pages.findIndex(existing =>
            Math.abs(existing.top - top) < 30
          );

          if (duplicateIndex >= 0) {
            const existingPage = pages[duplicateIndex];

            if (
              page.width * page.height >
              existingPage.width * existingPage.height
            ) {
              pages[duplicateIndex] = page;
            }
          } else {
            pages.push(page);
            added++;
          }
        } catch (error) {
          console.warn("Failed to capture an image:", error);
        }
      }

      pages.sort((a, b) => a.top - b.top);

      return added;
    }

    // 5. Scan the entire document
    for (
      let pass = 1;
      pass <= CONFIG.maxPasses;
      pass++
    ) {
      console.log(
        `Starting scan ${pass}/${CONFIG.maxPasses}...`
      );

      scroller.scrollTop = 0;

      await new Promise(resolve =>
        setTimeout(resolve, 2000)
      );

      let idleAtBottom = 0;
      let previousHeight = -1;

      for (let step = 0; step < 2000; step++) {
        await captureVisible();

        console.log(
          `Pass ${pass} | Captured ${pages.length}/${expectedPages} pages`
        );

        const maxScroll =
          scroller.scrollHeight - scroller.clientHeight;

        const current = scroller.scrollTop;

        if (current >= maxScroll - 5) {
          await new Promise(resolve =>
            setTimeout(resolve, CONFIG.waitMs)
          );

          const added = await captureVisible();

          if (
            added === 0 &&
            scroller.scrollHeight === previousHeight
          ) {
            idleAtBottom++;
          } else {
            idleAtBottom = 0;
          }

          previousHeight = scroller.scrollHeight;

          if (idleAtBottom >= 3) {
            console.log("Reached the end of the document.");
            break;
          }
        } else {
          idleAtBottom = 0;

          const stepSize = Math.max(
            250,
            scroller.clientHeight * 0.5
          );

          scroller.scrollTop = Math.min(
            current + stepSize,
            maxScroll
          );

          await new Promise(resolve =>
            setTimeout(resolve, CONFIG.waitMs)
          );
        }

        if (pages.length >= expectedPages) {
          break;
        }
      }

      if (pages.length >= expectedPages) {
        break;
      }
    }

    // 6. Verify the captured page count
    pages.sort((a, b) => a.top - b.top);

    console.log("==============================");
    console.log(
      `Total captured: ${pages.length}/${expectedPages}`
    );
    console.log("==============================");

    if (pages.length !== expectedPages) {
      console.error(
        "PDF capture incomplete. The expected page count was not reached."
      );

      alert(
        "PDF capture incomplete!\n" +
        `Captured: ${pages.length}/${expectedPages} pages.\n` +
        "The PDF was not exported."
      );

      return;
    }

    // 7. Generate the PDF
    console.log("Generating PDF...");

    let pdf = null;

    for (let index = 0; index < pages.length; index++) {
      const page = pages[index];

      const orientation =
        page.width > page.height ? "l" : "p";

      if (index === 0) {
        pdf = new jsPDF({
          orientation,
          unit: "px",
          format: [page.width, page.height],
          compress: true
        });
      } else {
        pdf.addPage(
          [page.width, page.height],
          orientation
        );
      }

      pdf.addImage(
        page.data,
        "JPEG",
        0,
        0,
        page.width,
        page.height,
        undefined,
        "FAST"
      );

      console.log(
        `PDF generation: ${index + 1}/${pages.length}`
      );
    }

    // 8. Download the generated PDF
    console.log("Downloading PDF...");

    await pdf.save(CONFIG.outputName, {
      returnPromise: true
    });

    console.log(
      "PDF downloaded successfully:",
      CONFIG.outputName
    );

    alert(
      "PDF generation completed successfully!\n" +
      `Total pages: ${pages.length}\n` +
      `File: ${CONFIG.outputName}`
    );

  } catch (error) {
    console.error("PDF capture failed:", error);
    alert("PDF capture failed: " + error.message);

  } finally {
    window.__pdfCaptureRunning = false;
  }
})();
```

## Troubleshooting

### Chrome Console does not allow pasting

Chrome may display a self-XSS warning and block pasted code.

Type `allow pasting` manually into the Console and press `Enter`. Then paste the script.

Only do this when you have reviewed and trust the script.

### The script cannot detect the total page count

The Google Drive Viewer may use a different page-counter layout or may not expose the counter in the DOM.

The script will ask you to enter the expected page count manually.

### Trusted Types policy creation failed

Some websites restrict dynamically created Trusted Types policies.

The script uses a Trusted Types policy to load jsPDF from the specified CDN. If the website blocks policy creation, the library cannot be loaded using this approach.

Do not disable browser security protections to work around this issue.

### Only a few pages are captured

Google Drive may virtualize page elements and reuse image elements while scrolling.

Try:

- Increasing `waitMs` to `2000` or `3000`.
- Keeping the document tab active during capture.
- Scrolling through the document before running the script.
- Running the script again after the viewer has fully loaded.
- Checking whether the selected scroll container is actually the PDF viewer.

### Captured page count does not match the expected count

The script may fail to capture some pages because of lazy loading, delayed rendering, or image replacement.

A matching count does not guarantee that every page is unique or in the correct order. Review the resulting PDF before relying on it.

### PDF generation fails

Large documents may require significant browser memory because the script keeps captured images in memory before generating the PDF.

Try lowering `imageQuality`, closing unused tabs, or processing a smaller document.

## Limitations

- This utility captures rendered images rather than extracting the original PDF data.
- Output quality depends on the resolution of the images rendered by Google Drive.
- Automatic page-count detection is best-effort and depends on the viewer's DOM structure.
- A page count match is not proof that every page was captured correctly.
- Duplicate detection based on image position is not a guaranteed unique-page identifier.
- Very large documents may exceed available browser memory.
- Google Drive may change its viewer implementation at any time, which can break the script.

## Privacy and Security

- The script runs in the browser tab where the document is open.
- Captured page images are processed locally in the browser.
- jsPDF is loaded from the unpkg CDN.
- The script does not intentionally upload document contents to an external server.
- Only run scripts you have reviewed and understand.
- Ensure you have permission to capture and reproduce the source document.

## Disclaimer

This project is provided strictly for personal experimentation, educational, and research purposes.

The script is provided "AS IS", without warranties or guarantees of any kind, either express or implied.

By using this project, you acknowledge and agree that:

- You use the script entirely at your own risk.
- The author makes no guarantees regarding accuracy, reliability, completeness, availability, or fitness for any particular purpose.
- The author is not responsible for any data loss, corrupted files, browser issues, service interruptions, or other damages arising from the use or misuse of this script.
- The author does not provide technical support, maintenance, or guarantees of future compatibility.
- You are solely responsible for ensuring that you have the legal right and appropriate authorization to access, capture, copy, and reproduce any documents processed using this script.
- You must comply with applicable laws, copyright requirements, document permissions, and Google Drive's terms of service.
- This project is not affiliated with, endorsed by, or supported by Google.

To the maximum extent permitted by applicable law, the author disclaims all liability arising from the use of this project.

## License and Usage Restrictions

This is a personal experimental project.

No open-source license is granted. All rights are reserved by the author.

No permission is granted to use, copy, modify, redistribute, sublicense, or commercially exploit this code without the author's prior written permission.

The repository is published for reference and personal experimentation only.
