- [FIXED]
Fill blank in template does not work: 
[auth][warn][debug-enabled] Read more: https://warnings.authjs.dev
⨯ TypeError: Cannot read properties of undefined (reading 'validateAnswer')
    at c.ActivityCoordinator.validateAnswer (.next/server/chunks/ssr/packages_0wbe0fc._.js:1:1872)
    at C.validateAnswer (.next/server/chunks/ssr/packages_0wbe0fc._.js:2:28669)
    at A (.next/server/chunks/ssr/[root-of-the-server]__0b3fti9._.js:1:6547)
    at async l (.next/server/chunks/ssr/0_us_next_dist_008g4s9._.js:1:10953)
    at async o (.next/server/chunks/ssr/0_us_next_dist_008g4s9._.js:2:4523) {
  digest: '1780220312'
}
---
- [FIXED]
Discuss the exercise with eibot does not work
⨯ Error: template not found
    at k.buildAssistanceYAML (.next/server/chunks/ssr/packages_1w2v86z._.js:1:14000)
    at async k.createOrLoad (.next/server/chunks/ssr/packages_1w2v86z._.js:1:12724)
    at async C.discussWithEibot (.next/server/chunks/ssr/packages_0wbe0fc._.js:2:26531)
    at async C.discussNamedComponentWithEibot (.next/server/chunks/ssr/packages_0wbe0fc._.js:2:25120)
    at async H (.next/server/chunks/ssr/[root-of-the-server]__0b3fti9._.js:1:7648) {
  digest: '2168686680'
}

---
- [FIXED]
Discussions are not counted as part of a workbook
---

- [FIXED]

Profile does not work:
⨯ Error [HTTPError]: Request failed with status code 404 Not Found: GET http://eivo-cloud-api.default/profile/384074779261141027
    at <unknown> (HTTPError: Request failed with status code 404 Not Found: GET http://eivo-cloud-api.default/profile/384074779261141027)
    at f (.next/server/chunks/ssr/_17tu7vd._.js:1:48755)
    at async #g (.next/server/chunks/ssr/_17tu7vd._.js:1:54136)
    at async e.getJson (.next/server/chunks/ssr/_17tu7vd._.js:1:57580)
    at async e.getProfile (.next/server/chunks/ssr/[root-of-the-server]__047cc9f._.js:2:20420)
    at async i (.next/server/chunks/ssr/[root-of-the-server]__0pgzy8v._.js:1:6229) {
  response: Response {
    status: 404,
    statusText: 'Not Found',
    headers: Headers {
      'x-powered-by': 'Express',
      'content-type': 'application/json; charset=utf-8',
      'content-length': '95',
      etag: 'W/"5f-4JOVM0vdLmDFraTVIHgs5svpP+g"',
      date: 'Sun, 30 Aug 2026 14:43:38 GMT',
      connection: 'keep-alive',
      'keep-alive': 'timeout=5'
    },
    body: ReadableStream { locked: false, state: 'readable', supportsBYOB: true },
    bodyUsed: false,
    ok: false,
    redirected: false,
    type: 'default',
    url: 'http://eivo-cloud-api.default/profile/384074779261141027'
  },
  request: Request {
    method: 'GET',
    url: 'http://eivo-cloud-api.default/profile/384074779261141027',
    headers: Headers { 'content-type': 'application/json' },
    destination: '',
    referrer: 'about:client',
    referrerPolicy: '',
    mode: 'cors',
    credentials: 'same-origin',
    cache: 'default',
    redirect: 'follow',
    integrity: '',
    keepalive: false,
    isReloadNavigation: false,
    isHistoryNavigation: false,
    signal: AbortSignal { aborted: false }
  },
  options: {
    fetch: [Function (anonymous)],
    timeout: false,
    signal: AbortSignal { aborted: false },
    headers: Headers { 'content-type': 'application/json' },
    method: 'GET',
    prefixUrl: '',
    retry: {
      limit: 2,
      methods: [Array],
      statusCodes: [Array],
      afterStatusCodes: [Array],
      maxRetryAfter: Infinity,
      backoffLimit: Infinity,
      delay: [Function: delay],
      jitter: undefined,
      retryOnTimeout: false
    },
    throwHttpErrors: true,
    context: {},
    duplex: 'half'
  },
  digest: '2372202069'
}

NotFoundException: Profile with ID 384074779261141027 not found
    at UserProfileController.getPersona (/app/main.js:572854:19)
    at process.processTicksAndRejections (node:internal/process/task_queues:95:5)

---

- [FIXED]

Fill in the blank don't make any sense. The prompts generated for the eivolet are wrong.

---

- [FIXED]

All LLM generations should include include the eivolet invariants and the aggregate descriptions.

---
- [FIXED]

Fill in the blank, Mnemonics, Multiple Choice close button is not rendered correctly

---

- [FIXED]

Mnemonics < Done > takes too much time

---
- [FIXED]

Multiple Choice Skeleton is Fill blank one

---

- [FIXED]

Multiple Choise Ok icon has issues

---
- [FIXED]

Feeding new exercise and v