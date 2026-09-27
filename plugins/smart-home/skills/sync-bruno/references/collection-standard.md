# Bruno collection `home` — conventions

Recorded from the collection on 2026-09-27. Location: `D:\OneDrive\Bruno\home` (OneDrive, not
a git repo). Follow these conventions for everything added or changed; when the files show
something different, the files win — mention the difference to the user.

## Layout

```
home/
├── bruno.json                  collection config (name "smart-home")
├── environments/
│   ├── home-local.bru          vars { baseUrl }  — services running locally
│   └── home-k8s.bru            vars { baseUrl }  — the cluster
├── <service>/                  one folder per service, short name without "-service"
│   ├── folder.bru              name, seq, auth inherit, port script (below)
│   ├── <request>.bru           requests directly in the service folder…
│   └── <domain>/               …or grouped by domain when the service has several
│       ├── folder.bru          name, seq, auth inherit (no script)
│       └── <request>.bru
```

Service folders today: `ai`, `amx`, `boiler`, `database` (subfolders `eaton`, `household`),
`heating`, `notification`, `presence` (subfolder `ubiquity` — direct calls to the UniFi
controller, not to a service), `water`. There is no folder for `api-gateway-service` or
`shelly-cloud-service`.

`bruno.json` also carries an `items` list that is out of sync with the folders (lists a
`gateway` folder that does not exist, misses `ai`, `amx`, `presence`). Leave it alone unless
the user asks; if a change touches it, propose aligning it with the actual folders.

## Service folder — `folder.bru`

`seq` follows the alphabetical order of the service folders (ai 2, amx 3, boiler 4,
database 5, heating 6, notification 7, presence 8, water 9). A new service folder goes into
that order — propose renumbering the following folders only if the user wants the order kept.

The pre-request script picks the port: the service's local application port (600x, from the
repository map in `organization.md`) for `home-local`, the global environment variable
`k8s-one-port` otherwise.

```
meta {
  name: <service>
  seq: <n>
}

auth {
  mode: inherit
}

script:pre-request {
  const env = bru.getEnvName();
  
  if (env === 'home-local') {
    bru.setVar('port', '<600x>');
  } else {
    bru.setVar('port', bru.getGlobalEnvVar('k8s-one-port'));
  }
}
```

Local ports in use: ai 6004, amx 6001, boiler 6007, database 6005, heating 6002,
notification 6003, presence 6009, water 6006 (shelly-cloud would be 6008).

## Domain subfolder — `folder.bru`

```
meta {
  name: <domain>
  seq: <n>
}

auth {
  mode: inherit
}
```

## Request files

- **File name = `meta.name`**, camelCase verb + noun: `get…` (read), `add…` (create),
  `update…` (change), `delete…` (remove), or a specific verb for an action endpoint
  (`sendMessage`, `activate…`). Singular for one item, plural for a list (`getHouseholds`).
- **`seq`** orders requests inside their folder; a new request takes the next free number.
  Existing requests keep theirs.
- **URL** always `{{baseUrl}}:{{port}}/home/...` for services (the port comes from the folder
  script). Direct calls to external systems keep their own base URL (see `presence/ubiquity`).
- **Query parameters** appear both in the URL and in a `params:query` block (Bruno keeps
  them in sync). Alternative values the user toggles are kept as disabled entries with a `~`
  prefix (`~gateway: lights`).
- **Path variables** use Bruno's `:name` syntax in the URL plus a `params:path` block.
- **Body**: `body: json` + a `body:json` block for requests with a body; `body: none` and **no
  `body:json` block** otherwise (several existing GET/DELETE requests still carry a leftover
  body copied from another request — that is drift to fix, not a convention).
- **Auth** `inherit`; secrets only as variables (`{{ubquityToken}}` is a global environment
  variable), never literal values.
- **`settings`** block always the same.

### GET with query parameters

```
meta {
  name: getEatonDevice
  type: http
  seq: 1
}

get {
  url: {{baseUrl}}:{{port}}/home/device/configuration/eaton?point=48&gateway=blinds
  body: none
  auth: inherit
}

params:query {
  point: 48
  gateway: blinds
  ~gateway: lights
}

settings {
  encodeUrl: true
  timeout: 0
}
```

### POST with a JSON body

```
meta {
  name: addEatonDevice
  type: http
  seq: 2
}

post {
  url: {{baseUrl}}:{{port}}/home/device/configuration/eaton
  body: json
  auth: inherit
}

body:json {
  {
    "point": 57,
    "type": "temperature sensor",
    "gateway": "blinds",
    "room": "garage"
  }
}

settings {
  encodeUrl: true
  timeout: 0
}
```

### Request with a path variable

```
meta {
  name: deleteHousehold
  type: http
  seq: 3
}

delete {
  url: {{baseUrl}}:{{port}}/home/household/member/:name
  body: none
  auth: inherit
}

params:path {
  name: Test
}

settings {
  encodeUrl: true
  timeout: 0
}
```

## Example values

Values already in a request belong to the user and stay, real names and numbers included —
if one no longer passes validation, only its format changes (`696027072` → `+48696027072`).
Values introduced by a change (a new request, a new field or path variable) are fictional and
valid: names like `Test`, phones like `+48500000001` (E.164), MACs like `02:00:00:00:00:01`
(lowercase, colon-separated, locally administered).
