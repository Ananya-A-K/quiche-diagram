# Quiche–ns-3: File-wise Workflow Flowcharts

These flowcharts are based on the uploaded `quiche-ns3-main.zip` source tree. They describe the control flow in each implementation file, rather than a generic QUIC architecture.

## 1. `quiche-context.cc` — shared QUIC configuration

```mermaid
flowchart TD
    A["Create QuicheContext"] --> B["Constructor calls quiche_config_new(QUICHE_PROTOCOL_VERSION)"]
    B --> C{"Configuration allocated?"}
    C -->|No| X["Abort: configuration creation failed"]
    C -->|Yes| D["Set ALPN: ns3-quic"]
    D --> E["Configure maximum idle timeout"]
    E --> F["Configure connection-level flow-control limit"]
    F --> G["Configure bidirectional and unidirectional stream data limits"]
    G --> H["Configure maximum bidirectional and unidirectional stream counts"]
    H --> I["Configuration ready for client/server connections"]

    I --> J{"Additional configuration needed?"}
    J -->|Client trust store| K["LoadVerifyLocations(caFile, caPath)"]
    K --> L["Load CA file and/or CA directory"]
    L --> M["Return success/failure"]
    J -->|Server credentials| N["LoadCertificateAndKey(certFile, keyFile)"]
    N --> O["Load certificate chain and private key"]
    O --> P["Return success/failure"]

    I --> Q["GetConfig() returns quiche_config*"]
    Q --> R["QuicheConnection uses shared configuration"]
    I --> S["Destructor"]
    S --> T["quiche_config_free(m_config)"]
```

**Main role:** owns `quiche_config*` and applies common QUIC settings. The client and server create a context; the server additionally loads its certificate and private key. `LoadVerifyLocations()` is available for configuring client-side trust roots.

## 2. `quiche-sim-time.cc` — ns-3 clock bridge

```mermaid
flowchart TD
    A["QuicheSimTime::GetInstance()"] --> B["Return singleton instance"]
    B --> C["Update() called before time-sensitive Quiche operation"]
    C --> D["Read Simulator::Now()"]
    D --> E["Convert ns-3 time to microseconds"]
    E --> F["quiche_set_simulated_time_us(simulatedTimeUs)"]
    F --> G["Quiche internal clock now follows ns-3 simulation time"]

    H["GetCurrentTimeUs()"] --> I["Return current stored/simulated time value"]
```

**Main role:** keeps Quiche's time-dependent protocol calculations aligned with the ns-3 simulation clock. The bridge is used by connection setup and transport receive/send/timeout processing.

## 3. `quiche-connection.cc` — wrapper around one `quiche_conn`

```mermaid
flowchart TD
    A["Create QuicheConnection"] --> B["m_conn = nullptr"]
    B --> C{"Create endpoint state"}
    C -->|Client| D["ConnectClient(context, serverName, local, peer)"]
    D --> E["Convert local and peer InetSocketAddress to sockaddr_in"]
    E --> F["Generate unique 8-byte client SCID"]
    F --> G["QuicheSimTime::Update()"]
    G --> H["quiche_connect(..., context->GetConfig())"]
    H --> I{"m_conn created?"}
    I -->|No| X["Return false"]
    I -->|Yes| Z["Connection wrapper owns quiche_conn"]

    C -->|Server| J["AcceptServer(context, local, peer, SCID, ODCID)"]
    J --> K["Validate context, CIDs and lengths"]
    K --> L["Convert endpoints to sockaddr_in"]
    L --> M["QuicheSimTime::Update()"]
    M --> N["quiche_accept(..., context->GetConfig())"]
    N --> O{"m_conn created?"}
    O -->|No| X
    O -->|Yes| Z

    Z --> P["GetSourceConnectionIds()"]
    P --> Q["Iterate quiche_conn_source_ids()"]
    Q --> R["Return active source CIDs to application for indexing"]

    Z --> S{"Application requests stream operation"}
    S -->|Check state| T["IsEstablished() → quiche_conn_is_established()"]
    S -->|Open bidi| U["Check established and peer stream capacity"]
    U --> V["Initialize stream ID bits; reserve next bidi ID"]
    S -->|Open uni| W["Check established and peer stream capacity"]
    W --> W1["Initialize stream ID bits; reserve next uni ID"]
    S -->|Send data| Y["SendStreamData() → quiche_conn_stream_send()"]
    Y --> Y1["Release reserved stream ID after successful write / FIN"]
    S -->|Read data| Y2["ReceiveStreamData() → quiche_conn_stream_recv()"]
    S -->|Find readable streams| Y3["GetReadableStreamIds() → quiche_conn_readable() iterator"]
    S -->|Shutdown| Y4["ShutdownStream() → quiche_conn_stream_shutdown()"]

    Z --> Z1["GetConnection() exposes underlying quiche_conn*"]
    Z1 --> Z2["Transport calls Quiche C APIs through this pointer"]
    Z --> Z3["Destructor"]
    Z3 --> Z4["quiche_conn_free(m_conn)"]
```

**Main role:** owns exactly one Quiche connection and wraps the C API for connect/accept, connection state, source-CID retrieval, stream IDs, stream send/receive, and stream shutdown. It does not itself send UDP packets.

## 4. `quiche-transport.cc` — packet adapter and per-connection timer

```mermaid
flowchart TD
    A["QuicheTransport configured"] --> B["SetSocket(ns-3 UDP socket)"]
    B --> C["SetConnection(QuicheConnection)"]
    C --> D{"Transport operation"}

    D -->|Send pending QUIC output| E["SendPending()"]
    E --> F["Check socket and connection"]
    F --> G["QuicheSimTime::Update()"]
    G --> H["Get quiche_conn*"]
    H --> I["quiche_conn_send()"]
    I --> J{"Output datagram produced?"}
    J -->|No / done| N["ScheduleTimer()"]
    J -->|Yes| K["Read send metadata and destination sockaddr"]
    K --> L["Copy QUIC bytes into ns-3 Packet"]
    L --> M["Socket::SendTo(packet, destination)"]
    M --> I

    D -->|Receive UDP packet| P["ReceivePacket(packet, peer, local)"]
    P --> Q["Copy ns-3 Packet bytes into buffer"]
    Q --> R["Convert peer/local endpoints to sockaddr_in"]
    R --> S["Populate quiche_recv_info"]
    S --> T["QuicheSimTime::Update()"]
    T --> U["quiche_conn_recv(conn, bytes, recv_info)"]
    U --> V["ScheduleTimer()"]
    V --> W["SendPending() to flush generated responses"]

    N --> X["Query quiche_conn_timeout_as_nanos()"]
    X --> Y{"Finite timeout?"}
    Y -->|No: UINT64_MAX| Z["Cancel/leave no timeout event"]
    Y -->|Yes| AA["Schedule Simulator event for deadline"]
    AA --> AB["HandleTimeout() when event fires"]
    AB --> AC["QuicheSimTime::Update()"]
    AC --> AD["quiche_conn_on_timeout()"]
    AD --> AE["SendPending()"]
    AE --> X

    D -->|Stop| AF["Stop()"]
    AF --> AG["Cancel pending timer event and detach/clear transport state"]
```

**Main role:** converts between ns-3 `Packet`/socket-address objects and Quiche's byte-buffer/`sockaddr` interface. It sends all generated QUIC datagrams through the ns-3 UDP socket and schedules Quiche deadlines on the ns-3 event scheduler.

## 5. `quiche-client-app.cc` — client lifecycle, connection map and packet dispatch

```mermaid
flowchart TD
    A["Configure client before start"] --> B["SetRemote / AddRemote / SetLocalAddress / SetServerName"]
    B --> C["StartApplication()"]
    C --> D{"At least one remote configured?"}
    D -->|No| X["Abort"]
    D -->|Yes| E["Create UDP socket"]
    E --> F["Bind requested local address or wildcard"]
    F --> G["Create QuicheContext"]
    G --> H["Register HandleRead() callback; mark app running"]
    H --> I["For each configured remote"]
    I --> J["Validate peer IPv4 address and UDP port"]
    J --> K["Resolve local endpoint using bound socket / IPv4 route"]
    K --> L["Create QuicheConnection and QuicheTransport"]
    L --> M["Attach shared UDP socket to transport"]
    M --> N["Update simulated time; ConnectClient()"]
    N --> O["quiche_connect() creates client QUIC state"]
    O --> P["Attach connection to transport"]
    P --> Q["Get source CIDs; choose primary CID"]
    Q --> R["Insert primary-key connection record"]
    R --> S["Insert local/peer endpoint-pair index"]
    S --> T["Refresh CID-to-connection index"]
    T --> U["Transport::SendPending() sends Initial datagram"]
    U --> I

    V["Application API call for a peer"] --> W["FindConnection(peer) using endpoint-pair index"]
    W --> Y{"Connection exists / established as required?"}
    Y -->|No| Z["Return failure / INVALID_STREAM_ID"]
    Y -->|Yes| AA["Delegate stream operation to QuicheConnection"]
    AA --> AB["For send/shutdown/read-credit update, call transport SendPending() where required"]

    AC["HandleRead(socket)"] --> AD["Receive UDP datagram(s) in loop"]
    AD --> AE["Validate source is InetSocketAddress"]
    AE --> AF["Extract packet DCID using quiche_header_info()"]
    AF --> AG{"Header parse successful?"}
    AG -->|No| AH["Ignore malformed/unparseable datagram"]
    AG -->|Yes| AI["Lookup DCID in m_cidToConnection"]
    AI --> AJ{"CID maps to known connection?"}
    AJ -->|No| AK["Ignore packet"]
    AJ -->|Yes| AL["Resolve primary CID in m_connections"]
    AL --> AM["Call that connection's transport ReceivePacket()"]
    AM --> AN["Quiche processes packet; transport flushes response and updates timer"]
    AN --> AO["RefreshCidIndex(primary CID)"]

    AP["StopApplication()"] --> AQ["Mark app stopped; cancel handshake polling events"]
    AQ --> AR["Clear socket receive callback and close socket"]
    AR --> AS["Stop each connection transport"]
    AS --> AT["Clear connection, CID and endpoint-pair maps"]
    AT --> AU["Release shared context"]
```

**Main role:** manages client-side configuration and lifecycle, creates one connection record per configured remote, maintains endpoint-pair and CID indexes, exposes stream APIs, and dispatches received UDP packets to the matching transport.

## 6. `quiche-server-app.cc` — listening, session creation, CID demultiplexing and cleanup

```mermaid
flowchart TD
    A["Configure local IPv4 endpoint and certificate/key paths"] --> B["StartApplication()"]
    B --> C{"Specific local IPv4 and nonzero port configured?"}
    C -->|No| X["Abort"]
    C -->|Yes| D{"Certificate and key paths configured?"}
    D -->|No| X
    D -->|Yes| E["Create QuicheContext"]
    E --> F["LoadCertificateAndKey()"]
    F --> G{"Credentials loaded?"}
    G -->|No| X
    G -->|Yes| H["Create UDP socket and bind local endpoint"]
    H --> I["Register HandleRead() callback"]
    I --> J["Schedule periodic closed-session cleanup"]
    J --> K["Server listens for UDP datagrams"]

    K --> L["HandleRead(): receive datagram and source peer"]
    L --> M["Parse QUIC header and extract DCID"]
    M --> N{"Header valid?"}
    N -->|No| L
    N -->|Yes| O["Lookup DCID in m_cidToSession"]
    O --> P{"Known CID?"}
    P -->|Yes| Q["Resolve session by primary CID"]
    Q --> R{"Session exists and source endpoint pair matches?"}
    R -->|No| S["Drop packet"]
    R -->|Yes| V["Dispatch to session transport"]

    P -->|No| T["Check minimum Initial size; parse header, type and version"]
    T --> U{"Supported-version Initial?"}
    U -->|No| S
    U -->|Yes| U1{"Session already exists for endpoint pair?"}
    U1 -->|Yes| S
    U1 -->|No| U2["CreateSession(peer, local, packet)"]
    U2 --> U3["Parse client SCID and original DCID"]
    U3 --> U4["Generate server SCID"]
    U4 --> U5["Create QuicheConnection; call AcceptServer()"]
    U5 --> U6{"quiche_accept() successful?"}
    U6 -->|No| S
    U6 -->|Yes| U7["Create QuicheTransport; attach server UDP socket and connection"]
    U7 --> U8["Read server source CIDs; select primary CID"]
    U8 --> U9["Insert session record and endpoint-pair index"]
    U9 --> U10["Refresh CID index, including client's original DCID"]
    U10 --> V

    V --> W["QuicheTransport::ReceivePacket(packet, peer, local)"]
    W --> Y["Refresh CID index after Quiche processes packet"]
    Y --> Z{"Connection closed or timed out?"}
    Z -->|Yes| ZA["RemoveSession(primary CID)"]
    Z -->|No| K

    ZB["Periodic CleanupClosedSessions()"] --> ZC["Scan all sessions"]
    ZC --> ZD{"Connection null, closed or timed out?"}
    ZD -->|Yes| ZE["Collect primary CID for removal"]
    ZD -->|No| ZF["Keep session"]
    ZE --> ZA
    ZF --> ZG["Schedule next cleanup after 1 second"]
    ZA --> ZG
    ZG --> ZB

    ZH["Server application stream API"] --> ZI["FindSession(peer) via endpoint-pair index"]
    ZI --> ZJ{"Session / established connection available?"}
    ZJ -->|No| ZK["Return failure"]
    ZJ -->|Yes| ZL["Delegate stream operation to QuicheConnection"]
    ZL --> ZM["SendPending() after send, shutdown or receive-credit release where required"]

    ZN["StopApplication()"] --> ZO["Cancel cleanup and handshake polling events"]
    ZO --> ZP["Clear callback and close UDP socket"]
    ZP --> ZQ["Stop each session transport"]
    ZQ --> ZR["Clear session, CID and endpoint-pair maps; release context"]
```

**Main role:** listens for QUIC traffic, creates a session only from a supported-version Initial packet with an unknown DCID, maps active CIDs to sessions, dispatches packets, exposes server-side stream operations, and periodically removes closed or timed-out sessions.

## How the six files work together

```mermaid
flowchart TD
    CTX["quiche-context.cc<br/>Shared QUIC configuration"] --> CLIENT["quiche-client-app.cc<br/>Client lifecycle and connection map"]
    CTX --> SERVER["quiche-server-app.cc<br/>Server lifecycle and session map"]
    CLIENT --> CONN_C["quiche-connection.cc<br/>Client quiche_conn wrapper"]
    SERVER --> CONN_S["quiche-connection.cc<br/>Server quiche_conn wrapper"]
    CONN_C --> TRANS_C["quiche-transport.cc<br/>Client packet and timer adapter"]
    CONN_S --> TRANS_S["quiche-transport.cc<br/>Server packet and timer adapter"]
    TIME["quiche-sim-time.cc<br/>ns-3 simulated clock bridge"] -.-> CONN_C
    TIME -.-> CONN_S
    TIME -.-> TRANS_C
    TIME -.-> TRANS_S
    TRANS_C <--> UDP["ns-3 UDP sockets and simulated network"]
    TRANS_S <--> UDP
```

## Important reading order

1. `quiche-context.cc` — understand how Quiche is configured.
2. `quiche-connection.cc` — understand the C API wrapper and stream operations.
3. `quiche-sim-time.cc` — understand the simulator-clock bridge.
4. `quiche-transport.cc` — understand packet conversion and timer scheduling.
5. `quiche-client-app.cc` — follow client startup, Initial transmission and receive dispatch.
6. `quiche-server-app.cc` — follow Initial validation, session creation, CID lookup and cleanup.

## Source-grounding notes

- The diagrams describe the uploaded source snapshot. They do not assume features that are not implemented in these files.
- The client and server use IPv4 `InetSocketAddress` / `sockaddr_in` conversion in this implementation.
- Server packet dispatch checks that the peer endpoint pair matches the session's original pair; this constrains connection migration.
- `quiche_connect()` and `quiche_accept()` create endpoint state; they do not by themselves mean the TLS handshake has completed. The wrapper checks `quiche_conn_is_established()`.
- The server's periodic cleanup is scheduled at one-second intervals.
- `quiche-transport.cc` owns the ns-3 timeout event; Quiche supplies the timeout deadline and processes it when `quiche_conn_on_timeout()` is called.
