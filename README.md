# internship-api-
# 1. नया प्रोजेक्ट फोल्डर बनाएं और उसमें जाएं
mkdir internship-api && cd internship-api

# 2. package.json फ़ाइल बनाएं
cat << 'EOF' > package.json
{
  "name": "internship-api",
  "version": "1.0.0",
  "description": "REST API for internship records with validation, pagination, and SQLite persistence",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "seed": "node seed.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "sqlite3": "^5.1.7"
  }
}
EOF

# 3. server.js फ़ाइल बनाएं
cat << 'EOF' > server.js
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const app = express();
const PORT = 3000;

app.use(express.json());

const db = new sqlite3.Database('./internships.db', (err) => {
    if (err) console.error('Database connection error:', err.message);
    else console.log('Connected to the SQLite database.');
});

db.run(`
    CREATE TABLE IF NOT EXISTS internships (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        candidate_name TEXT NOT NULL,
        role TEXT NOT NULL,
        company TEXT NOT NULL,
        status TEXT CHECK(status IN ('Applied', 'Interviewing', 'Accepted', 'Rejected')) DEFAULT 'Applied',
        start_date TEXT NOT NULL
    )
`);

const validateInternship = (req, res, next) => {
    const { candidate_name, role, company, status, start_date } = req.body;
    const validStatuses = ['Applied', 'Interviewing', 'Accepted', 'Rejected'];

    if (!candidate_name || !role || !company || !start_date) {
        return res.status(400).json({ error: "Fields (candidate_name, role, company, start_date) are required." });
    }
    if (status && !validStatuses.includes(status)) {
        return res.status(400).json({ error: "Status must be 'Applied', 'Interviewing', 'Accepted', or 'Rejected'." });
    }
    next();
};

app.post('/api/internships', validateInternship, (req, res) => {
    const { candidate_name, role, company, status = 'Applied', start_date } = req.body;
    const sql = `INSERT INTO internships (candidate_name, role, company, status, start_date) VALUES (?, ?, ?, ?, ?)`;
    db.run(sql, [candidate_name, role, company, status, start_date], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.status(201).json({ id: this.lastID, candidate_name, role, company, status, start_date });
    });
});

app.get('/api/internships', (req, res) => {
    let page = parseInt(req.query.page) || 1;
    let limit = parseInt(req.query.limit) || 10;
    let offset = (page - 1) * limit;

    db.get(`SELECT COUNT(*) as total FROM internships`, [], (err, row) => {
        if (err) return res.status(500).json({ error: err.message });
        const totalRecords = row.total;

        db.all(`SELECT * FROM internships LIMIT ? OFFSET ?`, [limit, offset], (err, rows) => {
            if (err) return res.status(500).json({ error: err.message });
            res.json({
                pagination: { total_records: totalRecords, current_page: page, limit, total_pages: Math.ceil(totalRecords / limit) },
                data: rows
            });
        });
    });
});

app.get('/api/internships/:id', (req, res) => {
    db.get(`SELECT * FROM internships WHERE id = ?`, [req.params.id], (err, row) => {
        if (err) return res.status(500).json({ error: err.message });
        if (!row) return res.status(404).json({ error: "Record not found." });
        res.json({ data: row });
    });
});

app.put('/api/internships/:id', validateInternship, (req, res) => {
    const { candidate_name, role, company, status, start_date } = req.body;
    const sql = `UPDATE internships SET candidate_name = ?, role = ?, company = ?, status = ?, start_date = ? WHERE id = ?`;
    db.run(sql, [candidate_name, role, company, status, start_date, req.params.id], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        if (this.changes === 0) return res.status(404).json({ error: "Record not found." });
        res.json({ message: "Record updated successfully." });
    });
});

app.delete('/api/internships/:id', (req, res) => {
    db.run(`DELETE FROM internships WHERE id = ?`, [req.params.id], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        if (this.changes === 0) return res.status(404).json({ error: "Record not found." });
        res.json({ message: "Record deleted successfully." });
    });
});

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
EOF

# 4. seed.js फ़ाइल बनाएं
cat << 'EOF' > seed.js
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./internships.db');

db.serialize(() => {
    db.run(`
        CREATE TABLE IF NOT EXISTS internships (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            candidate_name TEXT NOT NULL,
            role TEXT NOT NULL,
            company TEXT NOT NULL,
            status TEXT CHECK(status IN ('Applied', 'Interviewing', 'Accepted', 'Rejected')) DEFAULT 'Applied',
            start_date TEXT NOT NULL
        )
    `);

    const stmt = db.prepare(`INSERT INTO internships (candidate_name, role, company, status, start_date) VALUES (?, ?, ?, ?, ?)`);
    const dummyData = [
        ["Amit Sharma", "Backend Intern", "Google", "Interviewing", "2026-11-01"],
        ["Priya Patel", "Frontend Developer", "Meta", "Accepted", "2026-10-15"],
        ["Rohan Das", "Fullstack Intern", "Amazon", "Applied", "2026-12-01"]
    ];

    dummyData.forEach(row => stmt.run(row));
    stmt.finalize();
    console.log("Seed data successfully loaded into internships.db!");
});
db.close();
EOF

# 5. README.md फ़ाइल बनाएं
cat << 'EOF' > README.md
# Internship Records REST API

A predictable and safe Node.js and Express REST API with SQLite persistence, validation, and pagination.

## 1. Database Schema
The database uses SQLite with a table named `internships` containing the following fields:
- `id`: INTEGER PRIMARY KEY AUTOINCREMENT
- `candidate_name`: TEXT (NOT NULL)
- `role`: TEXT (NOT NULL)
- `company`: TEXT (NOT NULL)
- `status`: TEXT (Allowed: 'Applied', 'Interviewing', 'Accepted', 'Rejected')
- `start_date`: TEXT (NOT NULL)

## 2. Setup Instructions
Follow these steps to run the application locally:

1. Install dependencies:
   ```bash
   npm install
   ```
2. Run the Seed script to load sample data:
   ```bash
   npm run seed
   ```
3. Start the API server:
   ```bash
   npm start
   ```

## 3. API Examples

### Create Record
- **Method:** `POST`
- **URL:** `/api/internships`
- **Body (JSON):**
  ```json
  {
    "candidate_name": "Raj Malhotra",
    "role": "NodeJS Intern",
    "company": "Tech Labs",
    "status": "Applied",
    "start_date": "2026-11-10"
  }
  ```

### List Records (with Pagination)
- **Method:** `GET`
- **URL:** `/api/internships?page=1&limit=2`
EOF

# 6. डिपेंडेंसीज इंस्टॉल करें
npm install

echo "सफलतापूर्वक! सभी फ़ाइलें 'internship-api' फ़ोल्डर में बन चुकी हैं और सेटअप तैयार है।"
