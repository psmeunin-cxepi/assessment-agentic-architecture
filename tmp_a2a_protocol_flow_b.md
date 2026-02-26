# A2A Protocol Flow — CBP Task-first Pattern (input-required)

## Sequence Overview

```
User : "Show me the configuration assessment from my device?"
  SMR → CBP : Message(Text: "Show me the configuration assessment from my device?")

CBP → SMR : Task(Id: "cbp-123", ContextId: "001", Status: "input-required",
              Message: "Which device do you refer to?")

User : "The device name is router007"
  SMR → CBP : Message(TaskId: "cbp-123", ContextId: "001",
              Text: "The device name is router007")

CBP → SMR : Task(Id: "cbp-123", ContextId: "001", Status: "working", Message: "I am on it")
```

---

## Turn 1 — User asks about device assessment

### SMR → CBP Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "SendMessage",
  "params": {
    "message": {
      "messageId": "msg-001",
      "role": "ROLE_USER",
      "parts": [
        { "text": "Show me the configuration assessment from my device?" }
      ]
    },
    "configuration": {
      "acceptedOutputModes": ["text/plain", "application/json"]
    }
  }
}
```

### CBP → SMR Response (Task created immediately, needs more input)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "id": "cbp-123",
      "contextId": "001",
      "status": {
        "state": "TASK_STATE_INPUT_REQUIRED",
        "timestamp": "2026-02-25T14:30:00.000Z",
        "message": {
          "messageId": "msg-002",
          "role": "ROLE_AGENT",
          "parts": [
            { "text": "Which device do you refer to?" }
          ]
        }
      }
    }
  }
}
```

---

## Turn 2 — User provides the device name

### SMR → CBP Request (references existing taskId and contextId)

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "SendMessage",
  "params": {
    "message": {
      "messageId": "msg-003",
      "role": "ROLE_USER",
      "taskId": "cbp-123",
      "contextId": "001",
      "parts": [
        { "text": "The device name is router007" }
      ]
    },
    "configuration": {
      "acceptedOutputModes": ["text/plain", "application/json"]
    }
  }
}
```

### CBP → SMR Response (same Task, now processing)

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "task": {
      "id": "cbp-123",
      "contextId": "001",
      "status": {
        "state": "TASK_STATE_WORKING",
        "timestamp": "2026-02-25T14:30:05.000Z",
        "message": {
          "messageId": "msg-004",
          "role": "ROLE_AGENT",
          "parts": [
            { "text": "I am on it" }
          ]
        }
      }
    }
  }
}
```

---

## Turn 3 — SMR polls for completion

### SMR → CBP Request

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "GetTask",
  "params": {
    "id": "cbp-123",
    "historyLength": 5
  }
}
```

### CBP → SMR Response (Task completed with artifact)

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "task": {
      "id": "cbp-123",
      "contextId": "001",
      "status": {
        "state": "TASK_STATE_COMPLETED",
        "timestamp": "2026-02-25T14:30:12.000Z"
      },
      "artifacts": [
        {
          "artifactId": "artifact-001",
          "name": "Configuration Assessment for router007",
          "parts": [
            {
              "text": "Assessment summary for router007:\n- 42 checks performed\n- 5 critical findings\n- 12 high severity findings\n- 25 passed"
            }
          ]
        }
      ]
    }
  }
}
```

---

## Protocol Summary

| Element | Turn 1 | Turn 2 | Turn 3 |
|---|---|---|---|
| Method | `SendMessage` | `SendMessage` | `GetTask` |
| Response type | `Task` | `Task` | `Task` |
| `contextId` | absent → generated `"001"` | echoed `"001"` | inherited via taskId |
| `taskId` | `"cbp-123"` generated | `"cbp-123"` referenced | `"cbp-123"` polled |
| Task state | `TASK_STATE_INPUT_REQUIRED` | `TASK_STATE_WORKING` | `TASK_STATE_COMPLETED` |
| Results in | clarification (tracked) | processing started | artifact delivered |

---

**Source:** A2A Protocol Specification (RC v1.0), Sections 3.1.1, 3.4.1, 3.4.2, 3.4.3, 6.3, 9.4
