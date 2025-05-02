// 1. Імпортуємо необхідні модулі
const express = require('express');
const bodyParser = require('body-parser');
const { Pool } = require('pg');
require('dotenv').config();

// 2. Налаштування Express-серверу
const app = express();
const port = 3000;

app.use(bodyParser.json());

// 3. Налаштування з'єднання з базою даних PostgreSQL
const pool = new Pool({
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 5432,
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || 'vi349',
    database: process.env.DB_NAME || 'university'
});

// 4. Створення таблиці students (якщо вона не існує)
const createTable = async () => {
    await pool.query(`
        CREATE TABLE IF NOT EXISTS students (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            age INTEGER NOT NULL,
            major VARCHAR(100) NOT NULL
        );
    `);

    // Додавання початкових записів, якщо таблиця порожня
    const result = await pool.query('SELECT COUNT(*) FROM students');
    if (parseInt(result.rows[0].count) === 0) {
        await pool.query(`
            INSERT INTO students (name, age, major) VALUES
            ('Sophia Carter', 21, 'Computer Science'),
            ('Jackson Reed', 23, 'Machine Learning'),
            ('Ava Thompson', 20, 'Information Security'),
            ('Ethan Brooks', 22, 'Robotics');
        `);
    }
};

// Викликаємо функцію створення таблиці та додавання даних
createTable().catch((err) => console.error('Error creating table:', err));

// 5. API для CRUD операцій

// Create (POST /api/students)
app.post('/api/students', async (req, res) => {
    const { name, age, major } = req.body;
    try {
        const result = await pool.query(
            'INSERT INTO students (name, age, major) VALUES ($1, $2, $3) RETURNING *',
            [name, age, major]
        );
        res.status(201).json(result.rows[0]);
    } catch (err) {
        console.error(err);
        res.status(500).json({ message: 'Error adding student' });
    }
});

// Read (GET /api/students)
app.get('/api/students', async (req, res) => {
    try {
        const result = await pool.query('SELECT * FROM students');
        res.json(result.rows);
    } catch (err) {
        console.error(err);
        res.status(500).json({ message: 'Error fetching students' });
    }
});

// Read by ID (GET /api/students/:id)
app.get('/api/students/:id', async (req, res) => {
    const { id } = req.params;
    try {
        const result = await pool.query('SELECT * FROM students WHERE id = $1', [id]);
        if (result.rows.length > 0) {
            res.json(result.rows[0]);
        } else {
            res.status(404).json({ message: 'Student not found' });
        }
    } catch (err) {
        console.error(err);
        res.status(500).json({ message: 'Error fetching student' });
    }
});

// Update (PUT /api/students/:id)
app.put('/api/students/:id', async (req, res) => {
    const { id } = req.params;
    const { name, age, major } = req.body;
    try {
        const result = await pool.query(
            'UPDATE students SET name = $1, age = $2, major = $3 WHERE id = $4 RETURNING *',
            [name, age, major, id]
        );
        if (result.rows.length > 0) {
            res.json(result.rows[0]);
        } else {
            res.status(404).json({ message: 'Student not found' });
        }
    } catch (err) {
        console.error(err);
        res.status(500).json({ message: 'Error updating student' });
    }
});

// Delete (DELETE /api/students/:id)
app.delete('/api/students/:id', async (req, res) => {
    const { id } = req.params;
    try {
        const result = await pool.query('DELETE FROM students WHERE id = $1 RETURNING *', [id]);
        if (result.rows.length > 0) {
            res.status(204).send();
        } else {
            res.status(404).json({ message: 'Student not found' });
        }
    } catch (err) {
        console.error(err);
        res.status(500).json({ message: 'Error deleting student' });
    }
});

// 6. Запуск сервера
app.listen(port, () => {
    console.log(`Server is running on http://localhost:${port}`);
});
