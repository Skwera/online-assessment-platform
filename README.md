
"""# Online Assessment Management System

## Multi-Tenant Enterprise Assessment Platform

> Production-grade assessment system managing candidate evaluations across multiple KZN Provincial Treasury departments. Live at [onlineassessment.kzntreasury.gov.za](https://onlineassessment.kzntreasury.gov.za/)

---

## System Overview

The Online Assessment Management System is a **multi-tenant, full-stack platform** built for government-scale operations. It handles authentication, bulk data ingestion, timed test windows, automated token lifecycle management, asynchronous email/SMS dispatch, and PDF report generation — all deployed on Windows Server 2019 via IIS.

---

## Business Problem

Manual assessment processes across government departments created:

- **Administrative bottlenecks** — HR staff manually tracked hundreds of candidates on spreadsheets
- **Security vulnerabilities** — Paper-based tests leaked; no audit trail of who accessed what
- **Scheduling chaos** — Coordinating test windows across 5+ departments with no centralized system
- **Result fragmentation** — Scores stored in silos; no unified reporting for leadership
- **Accessibility gaps** — No accommodation system for candidates with disabilities

---

## Solution Architecture

### What We Built

| System Component | Engineering Decision |
|---|---|
| **Frontend** | React SPA with hooks-based state management, reusable component library across 6+ views |
| **Backend** | Express.js REST API with server-side pagination, multi-filter search, CSV/PDF export |
| **Database** | SQL Server with relational schema, foreign key constraints, indexing, and stored procedures |
| **Auth** | JWT with session persistence, protected route guards, rate-limiting (5 attempts/15min), bcrypt hashing |
| **Proctoring** | Real-time multi-vector cheat detection with debounced violation handling and 3-strike escalation |
| **Deployment** | IIS with iisnode module, ARR reverse proxy, dedicated application pools |

---

## Engineering Highlights

### 🔐 Authentication & Security System

- JWT-based auth with session persistence and protected route guards
- Role-based dashboard access (Admin vs. Candidate flows)
- Rate-limiting: 5 attempts per 15-minute window
- Bcrypt password hashing with legacy migration support

```javascript
// AuthGuard.js — Protects admin routes with role verification
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';

function AuthGuard({ children, requiredRole }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const navigate = useNavigate();

  useEffect(() => {
    const token = localStorage.getItem('authToken');
    const userRole = localStorage.getItem('userRole');
    
    if (!token) {
      navigate('/admin/login');
      return;
    }
    
    verifyToken(token).then(valid => {
      if (!valid || (requiredRole && userRole !== requiredRole)) {
        localStorage.clear();
        navigate('/admin/login');
      } else {
        setIsAuthenticated(true);
      }
      setIsLoading(false);
    });
  }, [navigate, requiredRole]);

  if (isLoading) return <div className="loading-spinner">Loading...</div>;
  return isAuthenticated ? children : null;
}
```

---

### 🛡️ Real-Time Proctoring & Anti-Cheat Engine

Monitors **7+ violation vectors** with debounced handling to eliminate false positives:

| Violation Vector | Detection Method |
|---|---|
| Fullscreen exit | `fullscreenchange` event + dimension check |
| Print screen | `keyup` listener for PrintScreen key |
| Dev tools (F12 / Ctrl+Shift+I/J) | Keyboard interception + window blur detection |
| Copy/paste | `copy` / `paste` event blocking |
| Tab switching / minimization | `visibilitychange` + `window.blur` |
| Window resize | Dimension delta tracking with grace period |
| Right-click context menu | `contextmenu` event prevention |

**Escalation logic:** Warning → Final Warning → Automatic Test Termination (3-strike system with 2-second debounce cooldown)

```javascript
// Debounced violation handler — prevents false positives from innocent actions
let violationCooldown = false;
let violationCount = 0;

const handleViolation = (type) => {
  if (violationCooldown) return;
  
  violationCount++;
  violationCooldown = true;
  
  if (violationCount === 1) {
    showWarningToast('Warning: Activity detected. Further violations will terminate your test.');
  } else if (violationCount === 2) {
    showWarningToast('FINAL WARNING: One more violation will auto-submit your test.');
  } else {
    terminateSession('violation');
  }
  
  setTimeout(() => violationCooldown = false, 2000); // 2s debounce
};
```

---

### 📊 Bulk Data Ingestion Pipeline

Excel/XLSX parser handling **hundreds of candidate records** with validation, error handling, and automated one-time token generation per candidate.

```javascript
// Server-side paginated candidate management with multi-filter search
router.get('/api/candidates', async (req, res) => {
  try {
    const { name, test, status, page = 1, limit = 50 } = req.query;
    const offset = (page - 1) * limit;
    
    const pool = await poolPromise;
    const result = await pool.request()
      .input('name', sql.NVarChar, `%${name}%`)
      .input('test', sql.NVarChar, test)
      .input('status', sql.NVarChar, status)
      .input('offset', sql.Int, offset)
      .input('limit', sql.Int, parseInt(limit))
      .query(`
        SELECT * FROM Candidates 
        WHERE (@name = '' OR Name LIKE @name)
          AND (@test = '' OR TestName = @test)
          AND (@status = '' OR Status = @status)
        ORDER BY CreatedDate DESC
        OFFSET @offset ROWS FETCH NEXT @limit ROWS ONLY;
        
        SELECT COUNT(*) as total FROM Candidates
        WHERE (@name = '' OR Name LIKE @name)
          AND (@test = '' OR TestName = @test)
          AND (@status = '' OR Status = @status);
      `);
    
    res.json({
      candidates: result.recordsets[0],
      totalPages: Math.ceil(result.recordsets[1][0].total / limit)
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

---

### ⚡ Asynchronous Job Processing

Non-blocking invitation dispatch system with **live progress tracking**, handling 50+ concurrent email/SMS sends without frontend timeout. Jobs stored in memory with auto-cleanup.

---

### 🗄️ Database Architecture

Relational schema (SQL Server) managing:
- Departments, Tests, Questions, Candidates, Assignments, Token States
- Foreign key constraints and indexing for paginated queries
- Stored procedures for answer scoring

---

### 📱 SMS Dispatch Integration

CSV generation pipeline conforming to **Vodacom bulk SMS gateway specifications**, with file-based dispatch logging and whitelist-aware email routing.

---

### ⏰ Time-Bound Access Control

- Token lifecycle with configurable start/end windows
- Real-time countdown sync between server (SAST) and client
- Session recovery: validates JWT on re-entry and restores test state without data loss or timer reset

---

### ♿ Accessibility & Accommodation System

- Disability accommodation module flags candidates with accessibility needs
- Automatic time extensions (+10 minutes)
- Accommodation badges displayed in test interface

---

### 📄 PDF Report Generation

Server-side PDF generation (jsPDF) for candidate result reports and audit documentation.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React.js, React Router, CSS3, JavaScript (ES6+) |
| Backend | Node.js, Express.js |
| Database | SQL Server |
| Auth | JWT, bcrypt |
| PDF | jsPDF |
| Deployment | IIS, iisnode, ARR Reverse Proxy, Windows Server 2019 |

---

## Deployment

```
Windows Server 2019
├── IIS with iisnode module
├── ARR Reverse Proxy
├── Dedicated Application Pools (process isolation + auto-restart)
└── SQL Server (backend database)
```

---

## Key Metrics

- **200+** concurrent candidates tested simultaneously
- **7+** anti-cheat violation vectors monitored in real-time
- **50+** concurrent email/SMS dispatches handled asynchronously
- **Zero** false terminations after debounce optimization
- **5** government departments onboarded

---

## What I Learned

- **Systems thinking:** Tracing bugs from UI state → API contracts → database queries → server configuration
- **Security engineering:** Building multi-layer protection without breaking user experience
- **Performance optimization:** Server-side pagination, indexing strategies, and async job queues
- **Production operations:** IIS deployment, reverse proxy configuration, and Windows Service integration
- **Stakeholder communication:** Translating non-technical government requirements into technical specifications

---

## Recognition

**Certificate of Recognition — KZN Provincial Treasury**

Awarded by the **Head of Department (Carol Coetzee)** for innovative system design and delivery. Recognized for end-to-end architecture, production deployment, and stakeholder impact across multiple government departments.

---

*Built during Software Engineer internship at KZN Provincial Treasury | 2024–2025*
"""

with open('/mnt/agents/output/online-assessment-platform-README.md', 'w', encoding='utf-8') as f:
    f.write(readme)

print(f"README saved: {len(readme)} characters")
