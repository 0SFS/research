{
  "version": 1,
  "capturedAt": "2026-10-22T01:56:31.863Z",
  "outcome": "failed",
  "elapsedMs": 336,
  "failure": "The application could not initialize. Check your connection and reload to try again.",
  "cause": {
    "message": "SyntaxError: The requested module 'http://localhost:7470/src/flight/physics/fixedStepLoop.ts[redacted]' doesn't provide an export named: 'FIXED_DT'",
    "stack": ""
  },
  "phases": {
    "app": {
      "state": "loading",
      "detail": "Downloading application code"
    },
    "world": {
      "state": "waiting"
    },
    "flight": {
      "state": "waiting"
    },
    "assets": {
      "state": "waiting"
    },
    "terrain": {
      "state": "waiting"
    }
  },
  "history": [
    {
      "elapsedMs": 0,
      "phase": "app",
      "update": {
        "state": "loading",
        "detail": "Downloading application code"
      }
    }
  ]
}
