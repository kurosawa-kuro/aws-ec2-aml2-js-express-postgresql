# aws-ec2-aml2-js-express-postgresql

```
# 既存のnodeを完全に削除
sudo yum remove -y nodejs
rm -rf ~/.nvm
rm -rf ~/.npm
rm -rf ~/.node-gyp

# 既存のNodeJSリポジトリをクリーン
sudo rm -rf /etc/yum.repos.d/nodesource*

# nvmインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# nvmを現在のシェルで有効化
source ~/.bashrc

# Node.js 14をインストール
nvm install 14

# Node.js 14を使用
nvm use 14

# 確認
node -v
npm -v
```

```
// app.js
require('dotenv').config();
const express = require('express');
const { Pool } = require('pg');
const app = express();
const port = 3000;

console.log("process.env.DB_USER",process.env.DB_USER)
console.log("process.env.DB_HOST",process.env.DB_HOST)
console.log("process.env.DB_NAME",process.env.DB_NAME)
console.log("process.env.DB_PASSWORD",process.env.DB_PASSWORD)
console.log("process.env.DB_PORT",process.env.DB_PORT)

// PostgreSQL接続設定
const pool = new Pool({
    user: process.env.DB_USER,
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    password: process.env.DB_PASSWORD,
    port: process.env.DB_PORT || 5432,
    ssl: {
      rejectUnauthorized: false
    }
  });

// JSONパーサーミドルウェア
app.use(express.json());

// テーブル作成用のSQL
const createTableQuery = `
  CREATE TABLE IF NOT EXISTS microposts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
`;



// 新規投稿を作成
app.post('/posts', async (req, res) => {
  try {
    const { title } = req.body;
    const result = await pool.query(
      'INSERT INTO microposts (title) VALUES ($1) RETURNING *',
      [title]
    );
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// 全ての投稿を取得
app.get('/posts', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT * FROM microposts ORDER BY created_at DESC'
    );
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Setup endpoint
app.get('/dev/setup', async (req, res) => {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');

    // Create table
    await client.query(createTableQuery);
    console.log('Created microposts table');

    // Insert seed data
    const insertQuery = `
      INSERT INTO microposts (title)
      VALUES ($1)
    `;

    for (const post of seedData) {
      await client.query(insertQuery, [post.title]);
    }

    await client.query('COMMIT');
    
    res.json({
      message: 'Database setup completed successfully',
      seededCount: seedData.length
    });

  } catch (err) {
    await client.query('ROLLBACK');
    res.status(500).json({
      error: 'Database setup failed',
      details: err.message
    });
  } finally {
    client.release();
  }
});

 app.get('/', async (req, res) => {
     res.json("Hello world");
 });

// サーバー起動
app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

```
DB_USER=my_username
DB_HOST=my-express-db.cvo2u2omijrr.ap-northeast-1.rds.amazonaws.com
DB_NAME=microposts
DB_PASSWORD=my_password
DB_PORT=5432
```