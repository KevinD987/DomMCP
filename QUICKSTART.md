# DomMCP Quickstart

## 1) Check tools inventory
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

## 2) Profile a database
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{
    "jsonrpc":"2.0",
    "id":2,
    "method":"tools/call",
    "params":{
      "name":"profile_database",
      "arguments":{"database_path":"log.nsf","grant_id":""}
    }
  }'
```

## 3) Read view entries
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{
    "jsonrpc":"2.0",
    "id":3,
    "method":"tools/call",
    "params":{
      "name":"read_view_entries",
      "arguments":{
        "database_path":"names.nsf",
        "view":"People",
        "limit":25,
        "offset":0,
        "grant_id":""
      }
    }
  }'
```

## 4) Extract log lines
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{
    "jsonrpc":"2.0",
    "id":4,
    "method":"tools/call",
    "params":{
      "name":"extract_log_lines",
      "arguments":{
        "database_path":"log.nsf",
        "view":"Miscellaneous Events|MiscEvents",
        "filter":"error",
        "limit":100,
        "grant_id":""
      }
    }
  }'
```

## 5) DQL query (Professional/Enterprise)
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{
    "jsonrpc":"2.0",
    "id":5,
    "method":"tools/call",
    "params":{
      "name":"query_documents_dql",
      "arguments":{
        "database_path":"Haushaltsbuch.nsf",
        "query":"Form = '\''fEinkaufszettel'\''",
        "return_mode":"ids_only",
        "limit":10,
        "grant_id":""
      }
    }
  }'
```

## 6) Batch fetch projected fields
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{
    "jsonrpc":"2.0",
    "id":6,
    "method":"tools/call",
    "params":{
      "name":"get_documents_batch",
      "arguments":{
        "database_path":"Haushaltsbuch.nsf",
        "unids":["<UNID_1>","<UNID_2>"],
        "fields":["Datum","Lieferant","Gesamtbetrag"],
        "grant_id":""
      }
    }
  }'
```
