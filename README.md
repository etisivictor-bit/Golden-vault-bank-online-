
// node.js - Golden Vault demo bank (self-contained)

const { execSync } = require("child_process");

// Auto-install dependencies if not found
function ensureDeps() {
  const deps = ["express", "body-parser", "express-session"];
  deps.forEach(dep => {
    try { require.resolve(dep); }
    catch(e) { console.log(`Installing ${dep}...`); execSync(`npm install ${dep}`); }
  });
}
ensureDeps();

const express = require("express");
const bodyParser = require("body-parser");
const session = require("express-session");

const app = express();
app.use(bodyParser.urlencoded({ extended: true }));
app.use(session({ secret: "goldenvault_secret_key", resave: false, saveUninitialized: true }));

// In-memory storage
let users = [];
let transactions = [];
let admin = { email: "admin@gv.com", password: "admin123" };

// Reusable page template
function renderPage(title, body, req) {
  return `
  <!DOCTYPE html>
  <html>
  <head>
    <title>${title} - Golden Vault</title>
    <style>
      body { font-family: Arial, sans-serif; background:#f4f7fb; margin:0; padding:0; }
      header { background:#1a237e; color:#fff; padding:15px; text-align:center; }
      nav a { color:#fff; margin:0 10px; text-decoration:none; }
      main { max-width:800px; margin:20px auto; padding:20px; background:#fff; border-radius:8px; box-shadow:0 0 10px rgba(0,0,0,0.1); }
      h1,h2 { color:#1a237e; }
      .error { color:red; }
      table { width:100%; border-collapse:collapse; margin-top:10px; }
      th,td { border:1px solid #ddd; padding:8px; text-align:left; }
      footer { text-align:center; padding:10px; color:#555; margin-top:20px; }
      button { background:#1a237e; color:#fff; border:none; padding:8px 14px; border-radius:4px; cursor:pointer; }
      input { padding:6px; margin:4px 0; width:95%; }
    </style>
  </head>
  <body>
    <header>
      <h1>Golden Vault</h1>
      <nav>
        <a href="/">Home</a>
        ${req.session.user ? `
          <a href="/dashboard">Dashboard</a>
          <a href="/deposit">Deposit</a>
          <a href="/withdraw">Withdraw</a>
          <a href="/logout">Logout</a>
        ` : `
          <a href="/signup">Sign Up</a>
          <a href="/login">Login</a>
        `}
        ${req.session.admin ? `
          <a href="/admin">Admin Panel</a>
          <a href="/admin/logout">Logout</a>
        ` : `
          <a href="/admin/login">Admin</a>
        `}
      </nav>
    </header>
    <main>${body}</main>
    <footer>Golden Vault Demo &copy; 2025</footer>
  </body>
  </html>`;
}

// === Routes ===
app.get("/", (req,res)=>res.send(renderPage("Home", "<h2>Welcome to Golden Vault</h2><p>Sign up or login to start.</p>", req)));

// Signup
app.get("/signup",(req,res)=>res.send(renderPage("Signup",`
  <h2>Sign Up</h2>
  <form method="post">
    <input name="email" type="email" placeholder="Email" required><br>
    <input name="password" type="password" placeholder="Password" required><br>
    <button>Sign Up</button>
  </form>`,req)));
app.post("/signup",(req,res)=>{
  const {email,password}=req.body;
  if(users.find(u=>u.email===email)) return res.send(renderPage("Signup","<p class='error'>Email exists</p>",req));
  const vat=Math.random().toString(36).substring(2,8).toUpperCase();
  users.push({email,password,balance:0,vat});
  res.redirect("/login");
});

// Login
app.get("/login",(req,res)=>res.send(renderPage("Login",`
  <h2>Login</h2>
  <form method="post">
    <input name="email" type="email"><br>
    <input name="password" type="password"><br>
    <button>Login</button>
  </form>`,req)));
app.post("/login",(req,res)=>{
  const u=users.find(x=>x.email===req.body.email&&x.password===req.body.password);
  if(!u) return res.send(renderPage("Login","<p class='error'>Invalid</p>",req));
  req.session.user=u; res.redirect("/dashboard");
});
app.get("/logout",(req,res)=>req.session.destroy(()=>res.redirect("/")));

// Dashboard
app.get("/dashboard",(req,res)=>{
  if(!req.session.user) return res.redirect("/login");
  const u=req.session.user;
  let tx=transactions.filter(t=>t.email===u.email).map(t=>`<tr><td>${t.type}</td><td>${t.amount}</td><td>${t.date}</td></tr>`).join("");
  res.send(renderPage("Dashboard",`
    <h2>Dashboard</h2>
    <p>Email: ${u.email}</p>
    <p>Balance: â¦ ${u.balance}</p>
    <p>Your VAT Code: <b>${u.vat}</b></p>
    <h3>Transactions</h3>
    <table><tr><th>Type</th><th>Amount</th><th>Date</th></tr>${tx}</table>`,req));
});

// Deposit
app.get("/deposit",(req,res)=>req.session.user?res.send(renderPage("Deposit",`
  <h2>Deposit</h2>
  <form method="post"><input name="amount" type="number"><br><button>Deposit</button></form>`,req)):res.redirect("/login"));
app.post("/deposit",(req,res)=>{
  if(!req.session.user) return res.redirect("/login");
  let amt=+req.body.amount; req.session.user.balance+=amt;
  transactions.push({email:req.session.user.email,type:"Deposit",amount:amt,date:new Date().toLocaleString()});
  res.redirect("/dashboard");
});

// Withdraw (5-step flow)
app.get("/withdraw",(req,res)=>{ if(!req.session.user)return res.redirect("/login"); req.session.flow={step:1}; res.redirect("/withdraw/step"); });
app.get("/withdraw/step",(req,res)=>{
  let f=req.session.flow||{step:1}; let form="";
  if(f.step===1) form=`<h2>Step 1: Amount</h2><form method="post"><input name="amount"><button>Next</button></form>`;
  else if(f.step===2) form=`<h2>Step 2: VAT</h2><form method="post"><input name="vat"><button>Next</button></form>`;
  else if(f.step===3) form=`<h2>Step 3: Password</h2><form method="post"><input type="password" name="password"><button>Next</button></form>`;
  else if(f.step===4){ let otp=Math.floor(100000+Math.random()*900000); f.otp=otp; req.session.flow=f;
    form=`<h2>Step 4: OTP</h2><p>Demo OTP: <b>${otp}</b></p><form method="post"><input name="otp"><button>Next</button></form>`; }
  else form=`<h2>Step 5: Confirm</h2><p>Withdraw â¦${f.amount}</p><form method="post"><button>Complete</button></form>`;
  res.send(renderPage("Withdraw",form,req));
});
app.post("/withdraw/step",(req,res)=>{
  let f=req.session.flow; let u=req.session.user;
  if(f.step===1){let a=+req.body.amount; if(a<=0||a>u.balance)return res.send(renderPage("Withdraw","<p class='error'>Bad amount</p>",req)); f.amount=a; f.step=2;}
  else if(f.step===2){ if(req.body.vat!==u.vat)return res.send(renderPage("Withdraw","<p class='error'>Bad VAT</p>",req)); f.step=3;}
  else if(f.step===3){ if(req.body.password!==u.password)return res.send(renderPage("Withdraw","<p class='error'>Bad pass</p>",req)); f.step=4;}
  else if(f.step===4){ if(req.body.otp!=f.otp)return res.send(renderPage("Withdraw","<p class='error'>Bad OTP</p>",req)); f.step=5;}
  else { u.balance-=f.amount; transactions.push({email:u.email,type:"Withdraw",amount:f.amount,date:new Date().toLocaleString()}); req.session.flow=null; return res.redirect("/dashboard"); }
  req.session.flow=f; res.redirect("/withdraw/step");
});

// Admin
app.get("/admin/login",(req,res)=>res.send(renderPage("Admin Login",`
  <h2>Admin Login</h2>
  <form method="post"><input name="email"><br><input name="password" type="password"><br><button>Login</button></form>`,req)));
app.post("/admin/login",(req,res)=>{ if(req.body.email===admin.email&&req.body.password===admin.password){req.session.admin=admin;res.redirect("/admin");}else res.send(renderPage("Admin Login","<p class='error'>Invalid</p>",req)); });
app.get("/admin",(req,res)=>{
  if(!req.session.admin)return res.redirect("/admin/login");
  let rows=users.map(u=>`<tr><td>${u.email}</td><td>${u.balance}</td><td>${u.vat}</td></tr>`).join("");
  res.send(renderPage("Admin Panel",`<h2>Users</h2><table><tr><th>Email</th><th>Balance</th><th>VAT</th></tr>${rows}</table>`,req));
});
app.get("/admin/logout",(req,res)=>{req.session.admin=null;res.redirect("/");});

// Start
const PORT=process.env.PORT||3000;
app.listen(PORT,()=>console.log("Golden Vault running on "+PORT));
