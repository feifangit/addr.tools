# dnscheck.tools Workflow Diagram

```mermaid
graph LR
    subgraph Browser
        A[Load dnscheck.tools UI]
        B[Generate clientId]
        C["Open WebSocket to /watch/&lt;clientId&gt;"]
        D[Issue HTTPS fetches for test hostnames]
    end

    subgraph ReverseProxy
        RP1[Serve static UI]
        RP2[Proxy /watch to addrd]
    end

    subgraph addrd Backend
        E[WebsocketWatcher registers client]
        F[ParseOptions extracts toggles & clientId]
        G[Fabricate DNS responses per options]
        H[Stream resolver metadata over WebSocket]
    end

    subgraph Authoritative DNS Flow
        I[Recursive resolver queries test zone]
        J[addrd ServeDNS receives query]
        K[Capture resolver IP, EDNS, TLS]
        L["Send response (A/AAAA, REFUSED, DNSSEC variants)"]
    end

    subgraph Browser_UI
        M[Update resolver list & feature badges]
    end

    A --> B --> C --> D
    A --> RP1
    C --> RP2 --> E
    D -->|Triggers lookups| I --> J --> F --> G --> L
    J --> K --> H --> M
    H --> C
```

**Flow description**

1. The browser loads the dnscheck.tools UI, generates a random `clientId`, opens a WebSocket watcher, and issues repeated HTTPS fetches whose hostnames encode desired DNS behaviors.
2. The reverse proxy serves the static application and forwards WebSocket traffic to the `addrd` backend.
3. Recursive resolvers that the user relies on contact the authoritative handler inside `addrd`, which parses query options, fabricates matching DNS responses (including intentional REFUSED or DNSSEC variants), and records resolver metadata.
4. `addrd` streams the observed resolver addresses, transport details, and DNSSEC outcomes back to the browser over the watcher channel, allowing the UI to list each resolver and badge its capabilities in real time.
