
const express = require("express");
const session = require("express-session");
const PgSession = require("connect-pg-simple")(session);
const { Pool } = require("pg");
const bcrypt = require("bcryptjs");

const app = express();
const PORT = process.env.PORT || 3000;

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.DATABASE_URL?.includes("localhost")
    ? false
    : { rejectUnauthorized: false }
});

app.set("trust proxy", 1);
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

app.use(session({
  store: new PgSession({
    pool,
    tableName: "user_sessions",
    createTableIfMissing: true
  }),
  secret: process.env.SESSION_SECRET || "troopa-do-galo-troque-esta-chave",
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 1000 * 60 * 60 * 12
  }
}));

const money = n => Number(n || 0).toLocaleString("pt-BR", {
  style: "currency",
  currency: "BRL"
});

const esc = s => String(s ?? "").replace(/[&<>"']/g, c => ({
  "&": "&amp;",
  "<": "&lt;",
  ">": "&gt;",
  '"': "&quot;",
  "'": "&#39;"
}[c]));

const today = () => new Date().toLocaleDateString(
  "en-CA",
  { timeZone: "America/Sao_Paulo" }
);

async function initDb() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      name TEXT NOT NULL,
      username TEXT UNIQUE NOT NULL,
      password_hash TEXT NOT NULL,
      role TEXT NOT NULL CHECK (
        role IN ('admin','attendant','tripeiro')
      ),
      ramal TEXT UNIQUE,
      pix_key TEXT,
      active BOOLEAN NOT NULL DEFAULT TRUE,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS production (
      id SERIAL PRIMARY KEY,
      employee_id INTEGER NOT NULL REFERENCES users(id),
      work_date DATE NOT NULL,
      amount NUMERIC(12,2) NOT NULL CHECK (amount >= 0),
      line_status TEXT NOT NULL CHECK (
        line_status IN ('first','both','tripeiro')
      ),
      employee_rate NUMERIC(5,2) NOT NULL,
      employee_commission NUMERIC(12,2) NOT NULL,
      company_rate NUMERIC(5,2) NOT NULL DEFAULT 50,
      company_commission NUMERIC(12,2) NOT NULL,
      note TEXT,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS withdrawals (
      id SERIAL PRIMARY KEY,
      employee_id INTEGER NOT NULL REFERENCES users(id),
      amount NUMERIC(12,2) NOT NULL CHECK (amount > 0),
      pix_key TEXT NOT NULL,
      status TEXT NOT NULL DEFAULT 'requested'
        CHECK (status IN ('requested','paid','rejected')),
      requested_at TIMESTAMPTZ DEFAULT NOW(),
      paid_at TIMESTAMPTZ,
      admin_note TEXT
    );
  `);

  const adminUser = process.env.ADMIN_USER || "admin";
  const adminPass = process.env.ADMIN_PASSWORD;

  if (adminPass) {
    const exists = await pool.query(
      "SELECT id FROM users WHERE username=$1",
      [adminUser]
    );

    if (!exists.rowCount) {
      const hash = await bcrypt.hash(adminPass, 12);

      await pool.query(
        `INSERT INTO users
        (name,username,password_hash,role)
        VALUES ($1,$2,$3,'admin')`,
        ["Administrador", adminUser, hash]
      );
    }
  }
}

function page(title, body, user) {
  return `<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>${esc(title)} | TROOPA DO GALO</title>

<style>
:root {
  --yellow: #ffd400;
  --gold: #e7b900;
  --ink: #171717;
  --paper: #fffdf2;
  --muted: #686868;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--yellow);
  font-family: Arial, Helvetica, sans-serif;
  color: var(--ink);
}

header {
  background: #171717;
  color: #ffd400;
  padding: 17px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

header strong {
  font-size: 20px;
  letter-spacing: 1px;
}

header a {
  color: #ffd400;
  text-decoration: none;
  font-weight: bold;
}

main {
  max-width: 1100px;
  margin: 24px auto;
  padding: 0 14px 40px;
}

.card {
  background: var(--paper);
  border: 2px solid #171717;
  border-radius: 15px;
  padding: 18px;
  margin: 14px 0;
  box-shadow: 0 5px 0 rgba(0,0,0,.12);
}

h1,h2,h3 { margin: 0 0 14px; }
h1 { font-size: 25px; }
h2 { font-size: 19px; }

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit,minmax(210px,1fr));
  gap: 12px;
}

label {
  display: block;
  font-size: 13px;
  font-weight: bold;
  margin: 10px 0 5px;
}

input,select,textarea {
  width: 100%;
  padding: 12px;
  border: 1.5px solid #b8a64d;
  border-radius: 9px;
  font-size: 16px;
  background: #fff;
}

button,.btn {
  display: inline-block;
  border: 0;
  border-radius: 9px;
  background: #171717;
  color: #ffd400;
  padding: 12px 16px;
  font-weight: bold;
  font-size: 15px;
  text-decoration: none;
  cursor: pointer;
  margin-top: 12px;
}

table {
  width: 100%;
  border-collapse: collapse;
  background: white;
}

th,td {
  text-align: left;
  padding: 10px;
  border-bottom: 1px solid #eee;
  font-size: 14px;
}

th {
  background: #171717;
  color: #ffd400;
}

.table-wrap { overflow-x: auto; }
.muted { color: var(--muted); font-size: 13px; }
.stat { font-size: 25px; font-weight: bold; }

.pill {
  display: inline-block;
  padding: 5px 9px;
  border-radius: 30px;
  background: #ffe56a;
  font-size: 12px;
  font-weight: bold;
}

.error {
  background: #ffe0e0;
  padding: 10px;
  border-radius: 8px;
  color: #8a0000;
}

.login {
  max-width: 430px;
  margin: 8vh auto;
}

.inline { display: inline; }

@media(max-width:600px) {
  header strong { font-size: 17px; }
  .card { padding: 14px; }
  th,td { font-size: 12px; padding: 8px; }
}
</style>
</head>

<body>
<header>
<strong>🐓 TROOPA DO GALO</strong>
<span>
${user ? `Olá, ${esc(user.name)} · <a href="/sair">Sair</a>` : "Produção e pagamentos"}
</span>
</header>
${body}
</body>
</html>`;
}

function requireLogin(req,res,next) {
  if (!req.session.user) return res.redirect("/");
  next();
}

function requireAdmin(req,res,next) {
  if (!req.session.user || req.session.user.role !== "admin") {
    return res.status(403).send(
      page("Acesso negado",
      '<main><div class="card"><h1>Acesso restrito</h1><a class="btn" href="/">Voltar</a></div></main>',
      req.session.user)
    );
  }
  next();
}

function fmtDate(d) {
  return new Date(d).toLocaleDateString("pt-BR", {
    timeZone: "UTC"
  });
}

app.get("/", async (req,res) => {
  if (req.session.user) return res.redirect("/painel");

  res.send(page("Entrar", `
  <main class="login">
  <div class="card">
  <h1>Entrar no sistema</h1>
  <p class="muted">Acesse com seu usuário e senha.</p>
  ${req.query.erro ? '<p class="error">Usuário ou senha inválidos.</p>' : ''}

  <form method="POST" action="/login">
  <label>Usuário</label>
  <input name="username" required autocomplete="username">

  <label>Senha</label>
  <input type="password" name="password" required autocomplete="current-password">

  <button type="submit">Entrar</button>
  </form>
  </div>
  </main>
  `));
});

app.post("/login", async (req,res) => {
  const {username,password} = req.body;

  const result = await pool.query(
    "SELECT * FROM users WHERE username=$1 AND active=TRUE",
    [username]
  );

  if (!result.rowCount ||
      !(await bcrypt.compare(password,result.rows[0].password_hash))) {
    return res.redirect("/?erro=1");
  }

  req.session.user = {
    id: result.rows[0].id,
    name: result.rows[0].name,
    username: result.rows[0].username,
    role: result.rows[0].role,
    ramal: result.rows[0].ramal
  };

  res.redirect("/painel");
});

app.get("/sair",(req,res) => {
  req.session.destroy(() => res.redirect("/"));
});

app.get("/painel",requireLogin,async(req,res) => {
  const u = req.session.user;

  if (u.role === "admin") {
    const [people,prod,withdraw] = await Promise.all([
      pool.query(
        "SELECT id,name,username,role,ramal,active FROM users WHERE role<>'admin' ORDER BY name"
      ),
      pool.query(`
        SELECT p.*,u.name,u.ramal
        FROM production p
        JOIN users u ON u.id=p.employee_id
        ORDER BY p.work_date DESC,p.id DESC
        LIMIT 30
      `),
      pool.query(`
        SELECT w.*,u.name,u.ramal
        FROM withdrawals w
        JOIN users u ON u.id=w.employee_id
        ORDER BY w.requested_at DESC
        LIMIT 30
      `)
    ]);

    const totals = await pool.query(`
      SELECT
      COALESCE(SUM(employee_commission),0) staff,
      COALESCE(SUM(company_commission),0) company
      FROM production
    `);

    const body = `
    <main>
    <h1>Painel administrativo</h1>

    <div class="grid">
      <div class="card">
        <div class="muted">Comissões dos funcionários</div>
        <div class="stat">${money(totals.rows[0].staff)}</div>
      </div>

      <div class="card">
        <div class="muted">Comissão da empresa</div>
        <div class="stat">${money(totals.rows[0].company)}</div>
      </div>

      <div class="card">
        <div class="muted">Funcionários cadastrados</div>
        <div class="stat">${people.rowCount}</div>
      </div>
    </div>

    <div class="card">
    <h2>Cadastrar funcionário</h2>

    <form method="POST" action="/admin/funcionarios">
    <div class="grid">

    <div>
    <label>Nome</label>
    <input name="name" required>
    </div>

    <div>
    <label>Usuário de acesso</label>
    <input name="username" required>
    </div>

    <div>
    <label>Senha inicial</label>
    <input name="password" type="password" minlength="8" required>
    </div>

    <div>
    <label>Ramal único</label>
    <input name="ramal" required>
    </div>

    <div>
    <label>Função</label>
    <select name="role">
    <option value="attendant">Atendente</option>
    <option value="tripeiro">Tripeiro</option>
    </select>
    </div>

    <div>
    <label>Chave PIX</label>
    <input name="pix_key">
    </div>

    </div>
    <button>Cadastrar funcionário</button>
    </form>
    </div>

    <div class="card">
    <h2>Lançar produção</h2>

    <form method="POST" action="/admin/producao">
    <div class="grid">

    <div>
    <label>Funcionário / ramal</label>
    <select name="employee_id" required>
    ${people.rows.filter(p=>p.active).map(p=>`
      <option value="${p.id}">
      ${esc(p.name)} · ${esc(p.ramal || "sem ramal")} ·
      ${p.role==="tripeiro"?"Tripeiro":"Atendente"}
      </option>
    `).join("")}
    </select>
    </div>

    <div>
    <label>Data</label>
    <input type="date" name="work_date" value="${today()}" required>
    </div>

    <div>
    <label>Valor produzido (R$)</label>
    <input type="number" name="amount" min="0.01" step="0.01" required>
    </div>

    <div>
    <label>Resultado</label>
    <select name="line_status">
    <option value="first">Somente primeira linha (25%)</option>
    <option value="both">Primeira e segunda linhas (50%)</option>
    <option value="tripeiro">Produção de tripeiro (30%)</option>
    </select>
    </div>

    <div>
    <label>Observação</label>
    <input name="note">
    </div>

    </div>

    <p class="muted">
    Atendente: 25% ou 50%. Tripeiro: 30%.
    Comissão da empresa: 50%, registrada separadamente.
    </p>

    <button>Salvar produção</button>
    </form>
    </div>

    <div class="card">
    <h2>Funcionários cadastrados</h2>
    <div class="table-wrap">
    <table>
    <tr>
    <th>Nome</th><th>Ramal</th><th>Função</th><th>Usuário</th><th>Status</th>
    </tr>

    ${people.rows.map(p=>`
    <tr>
    <td>${esc(p.name)}</td>
    <td>${esc(p.ramal || "-")}</td>
    <td>${p.role==="tripeiro"?"Tripeiro":"Atendente"}</td>
    <td>${esc(p.username)}</td>
    <td>${p.active?"Ativo":"Inativo"}</td>
    </tr>
    `).join("")}

    </table>
    </div>
    </div>

    <div class="card">
    <h2>Produção recente</h2>
    <div class="table-wrap">
    <table>
    <tr>
    <th>Data</th><th>Funcionário</th><th>Ramal</th>
    <th>Produção</th><th>Comissão funcionário</th><th>Empresa</th>
    </tr>

    ${prod.rows.map(p=>`
    <tr>
    <td>${fmtDate(p.work_date)}</td>
    <td>${esc(p.name)}</td>
    <td>${esc(p.ramal || "-")}</td>
    <td>${money(p.amount)}</td>
    <td>${money(p.employee_commission)}</td>
    <td>${money(p.company_commission)}</td>
    </tr>
    `).join("")}

    </table>
    </div>
    </div>

    <div class="card">
    <h2>Solicitações de saque</h2>
    <div class="table-wrap">
    <table>
    <tr>
    <th>Data</th><th>Funcionário</th><th>Ramal</th>
    <th>Valor</th><th>PIX</th><th>Status</th><th>Ação</th>
    </tr>

    ${withdraw.rows.map(w=>`
    <tr>
    <td>${new Date(w.requested_at).toLocaleString("pt-BR")}</td>
    <td>${esc(w.name)}</td>
    <td>${esc(w.ramal || "-")}</td>
    <td>${money(w.amount)}</td>
    <td>${esc(w.pix_key)}</td>
    <td>${esc(w.status)}</td>
    <td>
    ${w.status==="requested" ? `
      <form class="inline" method="POST" action="/admin/saque/${w.id}/pago">
      <button>Marcar pago</button>
      </form>
    ` : "-"}
    </td>
    </tr>
    `).join("")}

    </table>
    </div>

    <p class="muted">
    O pagamento PIX é feito pelo administrador no banco.
    Depois, marque o saque como pago.
    </p>
    </div>

    </main>
    `;

    return res.send(page("Painel administrativo",body,u));
  }

  const [prod,withdraw,balance] = await Promise.all([
    pool.query(
      "SELECT * FROM production WHERE employee_id=$1 ORDER BY work_date DESC,id DESC LIMIT 60",
      [u.id]
    ),
    pool.query(
      "SELECT * FROM withdrawals WHERE employee_id=$1 ORDER BY requested_at DESC LIMIT 20",
      [u.id]
    ),
    pool.query(`
      SELECT
      COALESCE(
        (SELECT SUM(employee_commission) FROM production WHERE employee_id=$1),0
      ) -
      COALESCE(
        (SELECT SUM(amount) FROM withdrawals
        WHERE employee_id=$1 AND status IN ('requested','paid')),0
      ) AS available
    `,[u.id])
  ]);

  const total = prod.rows.reduce(
    (s,p)=>s+Number(p.employee_commission),0
  );

  const pix = await pool.query(
    "SELECT pix_key FROM users WHERE id=$1",
    [u.id]
  );

  const body = `
  <main>
  <h1>Meu aplicativo</h1>

  <div class="card">
  <h2>${esc(u.name)} · Ramal ${esc(u.ramal || "-")}</h2>
  <p>${u.role==="tripeiro"?"Tripeiro":"Atendente"}</p>

  <span class="pill">
  Saldo disponível: ${money(balance.rows[0].available)}
  </span>

  <div class="grid" style="margin-top:14px">
  <div>
  <div class="muted">Comissões lançadas</div>
  <div class="stat">${money(total)}</div>
  </div>

  <div>
  <div class="muted">Produções registradas</div>
  <div class="stat">${prod.rowCount}</div>
  </div>
  </div>
  </div>

  <div class="card">
  <h2>Solicitar saque</h2>

  <form method="POST" action="/saque">
  <label>Valor (R$)</label>
  <input
    type="number"
    name="amount"
    min="0.01"
    step="0.01"
    max="${Math.max(0,Number(balance.rows[0].available))}"
    required
  >

  <label>Sua chave PIX</label>
  <input name="pix_key" value="${esc(pix.rows[0].pix_key || "")}" required>

  <button>Solicitar saque</button>
  </form>

  <p class="muted">
  O saque será enviado para análise do administrador.
  </p>
  </div>

  <div class="card">
  <h2>Meu extrato de produção</h2>
  <div class="table-wrap">
  <table>
  <tr><th>Data</th><th>Produção</th><th>Comissão</th><th>Observação</th></tr>

  ${prod.rows.map(p=>`
  <tr>
  <td>${fmtDate(p.work_date)}</td>
  <td>${money(p.amount)}</td>
  <td>${money(p.employee_commission)}</td>
  <td>${esc(p.note || "-")}</td>
  </tr>
  `).join("")}

  </table>
  </div>
  </div>

  <div class="card">
  <h2>Meus saques</h2>
  <div class="table-wrap">
  <table>
  <tr><th>Data</th><th>Valor</th><th>Status</th></tr>

  ${withdraw.rows.map(w=>`
  <tr>
  <td>${new Date(w.requested_at).toLocaleString("pt-BR")}</td>
  <td>${money(w.amount)}</td>
  <td>${esc(w.status)}</td>
  </tr>
  `).join("")}

  </table>
  </div>
  </div>

  </main>
  `;

  res.send(page("Meu aplicativo",body,u));
});

app.post("/admin/funcionarios",requireAdmin,async(req,res)=>{
  const {name,username,password,ramal,role,pix_key} = req.body;

  if (!["attendant","tripeiro"].includes(role)) {
    return res.status(400).send("Função inválida.");
  }

  try {
    const hash = await bcrypt.hash(password,12);

    await pool.query(
      `INSERT INTO users
      (name,username,password_hash,role,ramal,pix_key)
      VALUES($1,$2,$3,$4,$5,$6)`,
      [name,username,hash,role,ramal,pix_key || null]
    );

    res.redirect("/painel");

  } catch(e) {
    res.status(400).send(
      page("Erro",
      '<main><div class="card"><h2>Não foi possível cadastrar</h2><p>Verifique se o usuário ou ramal já existe.</p><a class="btn" href="/painel">Voltar</a></div></main>',
      req.session.user)
    );
  }
});

app.post("/admin/producao",requireAdmin,async(req,res)=>{
  const {employee_id,work_date,amount,line_status,note} = req.body;
  const value = Number(amount);

  if (!Number.isFinite(value) || value <= 0) {
    return res.status(400).send("Valor inválido.");
  }

  const result = await pool.query(
    "SELECT role FROM users WHERE id=$1 AND active=TRUE",
    [employee_id]
  );

  if (!result.rowCount) {
    return res.status(400).send("Funcionário não encontrado.");
  }

  const role = result.rows[0].role;
  let rate;

  if (role === "tripeiro") {
    rate = 30;
  } else if (line_status === "first") {
    rate = 25;
  } else if (line_status === "both") {
    rate = 50;
  } else {
    return res.status(400).send("Resultado incompatível com a função.");
  }

  const employeeCommission = Number(
    (value * rate / 100).toFixed(2)
  );

  const companyCommission = Number(
    (value * 0.5).toFixed(2)
  );

  await pool.query(`
    INSERT INTO production(
      employee_id,work_date,amount,line_status,
      employee_rate,employee_commission,
      company_rate,company_commission,note
    )
    VALUES($1,$2,$3,$4,$5,$6,50,$7,$8)
  `,[
    employee_id,
    work_date,
    value,
    role === "tripeiro" ? "tripeiro" : line_status,
    rate,
    employeeCommission,
    companyCommission,
    note || null
  ]);

  res.redirect("/painel");
});

app.post("/saque",requireLogin,async(req,res)=>{
  if (req.session.user.role === "admin") {
    return res.status(403).send("Administradores não solicitam saque.");
  }

  const amount = Number(req.body.amount);
  const pix = String(req.body.pix_key || "").trim();

  if (!Number.isFinite(amount) || amount <= 0 || !pix) {
    return res.status(400).send("Dados inválidos.");
  }

  const client = await pool.connect();

  try {
    await client.query("BEGIN");

    const b = await client.query(`
      SELECT
      COALESCE(
        (SELECT SUM(employee_commission) FROM production WHERE employee_id=$1),0
      ) -
      COALESCE(
        (SELECT SUM(amount) FROM withdrawals
        WHERE employee_id=$1 AND status IN ('requested','paid')),0
      ) AS available
    `,[req.session.user.id]);

    if (amount > Number(b.rows[0].available)) {
      await client.query("ROLLBACK");
      return res.status(400).send("Saldo insuficiente.");
    }

    await client.query(
      "INSERT INTO withdrawals(employee_id,amount,pix_key) VALUES($1,$2,$3)",
      [req.session.user.id,amount,pix]
    );

    await client.query(
      "UPDATE users SET pix_key=$1 WHERE id=$2",
      [pix,req.session.user.id]
    );

    await client.query("COMMIT");
    res.redirect("/painel");

  } catch(e) {
    await client.query("ROLLBACK");
    res.status(500).send("Erro ao solicitar saque.");
  } finally {
    client.release();
  }
});

app.post("/admin/saque/:id/pago",requireAdmin,async(req,res)=>{
  await pool.query(
    `UPDATE withdrawals
     SET status='paid',paid_at=NOW()
     WHERE id=$1 AND status='requested'`,
    [req.params.id]
  );

  res.redirect("/painel");
});

app.get("/health",(req,res)=>{
  res.json({
    ok: true,
    app: "TROOPA DO GALO"
  });
});

initDb().then(()=>{
  app.listen(PORT,()=>{
    console.log(`TROOPA DO GALO rodando na porta ${PORT}`);
  });
}).catch(err=>{
  console.error("Falha ao iniciar banco de dados:",err);
  process.exit(1);
});
