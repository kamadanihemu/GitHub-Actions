## 🌿 Branching Strategy Guide

## 📋 Overview

This document provides detailed information about the branching strategies used for different types of services in our microservices architecture.

#### 🚀 Environment Deployment Matrix

| Environment | Allowed Source Branches | Build Type | Ownership & Notes |
|-------------|------------------------|------------|-------------------|
| **DEV** | `develop`, `feature/*` | Direct Build | **Owner**: Dev Team<br/>Development testing environment |
| **QA** | `qa` only | Direct Build | **Owner**: Dev/QA TL<br/>Quality assurance testing |
| **STAGE** | `release/*`, `hotfix/*` | Direct Build | **Owner**: DevOps Team<br/>Pre-production staging |
| **PROD** | **Backend**: Promotion from STAGE only<br/>**UI**: `release/*` (direct build only)<br/>**Hotfix**: `hotfix/*` (emergency deploy) | **Backend**: Promotion only<br/>**UI**: Direct Build only<br/>**Hotfix**: Direct Deploy | **Owner**: DevOps Team<br/>Backend: No direct builds<br/>UI: Direct build from `release/*`, no promotion<br/>Hotfix: Emergency deployment allowed |


#### 🔄 Branch Flow & Environment Strategy

```mermaid
flowchart LR
    subgraph "Development Flow"
        MAIN[main] 
        FEAT[feature/*]
        DEV[develop]
        QA[qa]
        REL[release/*]
        HF[hotfix/*]
    end

    subgraph "Environment Targets"
        ENV_DEV["DEV Environment"]
        ENV_QA["QA Environment"]
        ENV_STAGE["STAGE Environment"]
        ENV_PROD["PROD Environment"]
    end

    %% Branch Flow
    FEAT -->|PR| DEV
    FEAT -->|PR| QA
    DEV -->|PR| QA
    QA -->|Merge features| REL
    REL -->|Cut from| MAIN

    %% Environment Deployments
    FEAT -.->|Build & Deploy| ENV_DEV
    DEV -->|Build & Deploy| ENV_DEV
    QA -->|Build & Deploy| ENV_QA
    REL -->|Build & Deploy| ENV_STAGE
    HF -->|Build & Deploy| ENV_STAGE

    %% Production Deployment
    ENV_STAGE -->|Backend: Promote| ENV_PROD
    REL -->|UI: Direct Build & Deploy| ENV_PROD
    HF -->|Hotfix: Deploy| ENV_PROD

    %% Post-Production Merges
    REL -.->|After stable prod| MAIN
    HF -.->|After stable prod| MAIN

    %% Emergency Path
    MAIN -->|Emergency| HF

    %% No deployment from main
    MAIN -.->|No Deployment| ENV_PROD

    %% Styling
    classDef branchStyle fill:#f9f9f9,stroke:#333,stroke-width:2px
    classDef envStyle fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef prodStyle fill:#ffebee,stroke:#c62828,stroke-width:2px

    class MAIN,DEV,FEAT,QA,REL,HF branchStyle
    class ENV_DEV,ENV_QA,ENV_STAGE envStyle
    class ENV_PROD prodStyle
```

#### Key Rules

1. **Main Branch**: 
   - ❌ No deployments from `main`
   - ✅ Only receives merges from `release/*` and `hotfix/*` after stable production deployment

2. **Production Deployment**:
   - 🔄 **Backend**: Promotion from STAGE environment only
   - ✅ **UI**: Direct build from `release/*` branch only (no promotion)
   - 🚨 **Hotfix**: Direct deployment from `hotfix/*` for emergency fixes
   - ⚠️ **Emergency Only**: Hotfix to PROD should be used only for critical production issues

3. **Branch Protection**:
   - Each environment can only build from its designated branches
   - No cross-environment contamination

#### ⚖️ UI vs Backend Production Deployment

```mermaid
flowchart TD
    subgraph "Release Branch"
        REL[release/*]
    end
    
    subgraph "STAGE Environment"
        STAGE[STAGE]
    end
    
    subgraph "PROD Environment"
        PRODUI[PROD - UI Services]
        PRODBE[PROD - Backend Services]
    end
    
    %% Stage deployment
    REL -->|Build & Deploy| STAGE
    
    %% UI Production - Direct build only
    REL -->|✅ Direct Build Only| PRODUI
    
    %% Backend Production - Promotion only
    STAGE -->|✅ Promotion Only| PRODBE
    REL -.->|❌ No Direct Build| PRODBE
    
    %% Styling
    classDef uiBox fill:#e8f5e8
    classDef backendBox fill:#fff2e8
    classDef stageBox fill:#e1f5fe
    classDef releaseBox fill:#f3e5f5
    
    class PRODUI uiBox
    class PRODBE backendBox
    class STAGE stageBox
    class REL releaseBox
```

### 🚨 Emergency Hotfix Process

```mermaid
sequenceDiagram
    participant PROD as Production Issue
    participant HF as hotfix/* branch
    participant STAGE as STAGE Environment
    participant MAIN as main branch
    participant REL as release/* branch
    participant QA as qa branch
    participant DEV as develop branch

    Note over PROD: Critical production issue detected
    
    PROD->>HF: 1. Create hotfix/* branch from main
    HF->>HF: 2. Develop fix
    HF->>STAGE: 3. Build & deploy to STAGE
    
    Note over STAGE: Test hotfix in staging
    
    STAGE->>PROD: 4. Promote to PRODUCTION
    
    Note over PROD: After stable deployment
    
    HF->>MAIN: 5. Merge hotfix/* to main
    
    Note over MAIN,DEV: Backport fix to other branches
    MAIN->>REL: 6. Merge/Cherry-pick to active release/*
    REL->>QA: 7. Merge/Cherry-pick to qa
    QA->>DEV: 8. Merge/Cherry-pick to develop
```


## 🔐 Merging & Scan Strategy Overview

#### PR Quality Gates & Security Scans

```mermaid
flowchart TD
    subgraph "Pull Request Triggers"
        PR_DEV[PR to develop]
        PR_QA[PR to qa]
        PR_MAIN[PR to main]
    end
    
    subgraph "Security Scans"
        SAST["SAST Pipeline<br/>(SonarQube Scan)"]
        MEND["Mend Scan<br/>(Dependency Security)"]
        CODEQL["CodeQL Scan<br/>(GitHub Security)"]
    end
    
    subgraph "Quality Gates"
        BLOCKING["BLOCKING SCANS<br/>Must Pass to Merge"]
        NON_BLOCKING["NON-BLOCKING SCAN<br/>Informational Only"]
        REVIEW["Code Review"]
        MERGE["Merge Approved"]
    end
    
    %% PR triggers all scans
    PR_DEV --> SAST
    PR_DEV --> MEND
    PR_DEV --> CODEQL
    
    PR_QA --> SAST
    PR_QA --> MEND
    PR_QA --> CODEQL
    
    PR_MAIN --> SAST
    PR_MAIN --> MEND
    PR_MAIN --> CODEQL
    
    %% Blocking vs Non-blocking
    SAST --> BLOCKING
    MEND --> BLOCKING
    CODEQL --> NON_BLOCKING
    
    %% Merge flow
    BLOCKING --> REVIEW
    NON_BLOCKING -.-> REVIEW
    REVIEW --> MERGE
    
    %% Styling
    classDef prStyle fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef scanStyle fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef blockStyle fill:#ffebee,stroke:#d32f2f,stroke-width:2px
    classDef nonBlockStyle fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef approveStyle fill:#e8f5e8,stroke:#388e3c,stroke-width:2px
    
    class PR_DEV,PR_QA,PR_MAIN prStyle
    class SAST,MEND,CODEQL scanStyle
    class BLOCKING blockStyle
    class NON_BLOCKING nonBlockStyle
    class REVIEW,MERGE approveStyle
```

#### Scan Strategy Rules

| Scan Type | Description | Target Branches | Merge Blocking | Action Required |
|-----------|-------------|----------------|----------------|-----------------|
| **SAST Pipeline** | SonarQube code quality & security scan | `develop`, `qa`, `main` | ✅ **BLOCKING** | Issues must be fixed before merge |
| **Mend Scan** | Dependency vulnerability scan | `develop`, `qa`, `main` | ✅ **BLOCKING** | Vulnerabilities must be resolved |
| **CodeQL Scan** | GitHub advanced security analysis | `develop`, `qa`, `main` | ❌ **NON-BLOCKING** | Informational, merge allowed |

#### Quality Gate Process

1. **PR Creation** → Automatically triggers all 3 scans
2. **SAST + Mend** → Must pass for merge approval
3. **CodeQL** → Runs in parallel, provides security insights
4. **Code Review** → Human review after scan validation
5. **Merge** → Approved only after blocking scans pass + review approval

#### Scan Failure Handling

```mermaid
flowchart LR
    SCAN_FAIL["SAST/Mend Failure"]
    FIX_CODE["Fix Issues in Code"]
    RERUN["Auto Re-scan on Push"]
    PASS["Scans Pass"]
    MERGE_OK["Ready for Review & Merge"]
    
    SCAN_FAIL --> FIX_CODE
    FIX_CODE --> RERUN
    RERUN --> PASS
    PASS --> MERGE_OK
    
    classDef failStyle fill:#ffebee,stroke:#d32f2f,stroke-width:2px
    classDef fixStyle fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef passStyle fill:#e8f5e8,stroke:#388e3c,stroke-width:2px
    
    class SCAN_FAIL failStyle
    class FIX_CODE,RERUN fixStyle
    class PASS,MERGE_OK passStyle
```

