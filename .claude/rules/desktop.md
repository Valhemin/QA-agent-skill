# Desktop

Desktop window testing is handled separately from web browser testing.

- Test case planning uses the same case table. Web records URL per screen. Desktop records how to open the app — no URL.
- Writing and running desktop tests does not use the web Playwright runner, web Page Objects, or `reports/ai/summary.json`.
- **Electron** — use Playwright `_electron.launch` + `firstWindow`. Do not use Spectron. Do not inherit `ELECTRON_RUN_AS_NODE`. Stub `dialog` via `evaluate` rather than driving the OS file picker. The splash screen is not the main window.
- **Tauri v2 on macOS** — use WebdriverIO with `@wdio/tauri-service`, provider `embedded`. `tauri-driver` via cargo is Windows/Linux only and is not available on macOS.
- **Native macOS** — use Appium 3 + mac2 (`platformName: mac`, `automationName: Mac2`, `bundleId`, requires XCTest and Xcode + Accessibility permission).
- **Native Windows** — use Appium Windows driver (UI Automation). Do not use the archived WinAppDriver.
- Locator priority for webview content: role / label / test id. For native controls: accessibility id then visible name, XPath last, no coordinates if the tree is readable.
- Each test opens and closes the app independently.
- Missing driver (e.g. mac2 on a machine without Xcode): write the spec, mark run as `unavailable`, do not install system-wide.
- Do not compare Figma for desktop windows — `qa-figma` is web only.
- Results (PASS or FAIL) go in `docs/test/product-test.md` in the product repo. `report-bug` and `release-readiness` read that file when there is no web `summary.json`.
