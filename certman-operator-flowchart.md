# Certman-Operator Logic Flowchart

```mermaid
graph TD
    A[Start: Operator Launch] --> B[Initialize Controllers]
    B --> C[ClusterDeployment Controller]
    B --> D[CertificateRequest Controller]

    %% ClusterDeployment Controller Flow
    C --> E{ClusterDeployment Event?}
    E -->|Yes| F[Validate Cluster]
    E -->|No| E

    F --> G{Is Managed Cluster?<br/>api.openshift.com/managed=true}
    G -->|No| H[Skip - Return]
    G -->|Yes| I{Is Fake Cluster?}

    I -->|Yes| H
    I -->|No| J{Is Installed?}

    J -->|No| H
    J -->|Yes| K{Is Relocating?}

    K -->|Yes| H
    K -->|No| L{Has DeletionTimestamp?}

    L -->|Yes| M[Delete CertificateRequests<br/>Remove Finalizer]
    L -->|No| N[Add Finalizer if Missing]

    M --> H
    N --> O[Process CertificateBundles]

    O --> P{For each Bundle<br/>Generate=true?}
    P -->|Yes| Q[Extract Domains<br/>Control Plane + Ingress]
    P -->|No| P

    Q --> R[Create/Update<br/>CertificateRequest]
    R --> S[Update Status]
    S --> H

    %% CertificateRequest Controller Flow
    D --> T{CertificateRequest Event?}
    T -->|Yes| U[Validate Request]
    T -->|No| T

    U --> V{Has DeletionTimestamp?}
    V -->|Yes| W[Revoke Certificate<br/>Delete Secret<br/>Remove Finalizer]
    V -->|No| X[Add Finalizer if Missing]

    W --> Y[End]
    X --> Z{Is Relocating?}

    Z -->|Yes| AA[Update Status: Relocating<br/>Skip Reconcile]
    Z -->|No| BB{Secret Exists?}

    AA --> Y
    BB -->|No| CC[Issue New Certificate]
    BB -->|Yes| DD[Check if Renewal Needed]

    %% Certificate Issuance Flow
    CC --> EE[Validate DNS Write Access]
    EE --> FF{DNS Access OK?}
    FF -->|No| GG[Error: No DNS Access]
    FF -->|Yes| HH[Update Let's Encrypt Account]

    GG --> Y
    HH --> II[Create Let's Encrypt Order]
    II --> JJ[For each Domain: DNS-01 Challenge]

    JJ --> KK[Set TXT Record<br/>_acme-challenge]
    KK --> LL[Verify DNS Propagation<br/>via Cloudflare DoH]
    LL --> MM{Verification OK?}

    MM -->|No| NN[Error: DNS Verification Failed]
    MM -->|Yes| OO[Update Challenge Status]

    NN --> Y
    OO --> PP[Generate RSA Key Pair]
    PP --> QQ[Create Certificate Signing Request]
    QQ --> RR[Finalize Let's Encrypt Order]
    RR --> SS[Fetch Certificates]
    SS --> TT[Store in Kubernetes Secret]
    TT --> UU[Clean DNS Challenge Records]
    UU --> VV[Update Status: Issued]
    VV --> Y

    %% Certificate Renewal Flow
    DD --> WW{Days Until Expiry ≤<br/>ReissueBeforeDays?}
    WW -->|No| XX{DNS Names Match?}
    WW -->|Yes| CC

    XX -->|No| CC
    XX -->|Yes| YY[Update Status: Valid]
    YY --> Y

    %% Styling
    classDef controller fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef action fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef end fill:#f3e5f5,stroke:#4a148c,stroke-width:2px

    class C,D controller
    class E,G,I,J,K,L,P,T,V,Z,BB,FF,MM,WW,XX decision
    class F,N,O,Q,R,S,U,X,EE,HH,II,JJ,KK,LL,OO,PP,QQ,RR,SS,TT,UU,VV,YY action
    class GG,NN error
    class H,Y,AA end
```

## Key Flow Explanations

### 1. ClusterDeployment Controller
- **Entry Point**: Watches for ClusterDeployment resource changes
- **Validation Chain**: Managed → Not Fake → Installed → Not Relocating
- **Action**: Creates CertificateRequest resources for each CertificateBundle with Generate=true

### 2. CertificateRequest Controller
- **Entry Point**: Watches for CertificateRequest resource changes
- **Decision Point**: Check if secret exists and if certificate needs renewal
- **Action**: Issues new certificates or renews existing ones

### 3. Certificate Issuance Process
- **DNS Validation**: Ensures write access to DNS zone
- **Let's Encrypt Flow**: Account → Order → Challenge → Verification → CSR → Certificate
- **DNS-01 Challenge**: Creates `_acme-challenge` TXT records, verifies via Cloudflare DoH

### 4. Renewal Logic
- **Default**: 45 days before expiry
- **Triggers**: Time-based OR DNS names mismatch
- **Process**: Same as initial issuance

### 5. Platform Support
- **AWS**: Route53 DNS management
- **GCP**: Cloud DNS management
- **Azure**: DNS Zone management
- **FedRAMP**: Special hosted zone handling