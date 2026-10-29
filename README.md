# Dezentrale Netzwerkarchitektur

```info
Projektname: Public Key Network
Kurzname:    PKN
Repo:        public-key-network
Ziel-TLD:    .pkn
```

```mermaid
flowchart TB
    subgraph ID["1. Lokale Identität und Schlüsselhierarchie"]
        SEED["Master Seed<br/>softwaregeneriert, >=128 Bit Entropie<br/>menschenlesbare Kodierung + Prüfsumme<br/>verlässt das Gerät nie"]
        KDF["Versionierte KDF mit Domain Separation<br/>Eingabe: Seed + network_id + Zweck + Version"]
        IK["Identity Key Pair<br/>dauerhafte Identität<br/>signiert nur Delegation, Widerruf, Rotation"]
        RK["Registry Key Pair<br/>Domain registrieren, verlängern, übertragen"]
        DK["Discovery Key Pair<br/>Session- und Rendezvous-Schlüssel delegieren"]
        TK["Transport Key Pair<br/>Handshake-Transkript authentifizieren"]
        REK["Recovery Key Pair, optional offline<br/>Notfallrotation nach festgelegter Policy"]
        SEED --> KDF
        KDF -->|"identity"| IK
        KDF -->|"registry"| RK
        KDF -->|"discovery"| DK
        KDF -->|"transport"| TK
        KDF -->|"recovery"| REK
        subgraph DEL["Signaturhülle: Identity-Delegation"]
            DELDATA["identity_pub, registry_pub, discovery_pub,<br/>transport_pub, recovery_policy,<br/>key_version, valid_from, valid_until"]
            DELSIG["SIGN(ID_PRIV, HASH(canonical(DELDATA)))"]
            DELDATA -->|"signiert"| DELSIG
        end
        IK --> DELSIG
        RK --> DELDATA
        DK --> DELDATA
        TK --> DELDATA
        REK --> DELDATA
    end

    subgraph CHAIN["2. Dauerhafte dezentrale Registry-Blockchain"]
        TXPOOL["Validierte Registry-Transaktionen<br/>keine IPs, Locate-Pakete oder Nutzdaten"]
        BFT["BFT-Konsens mit harter Finalität<br/>gewichtetes Validator-Quorum >= 2/3<br/>ohne Quorum keine neue Finalität"]
        FB["Finalisierter Block<br/>network_id, height, previous_hash, epoch,<br/>tx_root, state_root, validator_set_hash,<br/>aggregierter Finalitätsnachweis"]
        STATE["Deterministischer Registry State<br/>snapshot- und pruningfähig"]
        TXPOOL --> BFT --> FB --> STATE

        subgraph DTX["Signaturhülle: Domain-Operation"]
            DDATA["REGISTER | RENEW | TRANSFER | RELEASE<br/>domain, owner_identity_pub, registry_pub,<br/>sequence, valid_until, operation_data"]
            DSIG["SIGN(REG_PRIV, HASH(canonical(DDATA)))"]
            DDATA -->|"signiert"| DSIG
        end
        subgraph ROT["Doppelte Signaturhülle: Key-Swap"]
            ROTDATA["old_identity_pub, new_identity_pub,<br/>domain_set_root, new_delegation_hash,<br/>sequence, effective_height"]
            OSIG["SIGN(OLD_ID_PRIV, HASH(ROTDATA))"]
            NSIG["SIGN(NEW_ID_PRIV, HASH(ROTDATA))"]
            ROTDATA -->|"Übertragung autorisiert"| OSIG
            ROTDATA -->|"Übernahme bestätigt"| NSIG
        end
        subgraph VALTX["Validator- und Protokolloperationen"]
            VDATA["Validator-Beitritt/Austritt, Stake/Arbeitsnachweis,<br/>Slashing-Beweis, Parameter- und Versionsupdate"]
        end
        DSIG --> TXPOOL
        OSIG --> TXPOOL
        NSIG --> TXPOOL
        DELSIG --> TXPOOL
        VDATA --> TXPOOL

        STATE --> DOMIDX["Domain Index<br/>normalisierte .pkn-Domain -> Identity Public Key"]
        STATE --> DELIDX["Delegation/Revocation Index"]
        STATE --> VALIDX["Validator Set und Konsensparameter"]
        STATE --> PROTOIDX["Aktive Protokoll- und Algorithmusversionen"]
        STATE --> ROOT["Finalisierter State Root<br/>Merkle Proofs für Compact Nodes"]
    end

    subgraph DOMAIN["3. Domain-Lebenszyklus"]
        FREE["AVAILABLE"]
        ACTIVE["ACTIVE<br/>Registrierung für 1 Jahr"]
        GRACE["GRACE<br/>30 Tage, nur bisheriger Owner"]
        RELEASED["FREIGEGEBEN"]
        TRANSFER["ATOMARER TRANSFER / KEY-SWAP"]
        FREE -->|"REGISTER finalisiert"| ACTIVE
        ACTIVE -->|"RENEW vor Ablauf"| ACTIVE
        ACTIVE -->|"Jahresfrist endet"| GRACE
        GRACE -->|"RENEW"| ACTIVE
        GRACE -->|"30 Tage enden"| RELEASED
        RELEASED --> FREE
        ACTIVE -->|"alte + neue Signatur"| TRANSFER --> ACTIVE
    end
    DOMIDX --> DOMAIN

    subgraph BOOT["4. Dezentraler Bootstrap und Peer-Authentifizierung"]
        SOURCES["Mehrere unabhängige Startquellen<br/>mitgelieferte Public Keys, manuelle Peers,<br/>optionales DNS nur als unvertrauenswürdiger Locator"]
        HELLO["PEER_HELLO<br/>network_id, version, identity_pub,<br/>ephemeral_pub, capabilities, nonce"]
        CHALLENGE["PEER_CHALLENGE<br/>gegenseitige Nonces + stateless Cookie"]
        AUTH["PEER_AUTH<br/>Delegation + State-Proof + signiertes Transcript"]
        LINK["Authentifizierter verschlüsselter Peer-Kanal<br/>AEAD, Sequenznummern, Rekey, Keepalive"]
        SOURCES --> HELLO --> CHALLENGE --> AUTH --> LINK
        ROOT -->|"Proof prüfen"| AUTH
    end

    subgraph NODE["5. Identischer Node-Kern auf jedem Teilnehmer"]
        CHAINMODE["Chain Engine<br/>Full, Pruned oder Compact<br/>gleiche Validierungsregeln"]
        ROUTETABLE["XOR-Keyspace-Routingtabelle<br/>mehrere Peers je Bucket und Netzpräfix<br/>laufender Refresh und Ersatz"]
        CACHE["Begrenzter flüchtiger TTL-Cache<br/>Request-IDs, Leases, Reverse Routes,<br/>Rate Limits und Fehlbewertungen"]
        RESOLVER["Lokaler .pkn-Resolver und API<br/>Browser-Plug-in / Anwendungen"]
        CAP["Signierte Capabilities<br/>validate, route, store, relay, host, propose"]
        LINK --> ROUTETABLE
        ROOT --> CHAINMODE
        CHAINMODE --> RESOLVER
        CAP --> ROUTETABLE
    end

    subgraph LEASE["6. Flüchtige dezentrale Rendezvous-Ebene"]
        SESSION["Kurzlebiges Session Key Pair<br/>durch DISC_PRIV signiert, nicht on-chain"]
        subgraph LEASEBOX["Signaturhülle: RENDEZVOUS_LEASE"]
            LDATA["target_identity_pub, session_pub, epoch,<br/>sequence, relative_ttl, route_token,<br/>capabilities, encrypted_contact_blob"]
            LSIG["SIGN(DISC_PRIV, HASH(canonical(LDATA)))"]
            LDATA -->|"signiert"| LSIG
        end
        K1["zuständiger XOR-Keyspace-Node A<br/>nur TTL-Cache"]
        K2["zuständiger XOR-Keyspace-Node B<br/>nur TTL-Cache"]
        K3["zuständiger XOR-Keyspace-Node C<br/>nur TTL-Cache"]
        SESSION --> LDATA
        DK --> LSIG
        LSIG -->|"repliziert"| K1
        LSIG -->|"repliziert"| K2
        LSIG -->|"repliziert"| K3
        K1 -->|"Ablauf: logisch löschen"| CACHE
        K2 -->|"Ablauf: logisch löschen"| CACHE
        K3 -->|"Ablauf: logisch löschen"| CACHE
    end

    subgraph REQUEST["7. Flüchtiger Locate Request"]
        subgraph REQBOX["Äußere Signaturhülle: S_SESSION_PRIV"]
            RHEAD["version, network_id, LOCATE_REQUEST,<br/>request_id 256 Bit, source_identity_pub,<br/>source_session_pub, target_identity_pub"]
            RPROOF["Identity-Delegation + finalisierter State-Proof"]
            RROUTE["previous_hop_pub, hop_limit,<br/>path_nonce, finalized_height, relative_ttl"]
            RABUSE["monotonic_counter, Größenklasse,<br/>adaptiver Proof of Work, Quotenbeleg"]
            subgraph RENC["Ende-zu-Ende verschlüsselt für Ziel"]
                RPAY["S_EPH_PUB, Dienst, Antwortschlüssel,<br/>optionale Source Candidates und Padding"]
            end
            RSIG["SIGN(S_SESSION_PRIV,<br/>HASH(canonical(gesamtes Request-Paket)))"]
            RHEAD --> RPROOF --> RROUTE --> RABUSE --> RENC -->|"signiert"| RSIG
        end
    end
    RESOLVER -->|"Domain -> Z_ID_PUB + Merkle Proof"| DOMIDX
    RESOLVER --> REQUEST

    subgraph VALIDATE["8. Lokale Validierung an jedem Hop"]
        C1["1 Format, Version, network_id, Paketgröße"]
        C2["2 finalized_height, TTL, Hop Limit"]
        C3["3 Request-ID und Path-Nonce gegen Replay-Cache"]
        C4["4 State-Proof und Delegationskette"]
        C5["5 Session-Signatur über kanonisches Paket"]
        C6["6 Limits pro Identity, Session, Ziel und Epoch"]
        C7["7 adaptiven Arbeitsnachweis prüfen"]
        C8["8 lokal cachen und über 3-5 unabhängige XOR-Pfade leiten"]
        DROP["Fail closed<br/>sofort verwerfen, keinen teuren Zustand anlegen"]
        C1 --> C2 --> C3 --> C4 --> C5 --> C6 --> C7 --> C8
        C1 -.-> DROP
        C2 -.-> DROP
        C3 -.-> DROP
        C4 -.-> DROP
        C5 -.-> DROP
        C6 -.-> DROP
        C7 -.-> DROP
    end
    RSIG --> VALIDATE
    ROOT --> C4
    ROUTETABLE --> C8
    C8 -->|"Lease-Suche"| K1
    C8 -->|"redundant"| K2
    C8 -->|"redundant"| K3

    subgraph RESPONSE["9. Flüchtige Locate Response"]
        subgraph RESPBOX["Äußere Signaturhülle: Z_SESSION_PRIV"]
            PHEAD["LOCATE_RESPONSE, original_request_id,<br/>Z_ID_PUB, Z_SESSION_PUB, S_ID_PUB,<br/>finalized_height, relative_ttl, sequence"]
            PPROOF["Z-Delegation + finalisierter State-Proof"]
            subgraph PENC["Verschlüsselt ausschließlich für S_EPH_PUB"]
                PPAY["Z_EPH_PUB, direkte IPv4/IPv6 Candidates,<br/>Ports, QUIC bevorzugt, TCP-Fallback,<br/>NAT-Token, Relay-Keys, Circuit-Tokens, Padding"]
            end
            PSIG["SIGN(Z_SESSION_PRIV,<br/>HASH(canonical(gesamtes Response-Paket)))"]
            PHEAD --> PPROOF --> PENC -->|"signiert"| PSIG
        end
    end
    K1 -->|"gültiger Rückkanal zum Ziel"| RESPONSE
    K2 --> RESPONSE
    K3 --> RESPONSE
    PSIG -->|"Reverse Route über Request-ID"| CACHE
    CACHE --> RESOLVER

    subgraph CONNECT["10. Manipulationssicherer Verbindungsaufbau"]
        DIRECT["Direkten QUIC/UDP-Candidate versuchen"]
        NAT["NAT Traversal mit kurzlebigem signiertem Token"]
        RELAY["Mehrere austauschbare Relay-Circuits<br/>nur Ciphertext, kein Klartextzugriff"]
        COOKIE["Stateless Cookie vor teurem Zielzustand"]
        ECDH["Ephemerer Diffie-Hellman-Austausch<br/>S_EPH x Z_EPH"]
        TRANSCRIPT["Transcript Hash<br/>Identitäten, Ephemeral Keys, Request-ID,<br/>network_id, Version und Algorithmen"]
        SAUTH["SIGN(S_TRANS_PRIV, transcript_hash)"]
        ZAUTH["SIGN(Z_TRANS_PRIV, transcript_hash)"]
        HKDF["HKDF(shared_secret, transcript_hash)<br/>getrennte S->Z-, Z->S- und Rekey-Keys"]
        AEAD["Ende-zu-Ende AEAD-Kanal<br/>Sequenznummern, Replay-Schutz,<br/>Forward Secrecy und regelmäßiges Rekeying"]
        DIRECT --> COOKIE
        DIRECT -.->|"nicht erreichbar"| NAT --> COOKIE
        NAT -.->|"fehlgeschlagen"| RELAY --> COOKIE
        COOKIE --> ECDH --> TRANSCRIPT
        TRANSCRIPT --> SAUTH --> HKDF
        TRANSCRIPT --> ZAUTH --> HKDF
        HKDF --> AEAD
    end
    PPAY --> CONNECT
    TK --> SAUTH
    TK --> ZAUTH

    subgraph DATA["11. Datenhaltung und Löschung"]
        PERM["Dauerhaft: finalisierte Registry,<br/>Delegationen, Widerrufe, Rotationen, State Roots"]
        PRUNE["Prunable: alte Blöcke und abgelaufene Zustände<br/>nach finalisierten Snapshot-Regeln"]
        TEMP["Flüchtig: Leases, Requests, Responses,<br/>Reverse Routes und Rate-Limit-Zähler"]
        CONN["Nur Sitzung: ephemere Private Keys,<br/>Cookies, Verkehrsschlüssel und Nonces"]
        ERASE["Ablauf: aus aktivem Speicher entfernen,<br/>flüchtige Schlüssel verwerfen,<br/>keine Protokollhistorie erzeugen"]
        PERM --> PRUNE
        TEMP --> ERASE
        CONN --> ERASE
    end
    STATE --> PERM
    LEASE --> TEMP
    REQUEST --> TEMP
    RESPONSE --> TEMP
    AEAD --> CONN

    subgraph SECURITY["12. Sicherheitsgrenzen"]
        S1["Kein zentraler Index-, Discovery-, Bootstrap- oder Relay-Node ist autoritativ"]
        S2["Bösartige Hops können verwerfen oder verzögern,<br/>aber signierte Identitäten und AEAD-Daten nicht unbemerkt fälschen"]
        S3["DDoS wird begrenzt, nicht mathematisch ausgeschlossen:<br/>Billigprüfung, harte Limits, PoW, Cookies, Redundanz"]
        S4["Netzpartition: Discovery und bestehende Sitzungen dürfen weiterlaufen;<br/>Registry-Schreibvorgänge stoppen ohne Finalitätsquorum"]
        S5["DNS darf nur unvertrauenswürdige Bootstrap-Locators liefern;<br/>niemals Identität oder .pkn-Besitz bestimmen"]
    end
    VALIDATE --> SECURITY
    BFT --> SECURITY
    CONNECT --> SECURITY
```

```regeln
- PKN bildet genau ein globales logisches Netzwerk mit einer fest definierten network_id.
- Pakete, Blöcke, Proofs und Zustände einer anderen network_id werden abgelehnt.
- Die Netzwerkidentität eines Teilnehmers ist ausschließlich sein dauerhafter Identity Public Key.
- Eine IP-Adresse ist ausschließlich ein kurzlebiger Transport-Locator und niemals eine Identität.
- Jeder Node verwendet denselben Protokollkern und dieselben Validierungsregeln.
- Jeder Node muss Registry-Proofs validieren, signierte Pakete prüfen, gültige Pakete weiterleiten und einen begrenzten Discovery-Cache bereitstellen.
- Zusätzliche Rollen wie Blockvorschlag, Relay oder Hosting werden als signierte Capabilities veröffentlicht.
- Full, Pruned und Compact Nodes unterscheiden sich nur in Datenhaltung und Leistung, nicht in Sicherheitsregeln.
- Der Master Seed wird lokal durch einen kryptografisch sicheren Zufallszahlengenerator erzeugt.
- Der Master Seed besitzt mindestens 128 Bit Entropie, eine menschenlesbare Kodierung und eine Prüfsumme.
- Der Master Seed wird niemals übertragen, veröffentlicht oder in der Blockchain gespeichert.
- Derselbe Seed erzeugt mit derselben network_id, Key-Version und demselben Zweck deterministisch dieselben Schlüssel.
- Identity-, Registry-, Discovery-, Transport- und Recovery-Schlüssel werden mit getrennten KDF-Kontexten abgeleitet.
- Ein Schlüssel darf ausschließlich für seinen festgelegten Zweck verwendet werden.
- Der Identity Private Key signiert nur Delegationen, Widerrufe, Recovery-Policies und Identitätsrotationen.
- Registry-, Discovery- und Transport-Schlüssel sind nur über eine gültige Identity-Delegation autorisiert.
- Jede Delegation enthält Version, Gültigkeitsbeginn und Gültigkeitsende.
- Unbekannte oder widerrufene Schlüsselversionen werden abgelehnt.
- Private Schlüssel verbleiben lokal und werden nach Möglichkeit durch Betriebssystem-KeyStore oder Hardware-Schutz gesichert.
- Eine Wiederherstellung aus dem Master Seed erfolgt vollständig lokal.
- Ein verlorener Seed ohne eingerichteten Recovery Key führt zum Verlust der Identität.
- Ein Recovery Key ist optional, getrennt und vorzugsweise offline aufzubewahren.
- Ein Key-Swap benötigt die Signatur der alten und der neuen Identity.
- Ein Key-Swap überträgt Domains atomar und ohne zwischenzeitliche Freigabe.
- Nach finalisiertem Key-Swap werden alte Schlüssel für neue Operationen abgelehnt.
- Die Ziel-TLD des Netzwerks ist .pkn.
- Domainnamen werden vor Verarbeitung deterministisch normalisiert.
- Domainlabels verwenden Kleinbuchstaben a-z, Ziffern 0-9 und Bindestriche.
- Ein Label darf nicht mit einem Bindestrich beginnen oder enden.
- Unicode-Namen werden nur über eine eindeutig festgelegte IDNA-Version verarbeitet.
- Reservierte Protokoll-, System- und Missbrauchsnamen dürfen nicht registriert werden.
- Nur der finalisierte Registry State bestimmt den Besitzer einer .pkn-Domain.
- Eine Domainregistrierung gilt für genau ein Jahr ab finalisiertem Registrierungszeitpunkt.
- Nach Ablauf beginnt eine Grace Period von 30 Tagen.
- Während der Grace Period darf nur der bisherige Besitzer verlängern.
- Nach Ende der Grace Period wird die Domain wieder frei registrierbar.
- Subdomains werden vom Domainbesitzer off-chain signiert verwaltet und benötigen keine eigene Blockchain-Transaktion.
- Eine Domainoperation enthält eine monotone Sequenznummer.
- Eine Domainoperation ist nur gültig, wenn sie auf dem aktuell finalisierten Vorgängerzustand aufbaut.
- Für eine Domain kann in einem finalisierten State nur genau ein Besitzer existieren.
- Registrierung, Verlängerung, Transfer und Freigabe sind getrennte Operationstypen.
- Massenregistrierungen werden durch protocolweit definierte Kosten, Quoten oder Arbeitsnachweise begrenzt.
- Dauerhafte Registry-Transaktionen enthalten niemals IP-Adressen, Locate-Pakete oder Nutzdaten.
- Der Konsens verwendet harte Finalität.
- Ein Registry-Zustand gilt erst bei einem gültigen Finalitätsnachweis von mindestens zwei Dritteln des aktiven Validatorgewichts.
- Ohne Finalitätsquorum werden keine neuen Registry-Zustände finalisiert.
- Bei einer Netzpartition dürfen Discovery und bestehende Verbindungen weiterarbeiten, Registry-Schreibvorgänge jedoch nicht ohne Quorum finalisieren.
- Validator-Beitritt, Validator-Austritt und Validatorgewicht werden durch finalisierte Regeln bestimmt.
- Nachweisbares Doppelvoting oder widersprüchliche Finalisierung wird durch einen finalisierten Ausschluss- oder Slashing-Beweis behandelt.
- Blockformat, Blockintervall, Maximalgröße und Validatorgrenzen sind versionierte Konsensparameter.
- Genesis Block, initiale network_id und initiales Validator Set sind Bestandteil der Softwaredistribution.
- Protokollupgrades werden erst ab einer vorab finalisierten Aktivierungshöhe gültig.
- Jeder finalisierte Block enthält einen deterministischen State Root.
- Compact Nodes validieren Header, Finalitätsnachweise und Merkle Proofs selbst.
- Snapshots sind nur gültig, wenn ihr State Root durch einen finalisierten Block bestätigt wird.
- Alte Blöcke dürfen nach finalisierten Snapshot- und Pruningregeln entfernt werden.
- Discovery-Daten werden nicht dauerhaft in der Blockchain gespeichert.
- Ein Online-Node veröffentlicht kurzlebige signierte Rendezvous-Leases.
- Ein Rendezvous Lease enthält keine unverschlüsselte öffentliche Ziel-IP.
- Rendezvous-Leases werden bei mehreren für den Ziel-Key zuständigen XOR-Keyspace-Nodes repliziert.
- Zuständigkeit im Routing-Keyspace wird aus einem standardisierten Hash des Identity Public Key berechnet.
- Ein einzelner Rendezvous-Node ist niemals autoritativ.
- Ein Rendezvous Lease besitzt eine kurze, protocolweit begrenzte TTL.
- Der Ziel-Node erneuert seinen Lease vor Ablauf.
- Abgelaufene Leases werden nicht mehr verwendet und aus dem aktiven Cache entfernt.
- Lease-Sequenznummern verhindern Rücksetzungen auf ältere gültige Leases.
- Ein Locator wird nur akzeptiert, wenn er durch das erwartete Ziel oder einen gültig delegierten Session Key autorisiert ist.
- Direkte IP-Adressen und Ports werden nur Ende-zu-Ende verschlüsselt an den konkreten Kommunikationspartner übertragen.
- Jeder Node hält nur einen begrenzten Ausschnitt der Routing- und Rendezvous-Informationen.
- Das Overlay verwendet eine deterministische XOR-Distanz über gehashte Node-Identitäten.
- Routingtabellen halten mehrere unabhängige Peers je Distanzbereich.
- Peers werden über unterschiedliche Netzpräfixe und unabhängige Pfade verteilt, soweit verfügbar.
- Locate Requests werden parallel über drei bis fünf unabhängige key-basierte Pfade geleitet.
- Jedes weitergeleitete Paket besitzt ein Hop Limit.
- Request-ID und Path Nonce verhindern Schleifen und mehrfache Verarbeitung.
- Antworten verwenden die kurzzeitig gecachte Reverse Route der Request-ID.
- Fällt ein Pfad aus, wird ohne Sicherheitsherabstufung ein unabhängiger Ersatzpfad verwendet.
- Routingtabellen werden regelmäßig aktualisiert und ausgefallene Peers ersetzt.
- Ein Node akzeptiert Routinginformationen nie als Identitätsnachweis.
- Alle Overlay-Peer-Verbindungen werden gegenseitig authentifiziert und verschlüsselt.
- Bootstrap-Nodes sind nur Startpunkte und keine Autorität.
- Die Software enthält mehrere unabhängige Bootstrap-Public-Keys und unterstützt manuell importierte Peers.
- Konventionelles DNS darf optional Bootstrap-Locators liefern, aber niemals PKN-Identität oder Domainbesitz bestimmen.
- Ein Node startet sicherheitsrelevante Operationen erst nach Kontakt zu mehreren voneinander unabhängigen Peers.
- Jeder neue Peer-Kanal verwendet gegenseitige Challenges, Nonces und ein signiertes Handshake-Transkript.
- Jeder Pakettyp besitzt eine eindeutige Typkennung und Protokollversion.
- Alle signierten Daten werden vor dem Hashen kanonisch und deterministisch serialisiert.
- Alle sicherheitsrelevanten Felder eines Pakets liegen innerhalb der Signaturabdeckung.
- Unbekannte Pflichtfelder oder inkompatible Pflichtversionen führen zur Ablehnung.
- Paketgrößen sind je Pakettyp hart begrenzt.
- Fragmentierung ist nur nach authentifiziertem Kanalaufbau erlaubt.
- Vor teuren kryptografischen Prüfungen werden Version, network_id, Länge, TTL und Hop Limit geprüft.
- Abgelaufene Pakete werden verworfen.
- Bereits verarbeitete Request-IDs werden innerhalb ihrer TTL verworfen.
- Ungültige Signaturen, Proofs, Delegationen oder Sequenzen führen zum Fail-closed-Abbruch.
- Discovery-Anfragen werden nur akzeptiert, wenn die Identity-Delegation gegen einen finalisierten State Root prüfbar ist.
- Kurzlebige Session Keys werden off-chain durch den gültigen Discovery Key signiert.
- Session Keys benötigen keine eigene Blockchain-Transaktion.
- Jeder Node prüft State-Proofs, Delegationsketten, Signaturen und Gültigkeit selbst.
- Kein Weiterleitungsnode wird für die Echtheit fremder Daten als vertrauenswürdig behandelt.
- Source- und Target-Identität bleiben in der Basisversion für Routingnodes sichtbar; Locator und Nutzdaten bleiben verschlüsselt.
- Locate Request und Locate Response erhalten eine relative TTL, eine finalisierte Referenzhöhe und optional eine begrenzte Wall-Clock-Toleranz.
- Die Gültigkeit sicherheitsrelevanter Pakete darf nicht ausschließlich von einer lokalen Systemuhr abhängen.
- Die maximal zulässige Uhrabweichung ist ein versionierter Protokollparameter.
- Jede neue Ende-zu-Ende-Verbindung verwendet neue ephemere Diffie-Hellman-Schlüssel.
- Diffie-Hellman dient nur der Geheimnisableitung und nicht allein der Identitätsprüfung.
- Beide Endpunkte signieren das vollständige Handshake-Transkript mit ihren delegierten Transport Keys.
- Sitzungsschlüssel werden mit HKDF aus Shared Secret und Transcript Hash abgeleitet.
- Für beide Übertragungsrichtungen werden getrennte Schlüssel und Nonce-Bereiche verwendet.
- Nutzdaten werden ausschließlich mit AEAD verschlüsselt.
- Jede Sitzung verwendet Sequenznummern gegen Wiederholung, Verlust und Vertauschung.
- Lang laufende Sitzungen führen nach definierter Dauer oder Datenmenge ein Rekeying durch.
- Ephemere Schlüssel gewährleisten Forward Secrecy für abgeschlossene Sitzungen.
- QUIC über UDP ist der bevorzugte direkte Transport.
- TCP mit authentifiziertem Verschlüsselungskanal ist der Fallback, wenn UDP nicht nutzbar ist.
- Nach Discovery wird zuerst eine direkte Verbindung versucht.
- Ist die direkte Verbindung nicht möglich, wird NAT Traversal mit signierten kurzlebigen Tokens versucht.
- Erst danach werden austauschbare Relay-Circuits verwendet.
- Relay-Nodes erhalten keinen Zugriff auf Klartext oder Sitzungsschlüssel.
- Ein Ziel veröffentlicht mehrere Relay-Optionen, soweit verfügbar.
- Relays authentifizieren Quelle, Ziel und Circuit-Token vor Ressourcenreservierung.
- Relay-Ressourcen besitzen harte Zeit-, Bandbreiten- und Verbindungsgrenzen.
- Vor teurem Handshake-Zustand muss die Quelle ein stateless Cookie zurücksenden.
- Request-Limits gelten mindestens pro Identity, Session, Ziel, Peer und Epoch.
- Bei hoher Last wird ein adaptiver Proof of Work für neue Requests verlangt.
- Die Proof-of-Work-Schwierigkeit ist lokal lastabhängig, aber protocolweit begrenzt.
- Jeder Cache besitzt harte Grenzen für Eintragszahl, Gesamtgröße und TTL.
- Ein Node darf ungültige Pakete ohne Antwort verwerfen.
- Wiederholtes ungültiges Verhalten darf lokal zu zeitlich begrenzter Drosselung führen.
- Lokale Reputation darf niemals einen gültigen Blockchain-Beweis ersetzen.
- DDoS kann physisch nicht mathematisch ausgeschlossen werden; das Protokoll minimiert Verstärkung und zwingt frühe Kostenprüfung.
- Flüchtige Requests, Responses, Leases und Reverse Routes werden nicht als dauerhafte Historie gespeichert.
- Abgelaufene flüchtige Daten werden aus dem aktiven Speicher entfernt.
- Flüchtige Daten werden nach Möglichkeit verschlüsselt gespeichert und durch Verwerfen ihrer Schlüssel kryptografisch gelöscht.
- Physisches Überschreiben auf SSDs oder verwalteten Speichern wird nicht zugesichert.
- Sicherheitsrelevantes Logging enthält standardmäßig keine privaten Schlüssel, Klartext-Locators oder Nutzdaten.
- Telemetrie ist standardmäßig deaktiviert und darf keine Identitäts- oder Beziehungsdaten übertragen.
- Softwareupdates müssen signiert und gegen einen lokal verankerten Release Key geprüft werden.
- Algorithmuswechsel sind versioniert und werden erst nach finalisierter Aktivierung wirksam.
- Kann eine Sicherheitsentscheidung nicht eindeutig geprüft werden, wird die Operation abgelehnt.
```

```mermaid
flowchart LR
    subgraph CORE["PKN Node Software"]
        DAEMON["Node-Daemon<br/>Chain, Routing, Discovery, Transport"]
        API["Lokale authentifizierte API<br/>Unix Socket oder Loopback"]
        STORE["Verschlüsselter lokaler Datenspeicher"]
        CONFIG["Versionierte Konfiguration"]
        UPDATE["Signierte Update-Prüfung"]
        DAEMON <--> API
        DAEMON <--> STORE
        DAEMON <--> CONFIG
        UPDATE --> DAEMON
    end

    subgraph KEYMGR["Schlüssel- und Identitätsverwaltung"]
        GENERATE["Master Seed sicher generieren<br/>anzeigen, Prüfsumme prüfen"]
        RESTORE["Identität aus Seed lokal wiederherstellen"]
        DERIVE["Identity-, Registry-, Discovery-,<br/>Transport- und Recovery-Keys ableiten"]
        BACKUP["Offline-Backup und Recovery-Policy"]
        ROTATE["Neuen Seed erzeugen und<br/>atomaren Key-Swap vorbereiten"]
        REVOKE["Delegierte Schlüssel widerrufen"]
        GENERATE --> DERIVE
        RESTORE --> DERIVE
        DERIVE --> BACKUP
        DERIVE --> ROTATE
        DERIVE --> REVOKE
    end

    subgraph DOMAINS[".pkn-Domainverwaltung"]
        SEARCH["Verfügbarkeit prüfen"]
        REGISTER["Domain registrieren"]
        RENEW["Domain verlängern<br/>Ablaufwarnungen"]
        TRANSFER["Einzelne oder alle Domains übertragen"]
        RELEASE["Domain bewusst freigeben"]
        SUB["Subdomains und Dienste off-chain verwalten"]
        PROOFVIEW["Owner, Ablauf, Sequenz und Merkle Proof anzeigen"]
    end

    subgraph CHAINCLIENT["Blockchain-Funktionen"]
        SYNC["Full-, Pruned- oder Compact-Synchronisierung"]
        VERIFY["Header, Finalität, Snapshots und Merkle Proofs prüfen"]
        TX["Transaktionen kanonisch erzeugen,<br/>signieren, senden und Status verfolgen"]
        VALIDATEBLOCK["Blöcke und Zustandsübergänge validieren"]
        PROPOSE["Optional Blöcke vorschlagen und voten"]
        PRUNE["Snapshots erstellen/importieren und sicher prunen"]
        SYNC --> VERIFY --> VALIDATEBLOCK
        VALIDATEBLOCK --> TX
        VALIDATEBLOCK --> PROPOSE
        VALIDATEBLOCK --> PRUNE
    end

    subgraph NETWORK["Netzwerk und Discovery"]
        BOOTSTRAP["Mehrere Bootstrap-Quellen verwalten"]
        PEERAUTH["Peer-Kanäle gegenseitig authentifizieren"]
        ROUTING["XOR-Routingtabelle aufbauen und pflegen"]
        LEASEPUB["Rendezvous-Leases publizieren und erneuern"]
        LOCATE["Locate Requests parallel senden,<br/>prüfen, cachen und weiterleiten"]
        NATMOD["NAT-Typ erkennen und Traversal versuchen"]
        RELAYMOD["Relay-Circuits anbieten und verwenden"]
        LIMITS["Rate Limits, PoW, Cookies und Cachegrenzen"]
        BOOTSTRAP --> PEERAUTH --> ROUTING
        ROUTING --> LEASEPUB
        ROUTING --> LOCATE
        LOCATE --> NATMOD --> RELAYMOD
        LIMITS --> PEERAUTH
        LIMITS --> LOCATE
        LIMITS --> RELAYMOD
    end

    subgraph RESOLUTION["Lokale Nutzung"]
        RESOLVER["Lokaler .pkn-Resolver"]
        BROWSER["Browser-Erweiterung<br/>Domainstatus, Identität und Verbindung anzeigen"]
        APPAPI["SDK/API für Anwendungen"]
        CERT["PKN-Zertifikate und Transport-Delegation prüfen"]
        CONNECTION["Direkte, NAT- oder Relay-Verbindung aufbauen"]
        RESOLVER --> CONNECTION
        BROWSER --> RESOLVER
        APPAPI --> RESOLVER
        CERT --> CONNECTION
    end

    subgraph OPERATIONS["Betrieb und Diagnose"]
        STATUS["Sync-, Peer-, Routing-, Lease- und Domainstatus"]
        HEALTH["Selbsttests für RNG, Uhr, Speicher und Netzwerk"]
        LOGS["Datensparsame Logs mit Debug-Modus"]
        EXPORT["Diagnose exportieren ohne Secrets"]
        MIGRATE["Konfiguration und Datenbank versioniert migrieren"]
        CLOCK["Epoch, Blockhöhe und Uhrabweichung überwachen"]
    end

    KEYMGR --> STORE
    KEYMGR --> DOMAINS
    DOMAINS --> TX
    CHAINCLIENT --> DAEMON
    NETWORK --> DAEMON
    RESOLUTION --> API
    DAEMON --> OPERATIONS
    ROTATE --> TRANSFER
    REVOKE --> TX
    LEASEPUB --> CONNECTION
    VERIFY --> PROOFVIEW
```
