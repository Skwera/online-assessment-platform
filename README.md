# Online Assessment Platform

## Secure Enterprise Candidate Assessment System

A full-stack enterprise web application developed during my software development internship at KwaZulu-Natal Treasury.

---

## Project Overview

The Online Assessment Platform is a secure web-based system designed to manage candidate testing, assessment delivery, and evaluation workflows.

The system automates candidate onboarding, test assignment, secure authentication, and controlled assessment access.

---

## Business Problem

Manual assessment processes often create:

- Administrative delays
- Security risks
- Candidate management challenges
- Scheduling inefficiencies
- Difficult result tracking

---

## Solution

The platform digitises assessment management through:

- Automated candidate assignment
- Secure token-based access
- Time-restricted assessments
- Centralised test administration
- Structured result management

---

## My Role

### Software Development Intern
KwaZulu-Natal Treasury

Led development of core system functionality including:

- Full-stack application development
- Database schema design
- Authentication implementation
- Admin dashboard development
- Candidate management workflows
- Testing and debugging

---

## Technology Stack

### Frontend
- React
- HTML5
- CSS3
- JavaScript

### Backend
- Node.js
- Express.js

### Database
- SQL Server

---

## Core Features

## Admin Dashboard

- Create assessments
- Manage tests
- Add questions
- Configure duration
- Assign candidates

---

## Candidate Management

- Candidate profile creation
- Test assignment
- Token generation
- Due-date management

---

## Security Features

- Admin authentication
- Token-based candidate access
- Access expiry controls
- Time-restricted test availability

---

## Database Design

Implemented structured relational database design including:

- Administrators
- Tests
- Questions
- Candidate records
- Assignments
- Access tokens

---

## Software Engineering Contributions

- API development
- Database integration
- Authentication workflows
- Business logic implementation
- Application testing
- Performance troubleshooting

---

## Key Technical Skills Demonstrated

- Full-stack development
- Secure access control
- Enterprise application design
- SQL Server integration
- Problem solving
- Software maintenance

---

## Professional Impact

The platform improves assessment administration efficiency while providing secure, scalable digital candidate evaluation.

---

## Learning Outcomes

This project strengthened my practical expertise in:

- Enterprise system architecture
- React application development
- Backend API engineering
- Database-driven applications
- Security implementation

---

## Code Samples

### Authentication & Route Guards
Secure JWT-based authentication with role-based access control protecting admin routes.

```javascript
// AuthGuard.js — Protects admin dashboard routes
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';

function AuthGuard({ children, requiredRole }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const navigate = useNavigate();

  useEffect(() =&gt; {
    const token = localStorage.getItem('authToken');
    const userRole = localStorage.getItem('userRole');
    
    if (!token) {
      navigate('/admin/login');
      return;
    }
    
    verifyToken(token).then(valid =&gt; {
      if (!valid || (requiredRole && userRole !== requiredRole)) {
        localStorage.clear();
        navigate('/admin/login');
      } else {
        setIsAuthenticated(true);
      }
      setIsLoading(false);
    });
  }, [navigate, requiredRole]);

  if (isLoading) return &lt;div className="loading-spinner"&gt;Loading...&lt;/div&gt;;
  return isAuthenticated ? children : null;
}

export default AuthGuard;

// Candidates.js — Paginated candidate management
const [candidates, setCandidates] = useState([]);
const [filters, setFilters] = useState({ name: '', test: '', status: '' });
const [page, setPage] = useState(1);
const [totalPages, setTotalPages] = useState(1);

useEffect(() => {
  fetchCandidates();
}, [filters, page]);

const fetchCandidates = async () => {
  const response = await axios.get('/api/candidates', {
    params: { ...filters, page, limit: 50 }
  });
  setCandidates(response.data.candidates);
  setTotalPages(response.data.totalPages);
};

// CSV Export for audit and reporting
const exportCSV = () => {
  const headers = 'Name,Test,Status,Score,Date\\n';
  const rows = candidates.map(c => 
    `${c.name},${c.test},${c.status},${c.score},${c.date}`
  ).join('\\n');
  downloadFile(headers + rows, 'candidates-report.csv');
};

// TestBuilder.js — Flexible question creation
const questionTypes = {
  MULTIPLE_CHOICE: 'multiple_choice',
  MULTIPLE_CORRECT: 'multiple_correct',
  TEXT: 'text'
};

const [questions, setQuestions] = useState([{
  type: questionTypes.MULTIPLE_CHOICE,
  text: '',
  options: ['', '', '', ''],
  correctAnswers: [],
  marks: 1
}]);

const addQuestion = (type) => {
  setQuestions([...questions, {
    type,
    text: '',
    options: type === questionTypes.TEXT ? [] : ['', '', '', ''],
    correctAnswers: [],
    marks: 1
  }]);
};

// Server-side auto-grading logic
const gradeTest = (candidateAnswers, correctAnswers) => {
  return correctAnswers.map((correct, index) => {
    if (Array.isArray(correct)) {
      // Multiple correct: partial marking
      const correctCount = correct.filter(c => 
        candidateAnswers[index]?.includes(c)
      ).length;
      return (correctCount / correct.length) * questions[index].marks;
    }
    return candidateAnswers[index] === correct ? questions[index].marks : 0;
  });
};

// server/routes/candidates.js
const express = require('express');
const router = express.Router();
const { sql, poolPromise } = require('../models/database');

// GET /api/candidates — Paginated with filters
router.get('/', async (req, res) => {
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

module.exports = router;
