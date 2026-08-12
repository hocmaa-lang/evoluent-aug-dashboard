# Reconnecting team sync — 2 minutes

jsonblob.com started refusing writes (HTTP 403 from Cloudflare) on 12 Aug 2026.
Reads still work, so no data was lost, but nobody can save to the shared board.

Every free no-signup JSON store tested as a replacement was also dead or blocked
(kvdb.io, extendsclass, keyvalue.xyz, getpantry). The durable fix is a backend
that belongs to you and cannot be rate-limited away by someone else's abuse.

Google Apps Script is the cheapest one that fits: free, tied to the Google account
you already use, no server, and it will not disappear.

## Steps

1. Go to **script.google.com** → **New project**
2. Delete whatever is in the editor and paste the code below
3. **Deploy** → **New deployment** → gear icon → **Web app**
   - *Execute as*: **Me**
   - *Who has access*: **Anyone**
4. Click **Deploy**, authorise when asked, then copy the **Web app URL**
   (it looks like `https://script.google.com/macros/s/AKfy..../exec`)
5. Send me that URL and I will wire it in

## The code to paste

```javascript
const P = PropertiesService.getScriptProperties();

function doGet() {
  return ContentService
    .createTextOutput(P.getProperty('board') || '{"state":{},"times":{}}')
    .setMimeType(ContentService.MimeType.JSON);
}

function doPost(e) {
  const lock = LockService.getScriptLock();
  lock.waitLock(10000);
  try {
    const incoming = JSON.parse(e.postData.contents);
    const cur = JSON.parse(P.getProperty('board') || '{"state":{},"times":{}}');
    // newest-timestamp-per-key wins, so two people editing different lines both survive
    const keys = Object.keys(incoming.times || {}).concat(Object.keys(cur.times || {}));
    keys.forEach(function (k) {
      const it = (incoming.times || {})[k] || 0;
      const ct = (cur.times || {})[k] || 0;
      if (it > ct) {
        cur.times[k] = it;
        if (incoming.state[k] === undefined) delete cur.state[k];
        else cur.state[k] = incoming.state[k];
      }
    });
    cur.updated = incoming.updated;
    cur.by = incoming.by;
    P.setProperty('board', JSON.stringify(cur));
    return ContentService.createTextOutput(JSON.stringify(cur))
      .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

## Why this shape

The page must POST with `Content-Type: text/plain`. That keeps it a *simple request*
so the browser never sends a CORS preflight — Apps Script does not answer preflights.
Apps Script parses `e.postData.contents` as text either way, so nothing is lost.

The merge runs **on the server** inside a lock. The old design merged in the browser,
which meant two people saving at the same second could clobber each other. This is
strictly better than what we had.

## Seeding the existing data

The 6 ticks and 6 notes currently on the board are backed up in
`state-backup-2026-08-12.json`. Once the new URL is live they get pushed in as the
first write, so nothing has to be retyped.
