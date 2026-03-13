## paper-2 via Redis LRU Eviction
[paper-2](https://play.picoctf.org/practice/challenge/726) was the hardest challenge in [picoCTF 2026](https://play.picoctf.org/events/79/scoreboards)

i got really sick in the middle of solving paper-2 so that messed up my timings a lot and I ended up submitting the full solve wednesday morning 💀

`paper-2` can be solved by using CSS attribute selectors on `/secret` to make the bot fetch specific uploaded files only when parts of the secret matched, then forcing Redis `allkeys-lru` eviction and checking which files survived to reconstruct the secret and query `/flag`.

In [index.ts](./index.ts), we see `/secret` reflects the secret into a DOM attribute and also renders attacker-controlled `payload`:

```ts
const secret = req.cookies.get('secret') || '0123456789abcdef'.repeat(2);
const payload = new URL(req.url, 'http://127.0.0.1').searchParams.get('payload') || '';

return new Response(
  `<body secret="${secret}">${secret}\n${payload}</body>`,
  headers('text/html')
);
```

That means CSS can test the secret with selectors like:

```css
body[secret*="abc"]
body[secret^="de"]
body[secret$="f0"]
```

We can see in [docker-compose.yml](./docker-compose.yml) that Redis is configured with LRU eviction:

```yaml
command: redis-server --maxmemory 512M --maxmemory-policy allkeys-lru --save "" --appendonly no
```

From [index.ts](./index.ts), uploaded files are stored in Redis and later served back from `/paper/:id`:

```ts
const id = await redis.incr('current-id');
const data = JSON.stringify([file.type, (await file.bytes()).toBase64()]);
await redis.set(`file|${id}`, data, 'EX', 10 * 60);
```

So the exploit path is:

1. Upload many marker files.
2. Inject CSS that only references those marker files when parts of the secret match.
3. Make the bot visit that CSS.
4. Force Redis eviction with lots of padding uploads.
5. Check which marker files survived.
6. Reconstruct the secret from the surviving markers.
7. Request `/flag?secret=...`.

## exploit

a full exploit is in [exploit.js](./exploit.js).

Stage 1:

- Upload marker files for every hex triplet (`000` through `fff`).
- Upload marker files for every hex prefix/suffix bigram (`00` through `ff`).
- Upload CSS rules that conditionally fetch those markers:
  - `body[secret*="triplet"]`
  - `body[secret^="prefix"]`
  - `body[secret$="suffix"]`
- Build HTML frames that load `/secret?payload=...` with those CSS rules.
- Visit a “master ring” page that keeps the bot loading the attack frames.
- Flood Redis so that bot-touched files are more likely to survive than untouched ones.
- Read back the marker files and score which ones survived.

Stage 2:

- Build candidate 32-byte hex secrets from the strongest triplets plus the best prefix and suffix.
- If there is only one candidate, immediately fetch `/flag`.
- If there are multiple candidates, upload exact-match CSS rules:

```css
body[secret="candidate"]
```

- Repeat the eviction test and keep the unique survivor.
- The exploit is noisy and not perfectly deterministic. Some runs collapse to one candidate quickly; others need the exact-match stage; some fail and need a retry.


for example, a successful run's output:

```text
{
  maxTripletScore: 2,
  maxPrefixScore: 4,
  maxSuffixScore: 4,
  prefixes: [ '8d' ],
  suffixes: [ '44', '94' ]
}
{
  threshold: 2,
  triplets: 39,
  candidates: 1,
  sample: [ '8d07ce0e1c8b88859d18010a84eab094' ]
}
flag response 200 picoCTF{i_l1ke_frames_on_my_canvas_xxxxxxx}
```
