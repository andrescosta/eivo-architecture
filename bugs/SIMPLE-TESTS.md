

---

- Multiple Choice, when clicking OK, must disbale the screen and how a waiting.

---

- Realtime CORS is broken:

Access to XMLHttpRequest at 'http://realtime.jobico.local/socket.io/?EIO=4&transport=polling&t=5vsk4g6u' from origin 'https://lingv.jobico.local' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header has a value 'http://lingv.jobico.local' that is not equal to the supplied origin.

---

- All Chat interactions are broken, error:

Failed to load resource: the server responded with a status of 401 (Unauthorized)

[stream] {"message":"Invalid token","error":"Unauthorized","statusCode":401

<><><><> JsonWebTokenError: jwt audience invalid. expected: 346403198690459683
    at /app/main.js:83882:21
    at /app/main.js:82698:17
    at process.processTicksAndRejections (node:internal/process/task_queues:82:21)
{
  error: JsonWebTokenError: jwt audience invalid. expected: 346403198690459683
      at /app/main.js:83882:21
      at /app/main.js:82698:17
      at process.processTicksAndRejections (node:internal/process/task_queues:82:21),
  message: 'jwt audience invalid. expected: 346403198690459683'
}