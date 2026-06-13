// ╔══════════════════════════════════════════════════════════════════╗
// ║  DocFlow Platform — Multi-role SaaS with TDS + FIRC Extractors  ║
// ║  Roles: Super Admin · Admin · Member                             ║
// ╚══════════════════════════════════════════════════════════════════╝

import { useState, useEffect, useRef, useCallback, createContext, useContext } from "react";
import * as XLSX from "xlsx";

// ─── PERSISTENT STORAGE HELPERS ───────────────────────────────────────────────
const S = {
  async get(key) {
    try { const r = await window.storage.get(key, true); return r ? JSON.parse(r.value) : null; }
    catch { return null; }
  },
  async set(key, val) {
    try { await window.storage.set(key, JSON.stringify(val), true); return true; }
    catch { return false; }
  },
  async del(key) {
    try { await window.storage.delete(key, true); return true; }
    catch { return false; }
  },
  async list(prefix) {
    try { const r = await window.storage.list(prefix, true); return r?.keys || []; }
    catch { return []; }
  }
};

// ─── CRYPTO HELPERS ───────────────────────────────────────────────────────────
function hashPw(pw) {
  // Simple deterministic hash for demo (in production use bcrypt)
  let h = 0;
  for (let i = 0; i < pw.length; i++) {
    h = ((h << 5) - h) + pw.charCodeAt(i);
    h |= 0;
  }
  return "hsh_" + Math.abs(h).toString(36) + pw.length.toString(36);
}
function uid() { return Date.now().toString(36) + Math.random().toString(36).slice(2, 7); }
function resetToken() { return Math.random().toString(36).slice(2) + Math.random().toString(36).slice(2); }

// ─── DEFAULT DATA ─────────────────────────────────────────────────────────────
const DEFAULT_SUPER = {
  id: "superadmin",
  role: "superadmin",
  name: "Super Admin",
  username: "superadmin",
  email: "superadmin@docflow.app",
  passwordHash: hashPw("Admin@123"),
  createdAt: new Date().toISOString(),
  status: "active"
};

const DEFAULT_APPS = [
  {
    id: "tds-extractor",
    name: "TDS Extractor",
    icon: "🧾",
    description: "Extract TAN, Challan No, Assessment Year and tax fields from TDS documents",
    color: "#3b82f6",
    route: "tds",
    createdAt: new Date().toISOString(),
    status: "active"
  },
  {
    id: "firc-extractor",
    name: "FIRC Extractor",
    icon: "💱",
    description: "Extract inward remittance data, FCY/INR amounts and remitter details from FIRC documents",
    color: "#8b5cf6",
    route: "firc",
    createdAt: new Date().toISOString(),
    status: "active"
  }
];

// ─── DB LAYER ─────────────────────────────────────────────────────────────────
const DB = {
  async init() {
    const existing = await S.get("platform:init");
    if (!existing) {
      await S.set("user:superadmin", DEFAULT_SUPER);
      await S.set("platform:apps", DEFAULT_APPS);
      await S.set("platform:init", { version: 1, at: new Date().toISOString() });
    }
  },
  async getUser(username) {
    // Check superadmin first
    const sa = await S.get("user:superadmin");
    if (sa?.username === username) return sa;
    // Check all users
    const keys = await S.list("user:");
    for (const k of keys) {
      const u = await S.get(k);
      if (u?.username === username) return u;
    }
    return null;
  },
  async getUserById(id) { return await S.get("user:" + id); },
  async saveUser(user) { return await S.set("user:" + user.id, user); },
  async deleteUser(id) { return await S.del("user:" + id); },
  async getAllUsers() {
    const keys = await S.list("user:");
    const users = [];
    for (const k of keys) {
      const u = await S.get(k);
      if (u) users.push(u);
    }
    return users;
  },
  async getApps() { return (await S.get("platform:apps")) || DEFAULT_APPS; },
  async saveApps(apps) { return await S.set("platform:apps", apps); },
  async saveResetToken(username, token) {
    return await S.set("reset:" + username, { token, expires: Date.now() + 3600000 });
  },
  async getResetToken(username) { return await S.get("reset:" + username); },
  async delResetToken(username) { return await S.del("reset:" + username); }
};

// ─── AUTH CONTEXT ─────────────────────────────────────────────────────────────
const AuthCtx = createContext(null);
function useAuth() { return useContext(AuthCtx); }

// ─── TOAST ────────────────────────────────────────────────────────────────────
function Toast({ toasts, remove }) {
  return (
    <div style={{ position: "fixed", top: 20, right: 20, zIndex: 9999, display: "flex", flexDirection: "column", gap: 8 }}>
      {toasts.map(t => (
        <div key={t.id} style={{
          background: t.type === "error" ? "#450a0a" : t.type === "success" ? "#052e16" : "#1e293b",
          border: `1px solid ${t.type === "error" ? "#7f1d1d" : t.type === "success" ? "#14532d" : "#334155"}`,
          color: t.type === "error" ? "#fca5a5" : t.type === "success" ? "#86efac" : "#e2e8f0",
          padding: "10px 16px", borderRadius: 10, fontSize: 13,
          fontFamily: "'DM Sans', sans-serif", maxWidth: 320,
          boxShadow: "0 8px 24px rgba(0,0,0,0.4)",
          display: "flex", alignItems: "center", gap: 10,
          animation: "slideIn 0.2s ease"
        }}>
          <span>{t.type === "error" ? "✕" : t.type === "success" ? "✓" : "ℹ"}</span>
          <span style={{ flex: 1 }}>{t.msg}</span>
          <button onClick={() => remove(t.id)} style={{ background: "none", border: "none", color: "inherit", cursor: "pointer", opacity: 0.6, fontSize: 16 }}>×</button>
        </div>
      ))}
    </div>
  );
}

function useToast() {
  const [toasts, setToasts] = useState([]);
  const add = useCallback((msg, type = "info") => {
    const id = uid();
    setToasts(p => [...p, { id, msg, type }]);
    setTimeout(() => setToasts(p => p.filter(t => t.id !== id)), 4000);
  }, []);
  const remove = useCallback(id => setToasts(p => p.filter(t => t.id !== id)), []);
  return { toasts, toast: add, removeToast: remove };
}

// ─── MODAL ────────────────────────────────────────────────────────────────────
function Modal({ title, onClose, children, width = 460 }) {
  return (
    <div style={{
      position: "fixed", inset: 0, background: "rgba(2,8,23,0.85)",
      display: "flex", alignItems: "center", justifyContent: "center",
      zIndex: 1000, backdropFilter: "blur(4px)", padding: 16
    }} onClick={e => e.target === e.currentTarget && onClose()}>
      <div style={{
        background: "#0d1829", border: "1px solid #1e293b",
        borderRadius: 16, width: "100%", maxWidth: width,
        boxShadow: "0 24px 80px rgba(0,0,0,0.6)"
      }}>
        <div style={{
          padding: "18px 24px", borderBottom: "1px solid #1e293b",
          display: "flex", alignItems: "center", justifyContent: "space-between"
        }}>
          <h3 style={{ fontFamily: "'Syne', sans-serif", fontWeight: 700, fontSize: 16, color: "#f1f5f9" }}>{title}</h3>
          <button onClick={onClose} style={{
            background: "#1e293b", border: "none", color: "#94a3b8",
            width: 28, height: 28, borderRadius: 6, cursor: "pointer", fontSize: 16
          }}>×</button>
        </div>
        <div style={{ padding: 24 }}>{children}</div>
      </div>
    </div>
  );
}

// ─── FORM COMPONENTS ──────────────────────────────────────────────────────────
function Field({ label, children, hint }) {
  return (
    <div style={{ marginBottom: 16 }}>
      <label style={{ display: "block", fontSize: 12, fontWeight: 600, color: "#94a3b8", marginBottom: 6, letterSpacing: "0.06em", textTransform: "uppercase" }}>
        {label}
      </label>
      {children}
      {hint && <div style={{ fontSize: 11, color: "#475569", marginTop: 4 }}>{hint}</div>}
    </div>
  );
}
function Input({ ...props }) {
  return (
    <input {...props} style={{
      width: "100%", background: "#0f172a", border: "1px solid #1e293b",
      borderRadius: 8, padding: "9px 12px", color: "#f1f5f9",
      fontFamily: "'DM Sans', sans-serif", fontSize: 14, outline: "none",
      transition: "border-color 0.2s",
      ...(props.style || {})
    }}
      onFocus={e => e.target.style.borderColor = "#3b82f6"}
      onBlur={e => e.target.style.borderColor = "#1e293b"}
    />
  );
}
function Btn({ children, variant = "primary", onClick, disabled, full, small, style: extra = {} }) {
  const variants = {
    primary: { background: "linear-gradient(135deg, #3b82f6, #2563eb)", color: "#fff", border: "none" },
    danger: { background: "#450a0a", color: "#fca5a5", border: "1px solid #7f1d1d" },
    ghost: { background: "transparent", color: "#94a3b8", border: "1px solid #1e293b" },
    success: { background: "#052e16", color: "#86efac", border: "1px solid #14532d" },
    purple: { background: "linear-gradient(135deg, #7c3aed, #6d28d9)", color: "#fff", border: "none" },
  };
  const v = variants[variant] || variants.primary;
  return (
    <button onClick={onClick} disabled={disabled} style={{
      ...v, padding: small ? "6px 14px" : "10px 20px",
      borderRadius: 8, fontFamily: "'Syne', sans-serif",
      fontWeight: 700, fontSize: small ? 12 : 14, cursor: disabled ? "not-allowed" : "pointer",
      opacity: disabled ? 0.5 : 1, width: full ? "100%" : "auto",
      transition: "opacity 0.2s, transform 0.1s", letterSpacing: "0.03em",
      ...extra
    }}>{children}</button>
  );
}

// ─── LOGIN PAGE ───────────────────────────────────────────────────────────────
function LoginPage({ onLogin, toast }) {
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [loading, setLoading] = useState(false);
  const [forgotMode, setForgotMode] = useState(false);
  const [resetMode, setResetMode] = useState(false);
  const [resetTokenInput, setResetTokenInput] = useState("");
  const [newPw, setNewPw] = useState("");
  const [forgotUser, setForgotUser] = useState("");

  const handleLogin = async () => {
    if (!username || !password) { toast("Enter username and password", "error"); return; }
    setLoading(true);
    const user = await DB.getUser(username.trim().toLowerCase());
    if (!user || user.passwordHash !== hashPw(password)) {
      toast("Invalid username or password", "error"); setLoading(false); return;
    }
    if (user.status === "disabled") {
      toast("Your account has been disabled. Contact admin.", "error"); setLoading(false); return;
    }
    toast(`Welcome back, ${user.name}!`, "success");
    onLogin(user);
    setLoading(false);
  };

  const handleForgot = async () => {
    if (!forgotUser) { toast("Enter your username", "error"); return; }
    const user = await DB.getUser(forgotUser.trim().toLowerCase());
    if (!user || user.role !== "superadmin") {
      toast("Forgot password is only available for Super Admin. Contact your admin.", "error"); return;
    }
    const token = resetToken();
    await DB.saveResetToken(user.username, token);
    // Simulate email
    toast(`Reset token: ${token.slice(0, 8).toUpperCase()} (check email: ${user.email})`, "success");
    setForgotMode(false); setResetMode(true);
  };

  const handleReset = async () => {
    if (!newPw || newPw.length < 6) { toast("Password must be at least 6 characters", "error"); return; }
    const stored = await DB.getResetToken(forgotUser.trim().toLowerCase());
    if (!stored || stored.token !== resetTokenInput || Date.now() > stored.expires) {
      toast("Invalid or expired reset token", "error"); return;
    }
    const user = await DB.getUser(forgotUser.trim().toLowerCase());
    user.passwordHash = hashPw(newPw);
    await DB.saveUser(user);
    await DB.delResetToken(user.username);
    toast("Password reset successfully! Please login.", "success");
    setResetMode(false); setForgotUser(""); setResetTokenInput(""); setNewPw("");
  };

  return (
    <div style={{
      minHeight: "100vh", background: "#020817",
      display: "flex", alignItems: "center", justifyContent: "center",
      padding: 16, position: "relative", overflow: "hidden"
    }}>
      {/* Background grid */}
      <div style={{
        position: "absolute", inset: 0, opacity: 0.04,
        backgroundImage: "linear-gradient(#3b82f6 1px, transparent 1px), linear-gradient(90deg, #3b82f6 1px, transparent 1px)",
        backgroundSize: "40px 40px"
      }} />
      <div style={{
        position: "absolute", top: "20%", left: "10%",
        width: 400, height: 400, borderRadius: "50%",
        background: "radial-gradient(circle, #1e3a5f44, transparent 70%)"
      }} />

      <div style={{
        background: "#0d1829", border: "1px solid #1e293b",
        borderRadius: 20, padding: 40, width: "100%", maxWidth: 420,
        boxShadow: "0 32px 100px rgba(0,0,0,0.5)",
        position: "relative"
      }}>
        {/* Logo */}
        <div style={{ textAlign: "center", marginBottom: 32 }}>
          <div style={{
            width: 56, height: 56, borderRadius: 16, margin: "0 auto 12px",
            background: "linear-gradient(135deg, #1e3a5f, #1e1b4b)",
            border: "1px solid #1e40af",
            display: "flex", alignItems: "center", justifyContent: "center",
            fontSize: 26
          }}>⚡</div>
          <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, fontWeight: 800, color: "#f1f5f9", letterSpacing: "-0.02em" }}>
            DocFlow Platform
          </h1>
          <p style={{ color: "#475569", fontSize: 13, marginTop: 4 }}>
            {forgotMode ? "Reset your password" : resetMode ? "Enter reset token" : "Sign in to your workspace"}
          </p>
        </div>

        {!forgotMode && !resetMode && (
          <>
            <Field label="Username">
              <Input placeholder="Enter username" value={username} onChange={e => setUsername(e.target.value)}
                onKeyDown={e => e.key === "Enter" && handleLogin()} />
            </Field>
            <Field label="Password">
              <Input type="password" placeholder="Enter password" value={password}
                onChange={e => setPassword(e.target.value)}
                onKeyDown={e => e.key === "Enter" && handleLogin()} />
            </Field>
            <Btn full onClick={handleLogin} disabled={loading}>
              {loading ? "Signing in…" : "Sign In →"}
            </Btn>
            <button onClick={() => setForgotMode(true)} style={{
              background: "none", border: "none", color: "#3b82f6",
              cursor: "pointer", fontSize: 13, marginTop: 16, width: "100%",
              fontFamily: "'DM Sans', sans-serif"
            }}>Forgot password? (Super Admin only)</button>
          </>
        )}

        {forgotMode && (
          <>
            <Field label="Your Username" hint="Only super admin can reset via this form">
              <Input placeholder="superadmin" value={forgotUser} onChange={e => setForgotUser(e.target.value)} />
            </Field>
            <div style={{ display: "flex", gap: 10 }}>
              <Btn onClick={handleForgot} full>Send Reset Token</Btn>
              <Btn variant="ghost" onClick={() => setForgotMode(false)}>Back</Btn>
            </div>
          </>
        )}

        {resetMode && (
          <>
            <Field label="Reset Token (from email)">
              <Input placeholder="Paste token here" value={resetTokenInput} onChange={e => setResetTokenInput(e.target.value)} />
            </Field>
            <Field label="New Password">
              <Input type="password" placeholder="Min 6 characters" value={newPw} onChange={e => setNewPw(e.target.value)} />
            </Field>
            <div style={{ display: "flex", gap: 10 }}>
              <Btn onClick={handleReset} full>Set New Password</Btn>
              <Btn variant="ghost" onClick={() => setResetMode(false)}>Back</Btn>
            </div>
          </>
        )}

        <div style={{ marginTop: 24, padding: "12px 16px", background: "#0f172a", borderRadius: 10, border: "1px solid #1e293b" }}>
          <div style={{ fontSize: 11, color: "#475569", marginBottom: 6 }}>Default credentials</div>
          <div style={{ fontSize: 12, color: "#64748b", fontFamily: "'DM Mono', monospace" }}>
            Username: <span style={{ color: "#94a3b8" }}>superadmin</span><br />
            Password: <span style={{ color: "#94a3b8" }}>Admin@123</span>
          </div>
        </div>
      </div>
    </div>
  );
}

// ─── ROLE BADGE ───────────────────────────────────────────────────────────────
function RoleBadge({ role }) {
  const map = {
    superadmin: { bg: "#1e1b4b", color: "#a5b4fc", label: "Super Admin" },
    admin: { bg: "#1e3a5f", color: "#93c5fd", label: "Admin" },
    member: { bg: "#1c1917", color: "#d6d3d1", label: "Member" }
  };
  const s = map[role] || map.member;
  return (
    <span style={{
      background: s.bg, color: s.color, padding: "2px 10px",
      borderRadius: 20, fontSize: 11, fontWeight: 700,
      fontFamily: "'DM Sans', sans-serif", letterSpacing: "0.04em"
    }}>{s.label}</span>
  );
}

function StatusDot({ status }) {
  return (
    <span style={{
      display: "inline-block", width: 8, height: 8, borderRadius: "50%",
      background: status === "active" ? "#22c55e" : "#ef4444",
      marginRight: 6
    }} />
  );
}

// ─── USER MANAGEMENT TABLE ────────────────────────────────────────────────────
function UserTable({ users, currentUser, onEdit, onToggle, onDelete, showRole }) {
  if (!users.length) return (
    <div style={{ textAlign: "center", padding: 32, color: "#475569", fontSize: 13 }}>
      No users found
    </div>
  );
  return (
    <div style={{ overflowX: "auto", borderRadius: 10, border: "1px solid #1e293b" }}>
      <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 13, fontFamily: "'DM Sans', sans-serif" }}>
        <thead>
          <tr style={{ background: "#0f172a" }}>
            {["Name", "Username", "Email", showRole && "Role", "Status", "Created", "Actions"].filter(Boolean).map(h => (
              <th key={h} style={{ padding: "10px 14px", color: "#475569", textAlign: "left", fontWeight: 600, fontSize: 11, letterSpacing: "0.06em", textTransform: "uppercase", borderBottom: "1px solid #1e293b", whiteSpace: "nowrap" }}>{h}</th>
            ))}
          </tr>
        </thead>
        <tbody>
          {users.map((u, i) => (
            <tr key={u.id} style={{ background: i % 2 === 0 ? "#0d1829" : "#0a1120" }}>
              <td style={{ padding: "10px 14px", color: "#f1f5f9", whiteSpace: "nowrap" }}>{u.name}</td>
              <td style={{ padding: "10px 14px", color: "#94a3b8", fontFamily: "'DM Mono', monospace", fontSize: 12 }}>{u.username}</td>
              <td style={{ padding: "10px 14px", color: "#64748b", fontSize: 12 }}>{u.email || "—"}</td>
              {showRole && <td style={{ padding: "10px 14px" }}><RoleBadge role={u.role} /></td>}
              <td style={{ padding: "10px 14px" }}>
                <StatusDot status={u.status} />
                <span style={{ color: u.status === "active" ? "#22c55e" : "#ef4444", fontSize: 12 }}>{u.status}</span>
              </td>
              <td style={{ padding: "10px 14px", color: "#475569", fontSize: 11 }}>{new Date(u.createdAt).toLocaleDateString("en-IN")}</td>
              <td style={{ padding: "10px 14px" }}>
                {u.id !== currentUser.id && u.id !== "superadmin" && (
                  <div style={{ display: "flex", gap: 6 }}>
                    <Btn small variant="ghost" onClick={() => onToggle(u)}>
                      {u.status === "active" ? "Disable" : "Enable"}
                    </Btn>
                    <Btn small variant="danger" onClick={() => onDelete(u)}>Delete</Btn>
                  </div>
                )}
                {u.id === currentUser.id && <span style={{ color: "#475569", fontSize: 11 }}>You</span>}
                {u.id === "superadmin" && u.id !== currentUser.id && <span style={{ color: "#475569", fontSize: 11 }}>Protected</span>}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

// ─── CREATE USER MODAL ────────────────────────────────────────────────────────
function CreateUserModal({ onClose, onCreated, toast, currentUserRole, fixedRole }) {
  const [form, setForm] = useState({ name: "", username: "", email: "", password: "", role: fixedRole || "member" });
  const [loading, setLoading] = useState(false);
  const set = k => e => setForm(p => ({ ...p, [k]: e.target.value }));

  const submit = async () => {
    if (!form.name || !form.username || !form.password) { toast("Name, username and password are required", "error"); return; }
    if (form.password.length < 6) { toast("Password must be at least 6 chars", "error"); return; }
    setLoading(true);
    const existing = await DB.getUser(form.username.trim().toLowerCase());
    if (existing) { toast("Username already taken", "error"); setLoading(false); return; }
    const newUser = {
      id: "user_" + uid(),
      role: form.role,
      name: form.name.trim(),
      username: form.username.trim().toLowerCase(),
      email: form.email.trim().toLowerCase(),
      passwordHash: hashPw(form.password),
      createdAt: new Date().toISOString(),
      status: "active",
      createdBy: "system"
    };
    await DB.saveUser(newUser);
    // Simulate email
    if (form.email) {
      toast(`✉ Credentials sent to ${form.email}: user="${newUser.username}", pw="${form.password}"`, "success");
    } else {
      toast(`User "${newUser.username}" created. Share credentials manually.`, "success");
    }
    onCreated(newUser);
    onClose();
    setLoading(false);
  };

  return (
    <Modal title={`Create ${fixedRole === "admin" ? "Admin" : fixedRole === "member" ? "Member" : "User"}`} onClose={onClose}>
      <Field label="Full Name"><Input placeholder="e.g. Ravi Sharma" value={form.name} onChange={set("name")} /></Field>
      <Field label="Username" hint="Used to sign in. Lowercase, no spaces.">
        <Input placeholder="e.g. ravi.sharma" value={form.username} onChange={set("username")} />
      </Field>
      <Field label="Email" hint="Credentials will be sent here">
        <Input type="email" placeholder="e.g. ravi@company.com" value={form.email} onChange={set("email")} />
      </Field>
      <Field label="Password" hint="Min 6 characters">
        <Input type="password" placeholder="Set a strong password" value={form.password} onChange={set("password")} />
      </Field>
      {!fixedRole && currentUserRole === "superadmin" && (
        <Field label="Role">
          <select value={form.role} onChange={set("role")} style={{
            width: "100%", background: "#0f172a", border: "1px solid #1e293b",
            borderRadius: 8, padding: "9px 12px", color: "#f1f5f9",
            fontFamily: "'DM Sans', sans-serif", fontSize: 14
          }}>
            <option value="admin">Admin</option>
            <option value="member">Member</option>
          </select>
        </Field>
      )}
      <div style={{ display: "flex", gap: 10, marginTop: 8 }}>
        <Btn full onClick={submit} disabled={loading}>{loading ? "Creating…" : "Create User"}</Btn>
        <Btn variant="ghost" onClick={onClose}>Cancel</Btn>
      </div>
    </Modal>
  );
}

// ─── ADD APP MODAL ────────────────────────────────────────────────────────────
function AddAppModal({ onClose, onAdded, toast }) {
  const [form, setForm] = useState({ name: "", icon: "📦", description: "", color: "#3b82f6", route: "" });
  const set = k => e => setForm(p => ({ ...p, [k]: e.target.value }));
  const icons = ["📦", "📊", "🔍", "📋", "📈", "🗂️", "💼", "🛠️", "📌", "🔧"];

  const submit = async () => {
    if (!form.name || !form.route) { toast("Name and route are required", "error"); return; }
    const apps = await DB.getApps();
    const newApp = {
      id: "app_" + uid(),
      name: form.name.trim(),
      icon: form.icon,
      description: form.description.trim(),
      color: form.color,
      route: form.route.trim().toLowerCase().replace(/\s+/g, "-"),
      createdAt: new Date().toISOString(),
      status: "active"
    };
    await DB.saveApps([...apps, newApp]);
    toast(`App "${newApp.name}" added to platform!`, "success");
    onAdded(newApp);
    onClose();
  };

  return (
    <Modal title="Add New Application" onClose={onClose}>
      <Field label="App Name"><Input placeholder="e.g. Invoice Extractor" value={form.name} onChange={set("name")} /></Field>
      <Field label="Icon">
        <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
          {icons.map(ic => (
            <button key={ic} onClick={() => setForm(p => ({ ...p, icon: ic }))} style={{
              width: 36, height: 36, borderRadius: 8, fontSize: 18, cursor: "pointer",
              background: form.icon === ic ? "#1e3a5f" : "#0f172a",
              border: `1px solid ${form.icon === ic ? "#3b82f6" : "#1e293b"}`
            }}>{ic}</button>
          ))}
        </div>
      </Field>
      <Field label="Description"><Input placeholder="What does this app do?" value={form.description} onChange={set("description")} /></Field>
      <Field label="Accent Color">
        <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
          <input type="color" value={form.color} onChange={set("color")} style={{ width: 36, height: 36, border: "none", background: "none", cursor: "pointer" }} />
          <span style={{ color: "#64748b", fontSize: 12 }}>Pick an accent color</span>
        </div>
      </Field>
      <Field label="Route ID" hint="Unique short identifier, e.g. 'invoice'">
        <Input placeholder="e.g. invoice" value={form.route} onChange={set("route")} />
      </Field>
      <div style={{ display: "flex", gap: 10, marginTop: 8 }}>
        <Btn full onClick={submit}>Add Application</Btn>
        <Btn variant="ghost" onClick={onClose}>Cancel</Btn>
      </div>
    </Modal>
  );
}

// ─── CHANGE PASSWORD MODAL ────────────────────────────────────────────────────
function ChangePwModal({ user, onClose, toast }) {
  const [cur, setCur] = useState(""); const [nw, setNw] = useState(""); const [cn, setCn] = useState("");
  const submit = async () => {
    if (user.passwordHash !== hashPw(cur)) { toast("Current password incorrect", "error"); return; }
    if (nw.length < 6) { toast("Min 6 characters", "error"); return; }
    if (nw !== cn) { toast("Passwords don't match", "error"); return; }
    user.passwordHash = hashPw(nw);
    await DB.saveUser(user);
    toast("Password changed!", "success");
    onClose();
  };
  return (
    <Modal title="Change Password" onClose={onClose} width={380}>
      <Field label="Current Password"><Input type="password" value={cur} onChange={e => setCur(e.target.value)} /></Field>
      <Field label="New Password"><Input type="password" value={nw} onChange={e => setNw(e.target.value)} /></Field>
      <Field label="Confirm New Password"><Input type="password" value={cn} onChange={e => setCn(e.target.value)} /></Field>
      <div style={{ display: "flex", gap: 10 }}>
        <Btn full onClick={submit}>Update Password</Btn>
        <Btn variant="ghost" onClick={onClose}>Cancel</Btn>
      </div>
    </Modal>
  );
}

// ─── TDS EXTRACTOR APP ────────────────────────────────────────────────────────
const TDS_FIELDS = ["Sr. No.", "TAN", "Major Head", "Nature of Payment", "A Tax", "Interest", "Fee under", "Total", "Challan No", "Date of Receipt", "Challan Serial No", "Assessment Year"];

// ─── FIRC EXTRACTOR APP ───────────────────────────────────────────────────────
const FIRC_FIELDS = ["To (POS)", "Issue Date", "Inward No", "Credit Date", "Currency", "FCY Amount", "Rate", "INR Amount", "Remitter Details"];

// ─── SHARED EXTRACTOR UI ──────────────────────────────────────────────────────
function fileToBase64(file) {
  return new Promise((res, rej) => {
    const r = new FileReader();
    r.onload = () => res(r.result.split(",")[1]);
    r.onerror = () => rej(new Error("Read failed"));
    r.readAsDataURL(file);
  });
}

async function extractDoc(file, fields, mode) {
  const base64 = await fileToBase64(file);
  const isPDF = file.type === "application/pdf";
  const mediaType = file.type || "image/jpeg";
  const label = mode === "tds" ? "TAN/Challan tax payment" : "FIRC inward remittance foreign currency";
  const prompt = `Extract these fields from this ${label} document. Return ONLY valid JSON with these exact keys: ${JSON.stringify(fields)}. Use null for missing values. Dates as DD/MM/YYYY. Numbers as plain strings. No markdown, no fences.`;
  const content = [
    isPDF
      ? { type: "document", source: { type: "base64", media_type: "application/pdf", data: base64 } }
      : { type: "image", source: { type: "base64", media_type: mediaType, data: base64 } },
    { type: "text", text: prompt }
  ];
  const resp = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ model: "claude-sonnet-4-20250514", max_tokens: 1000, messages: [{ role: "user", content }] })
  });
  if (!resp.ok) throw new Error(`API ${resp.status}`);
  const data = await resp.json();
  const raw = data.content?.find(b => b.type === "text")?.text || "{}";
  return JSON.parse(raw.replace(/```json|```/g, "").trim());
}

// ─── REAL XLSX EXPORT via SheetJS ────────────────────────────────────────────
function downloadXLSX(rows, fields, fname) {
  // Build array-of-arrays: header row + data rows
  const header = ["Source File", ...fields];
  const data = rows.map(r => [r.__filename ?? "", ...fields.map(f => r[f] ?? "")]);
  const ws = XLSX.utils.aoa_to_sheet([header, ...data]);

  // Column widths — auto-size based on max content length
  ws["!cols"] = header.map((h, ci) => {
    const maxLen = Math.max(
      h.length,
      ...data.map(row => String(row[ci] ?? "").length)
    );
    return { wch: Math.min(Math.max(maxLen + 2, 10), 40) };
  });

  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, "Extracted Data");

  // Write to base64 string — works inside sandboxed iframes
  const wbout = XLSX.write(wb, { bookType: "xlsx", type: "base64" });
  const dataUri = `data:application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;base64,${wbout}`;

  const a = document.createElement("a");
  a.href = dataUri;
  a.download = fname + ".xlsx";
  a.style.display = "none";
  document.body.appendChild(a);
  a.click();
  setTimeout(() => document.body.removeChild(a), 100);
}

function ExtractorApp({ mode, fields, title, accent, icon }) {
  const [files, setFiles] = useState([]);
  const [statuses, setStatuses] = useState({});
  const [results, setResults] = useState([]);
  const [running, setRunning] = useState(false);
  const [done, setDone] = useState(0);
  const inputRef = useRef();

  const addFiles = fs => {
    setFiles(p => {
      const ex = new Set(p.map(f => f.name + f.size));
      return [...p, ...fs.filter(f => !ex.has(f.name + f.size))];
    });
  };
  const handleDrop = e => { e.preventDefault(); addFiles(Array.from(e.dataTransfer.files)); };

  const extract = async () => {
    if (!files.length || running) return;
    setRunning(true); setResults([]); setDone(0);
    const init = {}; files.forEach(f => { init[f.name] = "pending"; }); setStatuses(init);
    const BATCH = 5;
    let count = 0;
    const all = [];
    for (let i = 0; i < files.length; i += BATCH) {
      await Promise.all(files.slice(i, i + BATCH).map(async file => {
        setStatuses(p => ({ ...p, [file.name]: "processing" }));
        try {
          const d = await extractDoc(file, fields, mode);
          d.__filename = file.name;
          all.push(d);
          setResults(p => [...p, d]);
          setStatuses(p => ({ ...p, [file.name]: "done" }));
        } catch {
          const e = { __filename: file.name }; fields.forEach(f => { e[f] = null; });
          all.push(e); setResults(p => [...p, e]);
          setStatuses(p => ({ ...p, [file.name]: "error" }));
        }
        count++; setDone(count);
      }));
    }
    setRunning(false);
  };

  const pct = files.length ? Math.round((done / files.length) * 100) : 0;
  const success = results.filter(r => fields.some(f => r[f] !== null));

  return (
    <div>
      <div style={{ display: "flex", alignItems: "center", gap: 12, marginBottom: 20 }}>
        <div style={{ fontSize: 28 }}>{icon}</div>
        <div>
          <h2 style={{ fontFamily: "'Syne', sans-serif", fontWeight: 800, fontSize: 20, color: "#f1f5f9" }}>{title}</h2>
          <p style={{ color: "#475569", fontSize: 12 }}>Upload PDF/image documents · Batch of 100+ · AI extraction</p>
        </div>
      </div>

      {/* Drop zone */}
      <div
        onDragOver={e => e.preventDefault()}
        onDrop={handleDrop}
        onClick={() => inputRef.current?.click()}
        style={{
          border: `2px dashed ${accent}55`, borderRadius: 12, padding: "32px 20px",
          textAlign: "center", cursor: "pointer", background: `${accent}08`,
          transition: "all 0.2s", marginBottom: 16
        }}
      >
        <input ref={inputRef} type="file" multiple accept=".pdf,.jpg,.jpeg,.png,.webp"
          style={{ display: "none" }} onChange={e => addFiles(Array.from(e.target.files))} />
        <div style={{ fontSize: 32, marginBottom: 8 }}>📂</div>
        <div style={{ color: "#e2e8f0", fontFamily: "'Syne', sans-serif", fontWeight: 600 }}>Drop files or click to browse</div>
        <div style={{ color: "#475569", fontSize: 12, marginTop: 4 }}>PDF, JPG, PNG · 100+ files</div>
      </div>

      {files.length > 0 && (
        <div style={{ maxHeight: 160, overflowY: "auto", background: "#0f172a", borderRadius: 8, border: "1px solid #1e293b", padding: 8, marginBottom: 12 }}>
          {files.map(f => (
            <div key={f.name} style={{ display: "flex", alignItems: "center", justifyContent: "space-between", padding: "4px 8px", borderRadius: 6, marginBottom: 4, background: "#0d1829" }}>
              <span style={{ color: "#94a3b8", fontSize: 11, fontFamily: "monospace", overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap", flex: 1 }}>{f.name}</span>
              {statuses[f.name] && (
                <span style={{
                  fontSize: 10, padding: "2px 8px", borderRadius: 10, fontWeight: 700, marginLeft: 8,
                  background: statuses[f.name] === "done" ? "#14532d" : statuses[f.name] === "error" ? "#450a0a" : statuses[f.name] === "processing" ? "#1e3a5f" : "#1e293b",
                  color: statuses[f.name] === "done" ? "#4ade80" : statuses[f.name] === "error" ? "#f87171" : statuses[f.name] === "processing" ? "#60a5fa" : "#94a3b8"
                }}>{statuses[f.name]}</span>
              )}
            </div>
          ))}
        </div>
      )}

      {running && (
        <div style={{ marginBottom: 12 }}>
          <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 4 }}>
            <span style={{ color: "#94a3b8", fontSize: 12 }}>Extracting…</span>
            <span style={{ color: "#e2e8f0", fontSize: 12 }}>{done}/{files.length} — {pct}%</span>
          </div>
          <div style={{ background: "#1e293b", borderRadius: 8, height: 6 }}>
            <div style={{ width: `${pct}%`, height: "100%", background: `linear-gradient(90deg, ${accent}, ${accent}99)`, borderRadius: 8, transition: "width 0.3s" }} />
          </div>
        </div>
      )}

      {/* Action buttons */}
      <div style={{ display: "flex", gap: 10 }}>
        <button onClick={extract} disabled={!files.length || running} style={{
          flex: 1, padding: "11px 0", border: "none", borderRadius: 8,
          cursor: !files.length || running ? "not-allowed" : "pointer",
          background: running ? "#1e293b" : `linear-gradient(135deg, ${accent}, ${accent}cc)`,
          color: running ? "#64748b" : "#fff",
          fontFamily: "'Syne', sans-serif", fontWeight: 700, fontSize: 14, letterSpacing: "0.03em"
        }}>
          {running ? `⏳ Extracting… (${done}/${files.length})` : "⚡ Extract Data"}
        </button>
        <button
          onClick={() => downloadXLSX(success, fields, mode + "_extracted_data")}
          disabled={!success.length}
          style={{
            flex: 1, padding: "11px 0", borderRadius: 8,
            cursor: !success.length ? "not-allowed" : "pointer",
            background: success.length ? "#052e16" : "#1e293b",
            color: success.length ? "#86efac" : "#475569",
            border: `1px solid ${success.length ? "#14532d" : "#334155"}`,
            fontFamily: "'Syne', sans-serif", fontWeight: 700, fontSize: 14, letterSpacing: "0.03em"
          }}>
          📥 Download Excel (.xlsx)
        </button>
      </div>

      {success.length > 0 && (
        <div style={{ marginTop: 8, color: "#4ade80", fontSize: 12 }}>
          ✓ {success.length} record{success.length !== 1 ? "s" : ""} extracted — ready to download
        </div>
      )}

      {/* Results table */}
      {success.length > 0 && (
        <div style={{ overflowX: "auto", marginTop: 16, borderRadius: 10, border: "1px solid #1e293b" }}>
          <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 11, fontFamily: "monospace" }}>
            <thead>
              <tr style={{ background: "#0f172a" }}>
                <th style={{ padding: "8px 12px", color: "#475569", textAlign: "left", borderBottom: "1px solid #1e293b" }}>File</th>
                {fields.map(f => <th key={f} style={{ padding: "8px 10px", color: accent, textAlign: "left", fontSize: 10, borderBottom: "1px solid #1e293b", whiteSpace: "nowrap" }}>{f}</th>)}
              </tr>
            </thead>
            <tbody>
              {success.map((row, i) => (
                <tr key={i} style={{ background: i % 2 === 0 ? "#0d1829" : "#0a1120" }}>
                  <td style={{ padding: "6px 12px", color: "#475569", maxWidth: 100, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{row.__filename}</td>
                  {fields.map(f => <td key={f} style={{ padding: "6px 10px", color: "#e2e8f0", whiteSpace: "nowrap" }}>{row[f] ?? <span style={{ color: "#334155" }}>—</span>}</td>)}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
}

// ─── PLACEHOLDER APP ──────────────────────────────────────────────────────────
function PlaceholderApp({ app }) {
  return (
    <div style={{ textAlign: "center", padding: 60 }}>
      <div style={{ fontSize: 56, marginBottom: 16 }}>{app.icon}</div>
      <h2 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, color: "#f1f5f9", marginBottom: 8 }}>{app.name}</h2>
      <p style={{ color: "#64748b", fontSize: 14, maxWidth: 360, margin: "0 auto" }}>{app.description}</p>
      <div style={{ marginTop: 24, padding: "12px 20px", background: "#1e293b", borderRadius: 10, display: "inline-block", color: "#94a3b8", fontSize: 13 }}>
        This app is registered on the platform. Add its functionality to fully activate it.
      </div>
    </div>
  );
}

// ─── MAIN DASHBOARD ───────────────────────────────────────────────────────────
function Dashboard({ user, onLogout, toast }) {
  const [section, setSection] = useState("apps");
  const [activeApp, setActiveApp] = useState(null);
  const [users, setUsers] = useState([]);
  const [apps, setApps] = useState([]);
  const [showCreateUser, setShowCreateUser] = useState(null);
  const [showAddApp, setShowAddApp] = useState(false);
  const [showChangePw, setShowChangePw] = useState(false);
  const [loading, setLoading] = useState(true);

  const isSA = user.role === "superadmin";
  const isAdmin = user.role === "admin" || isSA;

  useEffect(() => {
    (async () => {
      const [allUsers, allApps] = await Promise.all([DB.getAllUsers(), DB.getApps()]);
      setUsers(allUsers);
      setApps(allApps);
      setLoading(false);
    })();
  }, []);

  const refreshUsers = async () => setUsers(await DB.getAllUsers());
  const refreshApps = async () => setApps(await DB.getApps());

  const toggleUser = async u => {
    u.status = u.status === "active" ? "disabled" : "active";
    await DB.saveUser(u);
    toast(`User ${u.name} ${u.status}`, "success");
    refreshUsers();
  };

  const deleteUser = async u => {
    if (!window.confirm(`Delete ${u.name}? This cannot be undone.`)) return;
    await DB.deleteUser(u.id);
    toast(`User ${u.name} deleted`, "success");
    refreshUsers();
  };

  const toggleApp = async app => {
    const all = await DB.getApps();
    const updated = all.map(a => a.id === app.id ? { ...a, status: a.status === "active" ? "disabled" : "active" } : a);
    await DB.saveApps(updated);
    refreshApps();
    toast(`App "${app.name}" ${updated.find(a => a.id === app.id).status}`, "success");
  };

  const deleteApp = async app => {
    if (!window.confirm(`Remove ${app.name}?`)) return;
    const all = await DB.getApps();
    await DB.saveApps(all.filter(a => a.id !== app.id));
    refreshApps();
    if (activeApp?.id === app.id) setActiveApp(null);
    toast(`App "${app.name}" removed`, "success");
  };

  const admins = users.filter(u => u.role === "admin");
  const members = users.filter(u => u.role === "member");

  const navItems = [
    { id: "apps", icon: "⬡", label: "App Launcher" },
    ...(isAdmin ? [{ id: "members", icon: "👥", label: "Members" }] : []),
    ...(isSA ? [{ id: "admins", icon: "🛡️", label: "Admins" }, { id: "platform", icon: "⚙️", label: "Platform" }] : []),
    { id: "profile", icon: "◉", label: "Profile" }
  ];

  if (loading) return (
    <div style={{ minHeight: "100vh", background: "#020817", display: "flex", alignItems: "center", justifyContent: "center", color: "#64748b", fontFamily: "'Syne', sans-serif" }}>
      Loading platform…
    </div>
  );

  return (
    <div style={{ minHeight: "100vh", background: "#020817", display: "flex", fontFamily: "'DM Sans', sans-serif" }}>
      {/* Sidebar */}
      <div style={{
        width: 220, background: "#040d1a", borderRight: "1px solid #0f1e35",
        display: "flex", flexDirection: "column", padding: "24px 0", flexShrink: 0,
        position: "sticky", top: 0, height: "100vh"
      }}>
        <div style={{ padding: "0 20px 24px", borderBottom: "1px solid #0f1e35" }}>
          <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
            <div style={{ width: 34, height: 34, borderRadius: 10, background: "linear-gradient(135deg, #1e3a5f, #1e1b4b)", border: "1px solid #1e40af", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16 }}>⚡</div>
            <div>
              <div style={{ fontFamily: "'Syne', sans-serif", fontWeight: 800, fontSize: 14, color: "#f1f5f9" }}>DocFlow</div>
              <div style={{ fontSize: 10, color: "#334155" }}>Platform</div>
            </div>
          </div>
        </div>

        <nav style={{ flex: 1, padding: "16px 12px" }}>
          {navItems.map(n => (
            <button key={n.id} onClick={() => { setSection(n.id); setActiveApp(null); }} style={{
              width: "100%", display: "flex", alignItems: "center", gap: 10,
              padding: "9px 12px", borderRadius: 8, border: "none",
              background: section === n.id ? "#0f172a" : "transparent",
              color: section === n.id ? "#e2e8f0" : "#475569",
              cursor: "pointer", fontSize: 13, fontFamily: "'DM Sans', sans-serif",
              fontWeight: section === n.id ? 600 : 400, marginBottom: 2,
              textAlign: "left", transition: "all 0.15s",
              borderLeft: section === n.id ? "2px solid #3b82f6" : "2px solid transparent"
            }}>
              <span style={{ fontSize: 14 }}>{n.icon}</span> {n.label}
            </button>
          ))}
        </nav>

        <div style={{ padding: "16px 12px", borderTop: "1px solid #0f1e35" }}>
          <div style={{ padding: "10px 12px", background: "#0f172a", borderRadius: 8, marginBottom: 10 }}>
            <div style={{ fontSize: 12, color: "#f1f5f9", fontWeight: 600 }}>{user.name}</div>
            <div style={{ fontSize: 11, color: "#475569" }}>{user.username}</div>
            <div style={{ marginTop: 4 }}><RoleBadge role={user.role} /></div>
          </div>
          <button onClick={onLogout} style={{
            width: "100%", background: "transparent", border: "1px solid #1e293b",
            color: "#64748b", borderRadius: 8, padding: "8px", cursor: "pointer",
            fontFamily: "'DM Sans', sans-serif", fontSize: 12
          }}>Sign Out</button>
        </div>
      </div>

      {/* Main content */}
      <div style={{ flex: 1, padding: 28, overflowY: "auto", minHeight: "100vh" }}>

        {/* APP LAUNCHER */}
        {section === "apps" && !activeApp && (
          <div>
            <div style={{ marginBottom: 24 }}>
              <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, fontWeight: 800, color: "#f1f5f9", marginBottom: 4 }}>App Launcher</h1>
              <p style={{ color: "#475569", fontSize: 13 }}>Click any app to open it. All apps use your single login.</p>
            </div>
            <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(240px, 1fr))", gap: 16 }}>
              {apps.filter(a => a.status === "active").map(app => (
                <div key={app.id} onClick={() => setActiveApp(app)}
                  style={{
                    background: "#0d1829", border: "1px solid #1e293b", borderRadius: 14,
                    padding: 20, cursor: "pointer", transition: "all 0.2s",
                    borderTop: `3px solid ${app.color}`
                  }}
                  onMouseEnter={e => e.currentTarget.style.borderColor = app.color}
                  onMouseLeave={e => e.currentTarget.style.borderColor = "#1e293b"}
                >
                  <div style={{ fontSize: 36, marginBottom: 12 }}>{app.icon}</div>
                  <div style={{ fontFamily: "'Syne', sans-serif", fontWeight: 700, fontSize: 15, color: "#f1f5f9", marginBottom: 6 }}>{app.name}</div>
                  <div style={{ color: "#475569", fontSize: 12, lineHeight: 1.5 }}>{app.description}</div>
                  <div style={{ marginTop: 14, display: "inline-flex", alignItems: "center", gap: 6, background: `${app.color}22`, color: app.color, padding: "4px 12px", borderRadius: 20, fontSize: 12, fontWeight: 600 }}>
                    Open App →
                  </div>
                </div>
              ))}
              {isSA && (
                <div onClick={() => setShowAddApp(true)} style={{
                  background: "#0d1829", border: "2px dashed #1e293b", borderRadius: 14,
                  padding: 20, cursor: "pointer", display: "flex", alignItems: "center",
                  justifyContent: "center", flexDirection: "column", gap: 8, minHeight: 160,
                  transition: "border-color 0.2s"
                }}
                  onMouseEnter={e => e.currentTarget.style.borderColor = "#3b82f6"}
                  onMouseLeave={e => e.currentTarget.style.borderColor = "#1e293b"}
                >
                  <div style={{ fontSize: 28, color: "#334155" }}>+</div>
                  <div style={{ color: "#475569", fontSize: 13, fontWeight: 600 }}>Add New App</div>
                </div>
              )}
            </div>
          </div>
        )}

        {/* ACTIVE APP */}
        {section === "apps" && activeApp && (
          <div>
            <button onClick={() => setActiveApp(null)} style={{
              background: "#0f172a", border: "1px solid #1e293b", color: "#94a3b8",
              borderRadius: 8, padding: "6px 14px", cursor: "pointer",
              fontFamily: "'DM Sans', sans-serif", fontSize: 12, marginBottom: 20,
              display: "flex", alignItems: "center", gap: 6
            }}>← Back to Launcher</button>
            <div style={{ background: "#0d1829", border: "1px solid #1e293b", borderRadius: 14, padding: 24 }}>
              {activeApp.route === "tds" && <ExtractorApp mode="tds" fields={TDS_FIELDS} title="TDS Extractor" accent="#3b82f6" icon="🧾" />}
              {activeApp.route === "firc" && <ExtractorApp mode="firc" fields={FIRC_FIELDS} title="FIRC Extractor" accent="#8b5cf6" icon="💱" />}
              {activeApp.route !== "tds" && activeApp.route !== "firc" && <PlaceholderApp app={activeApp} />}
            </div>
          </div>
        )}

        {/* MEMBERS */}
        {section === "members" && isAdmin && (
          <div>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 20 }}>
              <div>
                <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, fontWeight: 800, color: "#f1f5f9", marginBottom: 4 }}>Members</h1>
                <p style={{ color: "#475569", fontSize: 13 }}>{members.length} member{members.length !== 1 ? "s" : ""} registered</p>
              </div>
              <Btn onClick={() => setShowCreateUser("member")}>+ New Member</Btn>
            </div>
            <UserTable users={members} currentUser={user} onEdit={() => {}} onToggle={toggleUser} onDelete={deleteUser} showRole={false} />
          </div>
        )}

        {/* ADMINS */}
        {section === "admins" && isSA && (
          <div>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 20 }}>
              <div>
                <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, fontWeight: 800, color: "#f1f5f9", marginBottom: 4 }}>Admins</h1>
                <p style={{ color: "#475569", fontSize: 13 }}>{admins.length} admin{admins.length !== 1 ? "s" : ""} registered</p>
              </div>
              <Btn onClick={() => setShowCreateUser("admin")} variant="purple">+ New Admin</Btn>
            </div>
            <UserTable users={admins} currentUser={user} onEdit={() => {}} onToggle={toggleUser} onDelete={deleteUser} showRole={false} />
          </div>
        )}

        {/* PLATFORM SETTINGS */}
        {section === "platform" && isSA && (
          <div>
            <div style={{ marginBottom: 24 }}>
              <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, fontWeight: 800, color: "#f1f5f9", marginBottom: 4 }}>Platform Settings</h1>
              <p style={{ color: "#475569", fontSize: 13 }}>Manage registered applications</p>
            </div>
            <div style={{ display: "flex", justifyContent: "flex-end", marginBottom: 16 }}>
              <Btn onClick={() => setShowAddApp(true)}>+ Register App</Btn>
            </div>
            <div style={{ borderRadius: 10, border: "1px solid #1e293b", overflow: "hidden" }}>
              <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 13 }}>
                <thead>
                  <tr style={{ background: "#0f172a" }}>
                    {["App", "Route", "Status", "Added", "Actions"].map(h => (
                      <th key={h} style={{ padding: "10px 14px", color: "#475569", textAlign: "left", fontWeight: 600, fontSize: 11, letterSpacing: "0.06em", textTransform: "uppercase", borderBottom: "1px solid #1e293b" }}>{h}</th>
                    ))}
                  </tr>
                </thead>
                <tbody>
                  {apps.map((app, i) => (
                    <tr key={app.id} style={{ background: i % 2 === 0 ? "#0d1829" : "#0a1120" }}>
                      <td style={{ padding: "10px 14px" }}>
                        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                          <span style={{ fontSize: 20 }}>{app.icon}</span>
                          <div>
                            <div style={{ color: "#f1f5f9", fontWeight: 600 }}>{app.name}</div>
                            <div style={{ color: "#475569", fontSize: 11 }}>{app.description?.slice(0, 50)}…</div>
                          </div>
                        </div>
                      </td>
                      <td style={{ padding: "10px 14px", fontFamily: "monospace", color: "#64748b", fontSize: 12 }}>/{app.route}</td>
                      <td style={{ padding: "10px 14px" }}>
                        <StatusDot status={app.status} />
                        <span style={{ color: app.status === "active" ? "#22c55e" : "#ef4444", fontSize: 12 }}>{app.status}</span>
                      </td>
                      <td style={{ padding: "10px 14px", color: "#475569", fontSize: 11 }}>{new Date(app.createdAt).toLocaleDateString("en-IN")}</td>
                      <td style={{ padding: "10px 14px" }}>
                        <div style={{ display: "flex", gap: 6 }}>
                          <Btn small variant="ghost" onClick={() => toggleApp(app)}>{app.status === "active" ? "Disable" : "Enable"}</Btn>
                          <Btn small variant="danger" onClick={() => deleteApp(app)}>Remove</Btn>
                        </div>
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>

            {/* Stats */}
            <div style={{ marginTop: 24, display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(180px, 1fr))", gap: 14 }}>
              {[
                { label: "Total Users", value: users.length, icon: "👥", color: "#3b82f6" },
                { label: "Admins", value: admins.length, icon: "🛡️", color: "#8b5cf6" },
                { label: "Members", value: members.length, icon: "◉", color: "#22c55e" },
                { label: "Apps", value: apps.length, icon: "⬡", color: "#f59e0b" },
              ].map(s => (
                <div key={s.label} style={{ background: "#0d1829", border: "1px solid #1e293b", borderRadius: 12, padding: 16 }}>
                  <div style={{ fontSize: 22, marginBottom: 6 }}>{s.icon}</div>
                  <div style={{ fontFamily: "'Syne', sans-serif", fontSize: 26, fontWeight: 800, color: s.color }}>{s.value}</div>
                  <div style={{ fontSize: 12, color: "#475569" }}>{s.label}</div>
                </div>
              ))}
            </div>
          </div>
        )}

        {/* PROFILE */}
        {section === "profile" && (
          <div style={{ maxWidth: 500 }}>
            <h1 style={{ fontFamily: "'Syne', sans-serif", fontSize: 22, fontWeight: 800, color: "#f1f5f9", marginBottom: 20 }}>My Profile</h1>
            <div style={{ background: "#0d1829", border: "1px solid #1e293b", borderRadius: 14, padding: 24, marginBottom: 16 }}>
              <div style={{ display: "flex", alignItems: "center", gap: 16, marginBottom: 20 }}>
                <div style={{ width: 56, height: 56, borderRadius: 16, background: "linear-gradient(135deg, #1e3a5f, #1e1b4b)", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 24 }}>
                  {user.name[0].toUpperCase()}
                </div>
                <div>
                  <div style={{ fontFamily: "'Syne', sans-serif", fontWeight: 700, fontSize: 18, color: "#f1f5f9" }}>{user.name}</div>
                  <RoleBadge role={user.role} />
                </div>
              </div>
              {[["Username", user.username], ["Email", user.email || "Not set"], ["Member since", new Date(user.createdAt).toLocaleDateString("en-IN")]].map(([k, v]) => (
                <div key={k} style={{ display: "flex", justifyContent: "space-between", padding: "10px 0", borderBottom: "1px solid #0f172a" }}>
                  <span style={{ color: "#64748b", fontSize: 13 }}>{k}</span>
                  <span style={{ color: "#e2e8f0", fontSize: 13, fontFamily: k === "Username" ? "monospace" : "inherit" }}>{v}</span>
                </div>
              ))}
            </div>
            <Btn onClick={() => setShowChangePw(true)} variant="ghost" full>🔑 Change Password</Btn>
          </div>
        )}
      </div>

      {/* Modals */}
      {showCreateUser && (
        <CreateUserModal
          onClose={() => setShowCreateUser(null)}
          onCreated={() => refreshUsers()}
          toast={toast}
          currentUserRole={user.role}
          fixedRole={showCreateUser}
        />
      )}
      {showAddApp && (
        <AddAppModal
          onClose={() => setShowAddApp(false)}
          onAdded={() => refreshApps()}
          toast={toast}
        />
      )}
      {showChangePw && (
        <ChangePwModal user={user} onClose={() => setShowChangePw(false)} toast={toast} />
      )}
    </div>
  );
}

// ─── ROOT APP ─────────────────────────────────────────────────────────────────
export default function App() {
  const [session, setSession] = useState(null);
  const [ready, setReady] = useState(false);
  const { toasts, toast, removeToast } = useToast();

  useEffect(() => {
    (async () => {
      await DB.init();
      // Check session
      const saved = sessionStorage.getItem("docflow_session");
      if (saved) {
        try {
          const u = JSON.parse(saved);
          const fresh = await DB.getUserById(u.id);
          if (fresh && fresh.status === "active") setSession(fresh);
          else sessionStorage.removeItem("docflow_session");
        } catch { sessionStorage.removeItem("docflow_session"); }
      }
      setReady(true);
    })();
  }, []);

  const handleLogin = user => {
    setSession(user);
    sessionStorage.setItem("docflow_session", JSON.stringify(user));
  };

  const handleLogout = () => {
    setSession(null);
    sessionStorage.removeItem("docflow_session");
    toast("Signed out successfully", "success");
  };

  if (!ready) return (
    <div style={{ minHeight: "100vh", background: "#020817", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <div style={{ color: "#3b82f6", fontFamily: "'Syne', sans-serif", fontSize: 16 }}>Initialising DocFlow…</div>
    </div>
  );

  return (
    <>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Sans:wght@400;500;600&family=DM+Mono&display=swap');
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { background: #020817; }
        @keyframes slideIn { from { transform: translateX(20px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
        ::-webkit-scrollbar { width: 5px; height: 5px; }
        ::-webkit-scrollbar-track { background: #0f172a; }
        ::-webkit-scrollbar-thumb { background: #1e293b; border-radius: 3px; }
        select { outline: none; }
      `}</style>

      <Toast toasts={toasts} remove={removeToast} />

      {!session
        ? <LoginPage onLogin={handleLogin} toast={toast} />
        : <Dashboard user={session} onLogout={handleLogout} toast={toast} />
      }
    </>
  );
}
