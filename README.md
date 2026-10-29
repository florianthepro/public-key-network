# Dezentrale Netzwerkarchitektur
```
Projektname: Public Key Network
Kurzname:    PKN
Repo:        public-key-network
Test-TLD:    .pkn.alt
Ziel-TLD:    .pkn
```
```mermaid
flowchart TB

subgraph LEGEND["Legende"]
    direction LR
    L1["Dauerhaft und finalisiert"]
    L2["Flüchtig, nur lokaler TTL-Cache"]
    L3["Ende-zu-Ende verschlüsselt"]
    L4["Signierte äußere Hülle"]
    L5["Lokale geheime Daten"]
end

subgraph KEY_SYSTEM["Deterministische Schlüsselarchitektur je Teilnehmer"]
    direction TB
    MASTER_SEED["Master Seed<br/>softwaregeneriert<br/>mindestens 128 Bit Entropie<br/>menschenlesbar kodiert + Prüfsumme<br/>niemals übertragen"]
    KDF["Standardisierte KDF<br/>Eingabe:<br/>Master Seed + Network ID + Key Purpose + Key Version"]
    MASTER_SEED -->|"lokal"| KDF
    ID_KEY["Identity Key Pair<br/>ID_PRIV: möglichst selten verwenden<br/>ID_PUB: dauerhafte Identität"]
    REG_KEY["Registry Key Pair<br/>REG_PRIV / REG_PUB<br/>Domainoperationen"]
    DISC_KEY["Discovery Key Pair<br/>DISC_PRIV / DISC_PUB<br/>Discovery-Delegationen"]
    TRANSPORT_KEY["Transport Key Pair<br/>TRANS_PRIV / TRANS_PUB<br/>Handshake-Authentifizierung"]
    RECOVERY_KEY["Recovery Key Pair<br/>optional offline<br/>Key-Recovery oder Rotation"]
    KDF -->|"purpose = identity"| ID_KEY
    KDF -->|"purpose = registry"| REG_KEY
    KDF -->|"purpose = discovery"| DISC_KEY
    KDF -->|"purpose = transport"| TRANSPORT_KEY
    KDF -->|"purpose = recovery"| RECOVERY_KEY
    subgraph DELEGATION_SIGNATURE["Signierte Delegation"]
        direction TB
        DELEGATION_DATA["Delegation Data<br/>identity_public_key<br/>registry_public_key<br/>discovery_public_key<br/>transport_public_key<br/>key_version<br/>valid_from<br/>valid_until"]
        DELEGATION_SIG["Signatur mit ID_PRIV<br/>SIGN HASH canonical Delegation Data"]
        DELEGATION_DATA --> DELEGATION_SIG
    end
    ID_KEY -->|"ID_PRIV signiert"| DELEGATION_SIG
    REG_KEY --> DELEGATION_DATA
    DISC_KEY --> DELEGATION_DATA
    TRANSPORT_KEY --> DELEGATION_DATA
end

subgraph BLOCKCHAIN["Dezentrale Registry-Blockchain"]
    direction TB
    MEMPOOL["Validierte dauerhafte Transaktionen<br/>keine Locate Requests<br/>keine IP-Adressen<br/>keine Nutzdaten"]
    CONSENSUS["Dezentraler Konsens mit harter Finalität<br/>Validatoren prüfen unabhängig<br/>ohne Quorum keine Registry-Änderung<br/>keine zwei finalen Besitzstände"]
    BLOCK["Finalisierter Block<br/>previous_block_hash<br/>height<br/>timestamp_or_slot<br/>transaction_merkle_root<br/>state_root<br/>validator_commitment"]
    STATE["Aktueller Registry State<br/>pruning- und snapshotfähig"]
    MEMPOOL --> CONSENSUS
    CONSENSUS --> BLOCK
    BLOCK --> STATE
    subgraph DOMAIN_RECORD_SIGNATURE["Signierte Domain-Transaktion"]
        direction TB
        DOMAIN_RECORD["Domain Record<br/>domain_hash<br/>owner_identity_public_key<br/>delegation_hash<br/>sequence_number<br/>registered_at<br/>expires_at = maximal 1 Jahr<br/>network_id"]
        DOMAIN_SIGNATURE["Signatur mit REG_PRIV<br/>zusätzlich durch Identity Delegation autorisiert"]
        DOMAIN_RECORD --> DOMAIN_SIGNATURE
    end
    subgraph SESSION_COMMITMENT_SIGNATURE["Signiertes Session Commitment"]
        direction TB
        SESSION_RECORD["Session Commitment<br/>identity_public_key<br/>session_public_key<br/>epoch<br/>capabilities<br/>valid_from<br/>valid_until<br/>monotonic_counter"]
        SESSION_SIGNATURE["Signatur mit DISC_PRIV<br/>DISC_PUB ist durch ID_PRIV delegiert"]
        SESSION_RECORD --> SESSION_SIGNATURE
    end
    subgraph ROTATION_SIGNATURE["Atomarer Key-Swap"]
        direction TB
        ROTATION_RECORD["Rotation Record<br/>old_identity_public_key<br/>new_identity_public_key<br/>domain_set_root<br/>new_delegation_hash<br/>sequence_number<br/>effective_height"]
        OLD_SIGNATURE["Signatur mit altem ID_PRIV<br/>Übertragung autorisiert"]
        NEW_SIGNATURE["Signatur mit neuem ID_PRIV<br/>Übernahme akzeptiert"]
        ROTATION_RECORD --> OLD_SIGNATURE
        ROTATION_RECORD --> NEW_SIGNATURE
    end
    subgraph RENEWAL_SIGNATURE["Domainverlängerung"]
        direction TB
        RENEWAL_RECORD["Renewal Record<br/>domain_hash<br/>owner_identity_public_key<br/>previous_sequence + 1<br/>new_expires_at"]
        RENEWAL_SIGNATURE_DATA["Signatur mit gültigem REG_PRIV"]
        RENEWAL_RECORD --> RENEWAL_SIGNATURE_DATA
    end
    DOMAIN_SIGNATURE --> MEMPOOL
    SESSION_SIGNATURE --> MEMPOOL
    OLD_SIGNATURE --> MEMPOOL
    NEW_SIGNATURE --> MEMPOOL
    RENEWAL_SIGNATURE_DATA --> MEMPOOL
    STATE --> DOMAIN_INDEX["Domain Index<br/>domain_hash → Identity Public Key"]
    STATE --> DELEGATION_INDEX["Delegation Index<br/>Identity → autorisierte Funktionskeys"]
    STATE --> SESSION_INDEX["Session Index<br/>Identity → aktuell gültige Session Keys"]
    STATE --> REVOCATION_INDEX["Revocation Index<br/>widerrufene und rotierte Keys"]
    STATE --> EXPIRY_INDEX["Expiry Index<br/>ACTIVE → GRACE → AVAILABLE"]
    STATE --> STATE_ROOT["Finalisierter State Root<br/>Basis für Merkle Proofs"]
end

DELEGATION_SIG --> MEMPOOL
ID_KEY --> DOMAIN_RECORD
REG_KEY --> DOMAIN_SIGNATURE
DISC_KEY --> SESSION_SIGNATURE
ID_KEY --> OLD_SIGNATURE
ID_KEY --> NEW_SIGNATURE

subgraph SOURCE_NODE["Node S: Quelle, Client, Server und Protokollteilnehmer"]
    direction TB
    SOURCE_APP["Browser Plugin / Anwendung / lokaler Resolver"]
    SOURCE_CHAIN["Lokaler Chain-Modus<br/>Full, Pruned oder Compact<br/>validiert finalisierte Header und Proofs"]
    SOURCE_ROUTING["Lokale Routingtabelle<br/>Peer Public Key<br/>bestehender verschlüsselter Kanal<br/>Key-Distanzbereich<br/>Capabilities<br/>Fehlerstatus"]
    SOURCE_CACHE["Flüchtiger TTL-Cache<br/>Request IDs<br/>Reverse Routes<br/>Antwortstatus<br/>automatische Löschung"]
    SOURCE_SESSION["Kurzlebiges Session Key Pair<br/>S_SESSION_PRIV / S_SESSION_PUB<br/>regelmäßig erneuert"]
    SOURCE_EPHEMERAL["Ephemeres Handshake Key Pair<br/>S_EPH_PRIV / S_EPH_PUB<br/>nur für eine Verbindung"]
    SOURCE_APP -->|"Domain auflösen"| SOURCE_CHAIN
    SOURCE_CHAIN -->|"Domain → Z_ID_PUB<br/>mit Merkle Proof"| SOURCE_APP
end

STATE_ROOT -->|"finalisierter Header + Proof"| SOURCE_CHAIN
SESSION_COMMITMENT_SIGNATURE -->|"Commitment finalisieren"| SESSION_INDEX
SOURCE_SESSION --> SESSION_RECORD

subgraph LOCATE_REQUEST_SIGNATURE["Signaturhülle: SIGN mit S_SESSION_PRIV"]
    direction TB
    subgraph LOCATE_REQUEST_PACKET["LOCATE_REQUEST Packet"]
        direction TB
        LR_HEADER["Header<br/>version<br/>network_id<br/>message_type = LOCATE_REQUEST<br/>request_id = random 256 bit"]
        LR_IDENTITIES["Identitäten<br/>source_identity = S_ID_PUB<br/>source_session = S_SESSION_PUB<br/>target_identity = Z_ID_PUB"]
        LR_PROOF["Blockchain-Bindung<br/>Session Commitment<br/>Merkle Proof<br/>finalized_state_root_reference"]
        LR_ROUTING["Routing<br/>previous_hop_public_key<br/>hop_limit<br/>path_nonce<br/>keine Next-Hop-IP im Paket"]
        LR_TIME["Zeitbindung<br/>created_at<br/>expires_at<br/>maximal 1 bis 5 Minuten"]
        LR_ABUSE["Missbrauchsschutz<br/>monotonic_request_counter<br/>adaptive Proof of Work<br/>Größenlimit<br/>optionales Gebühren- oder Quoten-Credential"]
        subgraph LR_ENCRYPTED["Ende-zu-Ende verschlüsselter Request Payload"]
            direction TB
            LR_SECRET["S_EPH_PUB<br/>optionale Source Connection Candidates<br/>angeforderter Dienst<br/>Antwortschlüssel<br/>verschlüsselt für Z_ID_PUB oder Z_DISC_PUB"]
        end
        LR_HEADER --> LR_IDENTITIES --> LR_PROOF --> LR_ROUTING --> LR_TIME --> LR_ABUSE --> LR_ENCRYPTED
    end
    LR_SIGNATURE["signature_algorithm<br/>signature = SIGN S_SESSION_PRIV<br/>HASH canonical LOCATE_REQUEST Packet"]
    LOCATE_REQUEST_PACKET --> LR_SIGNATURE
end

SOURCE_APP -->|"erzeugt Request"| LOCATE_REQUEST_PACKET
SOURCE_EPHEMERAL --> LR_SECRET
SOURCE_SESSION --> LR_SIGNATURE
SOURCE_CHAIN --> LR_PROOF

subgraph ROUTING_OVERLAY["Dezentrales Public-Key-Routing-Overlay"]
    direction LR
    subgraph NODE_A["Overlay Node A"]
        A_CONN["verschlüsselte Peer-Kanäle"]
        A_VALIDATE["stufenweise Validierung"]
        A_CACHE["lokaler TTL-Cache"]
        A_ROUTE["Key-basierte Next-Hop-Auswahl"]
    end
    subgraph NODE_B["Overlay Node B"]
        B_CONN["verschlüsselte Peer-Kanäle"]
        B_VALIDATE["stufenweise Validierung"]
        B_CACHE["lokaler TTL-Cache"]
        B_ROUTE["Key-basierte Next-Hop-Auswahl"]
    end
    subgraph NODE_C["Overlay Node C"]
        C_CONN["verschlüsselte Peer-Kanäle"]
        C_VALIDATE["stufenweise Validierung"]
        C_CACHE["lokaler TTL-Cache"]
        C_ROUTE["Key-basierte Next-Hop-Auswahl"]
    end
    subgraph NODE_D["Overlay Node D"]
        D_CONN["verschlüsselte Peer-Kanäle"]
        D_VALIDATE["stufenweise Validierung"]
        D_CACHE["lokaler TTL-Cache"]
        D_ROUTE["Key-basierte Next-Hop-Auswahl"]
    end
    A_CONN <-->|"authentifizierter Node-Kanal"| B_CONN
    A_CONN <-->|"redundanter Pfad"| C_CONN
    B_CONN <-->|"authentifizierter Node-Kanal"| D_CONN
    C_CONN <-->|"authentifizierter Node-Kanal"| D_CONN
    A_VALIDATE --> A_CACHE --> A_ROUTE
    B_VALIDATE --> B_CACHE --> B_ROUTE
    C_VALIDATE --> C_CACHE --> C_ROUTE
    D_VALIDATE --> D_CACHE --> D_ROUTE
end

LR_SIGNATURE -->|"über bestehenden Node-Kanal"| A_CONN
A_ROUTE -->|"Pfad 1: Public-Key-Distanz sinkt"| B_CONN
A_ROUTE -->|"Pfad 2: unabhängige Redundanz"| C_CONN
B_ROUTE -->|"Pfad 1 fortsetzen"| D_CONN
C_ROUTE -->|"Pfad 2 fortsetzen"| D_CONN
STATE_ROOT -.->|"lokal validierbarer Merkle Proof"| A_VALIDATE
STATE_ROOT -.->|"lokal validierbarer Merkle Proof"| B_VALIDATE
STATE_ROOT -.->|"lokal validierbarer Merkle Proof"| C_VALIDATE
STATE_ROOT -.->|"lokal validierbarer Merkle Proof"| D_VALIDATE

subgraph VALIDATION["Jeder Overlay Node validiert lokal, unabhängig und ohne Vertrauen"]
    direction TB
    V1["1. Paketlänge, Version und Network ID prüfen<br/>konstante geringe Kosten"]
    V2["2. expires_at, hop_limit und Zeitfenster prüfen"]
    V3["3. Request-ID und Pfad-Nonce gegen TTL-Cache prüfen<br/>Replay oder Schleife → verwerfen"]
    V4["4. Finalized State Root prüfen<br/>keine unfinalisierten Branches akzeptieren"]
    V5["5. Merkle Proof prüfen<br/>Session Commitment existiert"]
    V6["6. Delegationskette prüfen<br/>ID_PUB → DISC_PUB → SESSION_PUB"]
    V7["7. Session-Signatur über gesamtes Paket prüfen"]
    V8["8. Rate Limit pro Identity, Session, Ziel und Epoch"]
    V9["9. Adaptives Proof of Work prüfen<br/>Schwierigkeit steigt mit Last"]
    V10["10. Nur gültige Anfrage cachen und weiterleiten<br/>keine dauerhafte Chain-Speicherung"]
    DROP["Sofort verwerfen<br/>kein teurer Handshake<br/>kein Antwortzustand<br/>optional lokale Fehlbewertung"]
    V1 --> V2 --> V3 --> V4 --> V5 --> V6 --> V7 --> V8 --> V9 --> V10
    V1 -.->|"ungültig"| DROP
    V2 -.->|"abgelaufen"| DROP
    V3 -.->|"Replay"| DROP
    V4 -.->|"nicht finalisiert"| DROP
    V5 -.->|"Proof ungültig"| DROP
    V6 -.->|"nicht autorisiert"| DROP
    V7 -.->|"manipuliert"| DROP
    V8 -.->|"Limit überschritten"| DROP
    V9 -.->|"Arbeitsnachweis unzureichend"| DROP
end

A_VALIDATE --> V1
B_VALIDATE --> V1
C_VALIDATE --> V1
D_VALIDATE --> V1
V10 -->|"Cache: request_id, previous peer, expiry"| SOURCE_CACHE

subgraph TARGET_NODE["Node Z: Ziel, Client, Server und Protokollteilnehmer"]
    direction TB
    TARGET_CHAIN["Lokaler validierter Blockchain-State"]
    TARGET_ROUTING["Routingtabelle und verschlüsselte Peer-Kanäle"]
    TARGET_REQUEST_CACHE["Flüchtiger Request-Cache<br/>Request-ID bis Ablauf"]
    TARGET_SERVICE["Lokaler Dienst<br/>DNS, Web, API oder anderer Service"]
    TARGET_ENDPOINT["Endpoint Manager<br/>lokale Interfaces<br/>NAT-Zustand<br/>direkte Candidates<br/>Relay-Circuits"]
    TARGET_SESSION["Kurzlebiges Session Key Pair<br/>Z_SESSION_PRIV / Z_SESSION_PUB"]
    TARGET_EPHEMERAL["Neues ephemeres Handshake Key Pair<br/>Z_EPH_PRIV / Z_EPH_PUB"]
    TARGET_CHAIN --> TARGET_REQUEST_CACHE
    TARGET_ENDPOINT --> TARGET_SERVICE
end

D_ROUTE -->|"Request anhand Z_ID_PUB zustellen"| TARGET_ROUTING
TARGET_ROUTING --> TARGET_REQUEST_CACHE
STATE_ROOT --> TARGET_CHAIN
TARGET_SESSION --> SESSION_RECORD

subgraph LOCATE_RESPONSE_SIGNATURE["Signaturhülle: SIGN mit Z_SESSION_PRIV"]
    direction TB
    subgraph LOCATE_RESPONSE_PACKET["LOCATE_RESPONSE Packet"]
        direction TB
        LRES_HEADER["Header<br/>version<br/>network_id<br/>message_type = LOCATE_RESPONSE<br/>original_request_id"]
        LRES_IDENTITIES["Identitäten<br/>source_identity = Z_ID_PUB<br/>source_session = Z_SESSION_PUB<br/>target_identity = S_ID_PUB"]
        LRES_PROOF["Blockchain-Bindung<br/>Z Session Commitment<br/>Merkle Proof<br/>finalized_state_root_reference"]
        LRES_TIME["Zeitbindung<br/>created_at<br/>expires_at<br/>sequence_number"]
        subgraph RESPONSE_ENCRYPTED["Ende-zu-Ende verschlüsselt für S_EPH_PUB"]
            direction TB
            RESPONSE_SECRET["Z_EPH_PUB<br/>direkte IPv4 oder IPv6 Candidates<br/>Ports und Transportarten<br/>NAT Traversal Token<br/>Relay Public Keys<br/>kurzlebige Relay Circuit Tokens<br/>Server Capabilities"]
        end
        LRES_HEADER --> LRES_IDENTITIES --> LRES_PROOF --> LRES_TIME --> RESPONSE_ENCRYPTED
    end
    LRES_SIGNATURE["signature = SIGN Z_SESSION_PRIV<br/>HASH canonical LOCATE_RESPONSE Packet"]
    LOCATE_RESPONSE_PACKET --> LRES_SIGNATURE
end

TARGET_REQUEST_CACHE -->|"Request vollständig validiert"| LOCATE_RESPONSE_PACKET
TARGET_ENDPOINT --> RESPONSE_SECRET
TARGET_EPHEMERAL --> RESPONSE_SECRET
TARGET_SESSION --> LRES_SIGNATURE
TARGET_CHAIN --> LRES_PROOF

LRES_SIGNATURE -->|"Request-ID bestimmt Reverse Route"| D_CONN
D_CONN -->|"bestehende Verbindung"| B_CONN
D_CONN -->|"redundanter Rückweg"| C_CONN
B_CONN --> A_CONN
C_CONN --> A_CONN
A_CONN -->|"Antwort an Source"| SOURCE_CACHE
SOURCE_CACHE -->|"Request-ID, TTL, Z-Signatur und Proof prüfen"| SOURCE_APP
SOURCE_EPHEMERAL -->|"S_EPH_PRIV entschlüsselt"| RESPONSE_ENCRYPTED

subgraph CONNECTION["Authentifizierter Ende-zu-Ende-Verbindungsaufbau"]
    direction TB
    DIRECT["1. Direkte Candidate-Verbindung versuchen"]
    NAT["2. Falls nötig NAT Traversal<br/>kurzlebiges signiertes Token"]
    RELAY["3. Falls direkt unmöglich:<br/>austauschbare Relay Nodes<br/>Relay entschlüsselt keine Nutzdaten"]
    HANDSHAKE_INIT["Handshake Initial<br/>S_EPH_PUB<br/>Z_EPH_PUB Referenz<br/>Request-ID<br/>stateless Cookie Anfrage"]
    COOKIE["Z erzeugt stateless Cookie<br/>MAC mit lokal rotierendem Cookie Secret<br/>vor Rückgabe kein Session State"]
    COOKIE_RETURN["S sendet Cookie zurück<br/>beweist erreichbaren Rückweg"]
    DH["Ephemerer Diffie-Hellman-Austausch<br/>gemeinsames Geheimnis aus S_EPH und Z_EPH"]
    TRANSCRIPT["Handshake Transcript Hash<br/>Identitäten, Ephemeral Keys, Network ID,<br/>Request-ID und Algorithmen"]
    SOURCE_AUTH["Source Authentication<br/>SIGN mit S_TRANS_PRIV über Transcript Hash"]
    TARGET_AUTH["Target Authentication<br/>SIGN mit Z_TRANS_PRIV über Transcript Hash"]
    SESSION_DERIVE["Session Key Derivation<br/>HKDF shared_secret + transcript_hash<br/>getrennte Richtungs- und Rekey-Schlüssel"]
    SECURE_CHANNEL["Ende-zu-Ende AEAD-Kanal<br/>Vertraulichkeit, Integrität, Authentizität<br/>Sequenznummern und Forward Secrecy"]
    DIRECT --> HANDSHAKE_INIT
    DIRECT -.->|"nicht erreichbar"| NAT
    NAT --> HANDSHAKE_INIT
    NAT -.->|"fehlgeschlagen"| RELAY
    RELAY --> HANDSHAKE_INIT
    HANDSHAKE_INIT --> COOKIE --> COOKIE_RETURN --> DH --> TRANSCRIPT
    TRANSCRIPT --> SOURCE_AUTH --> SESSION_DERIVE
    TRANSCRIPT --> TARGET_AUTH --> SESSION_DERIVE
    SESSION_DERIVE --> SECURE_CHANNEL
end

SOURCE_APP --> DIRECT
TARGET_ENDPOINT --> DIRECT
RESPONSE_SECRET --> DIRECT
TRANSPORT_KEY --> SOURCE_AUTH
TRANSPORT_KEY --> TARGET_AUTH
TARGET_EPHEMERAL --> DH
SOURCE_EPHEMERAL --> DH
SECURE_CHANNEL -->|"verschlüsselte Protokolldaten"| TARGET_SERVICE

subgraph DATA_LIFECYCLE["Datenlebenszyklus"]
    direction LR
    PERMANENT["Dauerhaft oder bis State-Pruning<br/>Domain Ownership, Delegationen,<br/>Rotationen, Ablaufzustände, State Roots"]
    SHORT_CHAIN["Kurzlebiger Blockchain-State<br/>Session Commitments<br/>nach Ablauf aus aktivem State entfernbar"]
    TEMPORARY["Nur lokaler TTL-Cache<br/>Locate Requests, Responses,<br/>Request IDs und Reverse Routes"]
    CONNECTION_STATE["Nur während Verbindung<br/>ephemere Private Keys, Cookies,<br/>Session Keys und Sequenznummern"]
    DELETION["Sichere lokale Löschung<br/>nach TTL, Verbindungsende oder Rekey<br/>keine flüchtigen Pakete in Blöcken"]
    SHORT_CHAIN --> DELETION
    TEMPORARY --> DELETION
    CONNECTION_STATE --> DELETION
end

STATE --> PERMANENT
SESSION_INDEX --> SHORT_CHAIN
SOURCE_CACHE --> TEMPORARY
TARGET_REQUEST_CACHE --> TEMPORARY
SECURE_CHANNEL --> CONNECTION_STATE

subgraph DOMAIN_LIFECYCLE["Domain- und Identitätslebenszyklus"]
    direction LR
    AVAILABLE["AVAILABLE<br/>frei registrierbar"]
    ACTIVE["ACTIVE<br/>Owner Key gültig<br/>maximal ein Jahr"]
    GRACE["GRACE<br/>optional<br/>nur bisheriger Owner darf verlängern"]
    EXPIRED["EXPIRED<br/>Besitzbindung entfernt"]
    SWAP["KEY SWAP<br/>alter und neuer Identity Key signieren<br/>Domain bleibt registriert"]
    AVAILABLE -->|"Registrierung finalisiert"| ACTIVE
    ACTIVE -->|"Renewal finalisiert"| ACTIVE
    ACTIVE -->|"expires_at erreicht"| GRACE
    GRACE -->|"gültige Renewal"| ACTIVE
    GRACE -->|"Grace Period endet"| EXPIRED
    EXPIRED --> AVAILABLE
    ACTIVE -->|"atomare Rotation finalisiert"| SWAP
    SWAP --> ACTIVE
end

EXPIRY_INDEX --> DOMAIN_LIFECYCLE
ROTATION_SIGNATURE --> SWAP

subgraph SECURITY_RESULT["Erreichte Protokolleigenschaften"]
    direction TB
    SEC1["Manipulationsschutz<br/>kanonische Kodierung und Signaturen"]
    SEC2["Keine zentrale Discovery-Autorität<br/>mehrere unabhängige key-basierte Pfade"]
    SEC3["Blockchain-Zwang<br/>ohne finalisierten Session Proof keine akzeptierte Anfrage"]
    SEC4["IP-Abstraktion<br/>Anwendung adressiert Public Keys<br/>Locator nur für Kommunikationspartner verschlüsselt"]
    SEC5["Fork-Verhalten<br/>nur finalisierte State Roots<br/>ohne Quorum keine Registry-Schreiboperationen"]
    SEC6["DDoS-Begrenzung, nicht absolute Unmöglichkeit<br/>frühe Billigprüfungen, TTL, Replay Cache,<br/>Identitätslimits, adaptives Proof of Work,<br/>stateless Cookies, keine Request-Persistenz"]
    SEC7["Bösartige Nodes können verwerfen,<br/>aber keine gültigen Identitäten,<br/>Antworten oder Nutzdaten unbemerkt fälschen"]
    SEC8["Verfügbarkeit<br/>redundante Pfade und Relay Nodes<br/>keine zentrale Abhängigkeit"]
end

VALIDATION --> SECURITY_RESULT
BLOCKCHAIN --> SECURITY_RESULT
ROUTING_OVERLAY --> SECURITY_RESULT
CONNECTION --> SECURITY_RESULT
```
