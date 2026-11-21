# F5 BIG-IP Lab Workspace Guidelines

## Documentation & README Formatting Rules

### Screenshot Layout Invariant (Vertical Stacking Only)
When documenting lab exercises, packet captures, and GUI verification in `README.md` files:
1. **Never use charts / tables for screenshots:** Do not embed images side-by-side inside Markdown table columns (e.g., avoid `| Header 1 | Header 2 |` with images in table cells).
2. **Always list screenshots vertically:** Every screenshot must appear sequentially on its own line in a vertical layout.
3. **Precede each screenshot with a bold label:** Place a bold descriptive label immediately above each image embed.

#### Example Reference:
Do NOT format like this:
```markdown
| Packet List: HTTP Request & Immediate 301 Response | Deep Packet Inspection of Frame 68 (`301 Moved Permanently`) |
|---|---|
| ![Wireshark Overview - HTTP and 301](screenshots/wireshark-http-tls.png) | ![Wireshark Frame 68 Details](screenshots/wireshark-301-details.png) |
```

DO format like this:
```markdown
**HTTP Request & Immediate 301 Response**

![Wireshark Overview - HTTP and 301](screenshots/wireshark-http-tls.png)

**Deep Packet Inspection of Frame 68 (`301 Moved Permanently`)**

![Wireshark Frame 68 Details](screenshots/wireshark-301-details.png)
```

4. **Clickable File Links:** Always create clickable links for referenced capture files, scripts, and logs using the `file://` scheme where appropriate.
