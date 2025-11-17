import React, { useEffect, useState, useRef } from "react";

// Single-file React app (JSX) for a client-side "Create to Earn" demo
// Styling uses Tailwind CSS classes (assumes Tailwind is available in the host project).
// Persisted in localStorage so the app "feels real" without a backend.

export default function CreateToEarnApp() {
  // --- loader ---
  const [loading, setLoading] = useState(true);

  // --- auth ---
  const [currentUser, setCurrentUser] = useState(null);
  const [view, setView] = useState("auth"); // 'auth' or 'app'

  // UI state for auth forms
  const [loginUser, setLoginUser] = useState("");
  const [loginPass, setLoginPass] = useState("");

  const [regUser, setRegUser] = useState("");
  const [regPass, setRegPass] = useState("");
  const [regReferral, setRegReferral] = useState("");

  // data
  const [users, setUsers] = useState({}); // keyed by username
  const [posts, setPosts] = useState([]); // global posts

  // app UI
  const [activePage, setActivePage] = useState("home");
  const fileRef = useRef(null);
  const [postText, setPostText] = useState("");
  const [selectedFile, setSelectedFile] = useState(null);

  // mining
  const [isMining, setIsMining] = useState(false);
  const [mineProgress, setMineProgress] = useState(0);
  const mineIntervalRef = useRef(null);

  // messages
  const [message, setMessage] = useState("");

  // --- helpers for localStorage persistence ---
  useEffect(() => {
    const savedUsers = localStorage.getItem("cte_users");
    const savedPosts = localStorage.getItem("cte_posts");
    const savedCurrent = localStorage.getItem("cte_currentUser");
    if (savedUsers) setUsers(JSON.parse(savedUsers));
    if (savedPosts) setPosts(JSON.parse(savedPosts));
    if (savedCurrent) {
      try {
        const cu = JSON.parse(savedCurrent);
        setCurrentUser(cu);
        setView("app");
      } catch {}
    }

    // loader 5s
    const t = setTimeout(() => setLoading(false), 5000);
    return () => clearTimeout(t);
  }, []);

  useEffect(() => {
    localStorage.setItem("cte_users", JSON.stringify(users));
  }, [users]);
  useEffect(() => {
    localStorage.setItem("cte_posts", JSON.stringify(posts));
  }, [posts]);
  useEffect(() => {
    localStorage.setItem("cte_currentUser", JSON.stringify(currentUser));
  }, [currentUser]);

  // --- utility functions ---
  function genReferralCode(name) {
    const base = (name || "user").slice(0, 3).toUpperCase();
    const rnd = Math.random().toString(36).slice(2, 8).toUpperCase();
    return base + rnd;
  }

  function showMsg(text, timeout = 3000) {
    setMessage(text);
    setTimeout(() => setMessage(""), timeout);
  }

  // --- auth actions ---
  function handleRegister(e) {
    e && e.preventDefault();
    if (!regUser || !regPass) return showMsg("برائے مہربانی یوزرنیم اور پاسورڈ ڈالیں");
    if (users[regUser]) return showMsg("یہ یوزرنیم پہلے موجود ہے");
    const refCode = genReferralCode(regUser);
    const newUser = {
      username: regUser,
      password: regPass,
      balance: 0.0,
      minedAmount: 0.0,
      referralCode: refCode,
      referredBy: null,
      referrals: [],
      posts: [],
      payouts: [],
    };
    // handle referral bonus
    if (regReferral) {
      const refOwnerKey = Object.keys(users).find(
        (k) => users[k].referralCode === regReferral
      );
      if (refOwnerKey) {
        // credit referrer
        users[refOwnerKey].balance = +(users[refOwnerKey].balance + 1).toFixed(2);
        users[refOwnerKey].referrals.push(regUser);
        newUser.referredBy = users[refOwnerKey].username;
        showMsg("رجسٹریشن مکمل ہوئی — ریفر کرنے والے کو 1$ کریڈٹ ملا");
      } else {
        showMsg("ریفرل کوڈ درست نہیں ہے — براہِ کرم درست کوڈ ڈالیں یا چھوڑ دیں");
      }
    }

    const newUsers = { ...users, [regUser]: newUser };
    setUsers(newUsers);
    setCurrentUser(newUser);
    setView("app");
    setActivePage("home");
  }

  function handleLogin(e) {
    e && e.preventDefault();
    const u = users[loginUser];
    if (!u || u.password !== loginPass) return showMsg("غلط یوزرنیم یا پاسورڈ");
    setCurrentUser(u);
    setView("app");
    setActivePage("home");
  }

  function handleLogout() {
    setCurrentUser(null);
    setView("auth");
    setLoginUser("");
    setLoginPass("");
  }

  // --- posting / create ---
  function handleFileChange(e) {
    const f = e.target.files[0];
    if (f) setSelectedFile(f);
  }

  function handleCreatePost(e) {
    e && e.preventDefault();
    if (!currentUser) return showMsg("پہلے لوگن کریں");
    if (!postText && !selectedFile) return showMsg("کچھ ٹیکسٹ یا فائل اپلوڈ کریں");

    const post = {
      id: Date.now(),
      author: currentUser.username,
      text: postText,
      fileName: selectedFile ? selectedFile.name : null,
      fileType: selectedFile ? selectedFile.type : null,
      fileURL: selectedFile ? URL.createObjectURL(selectedFile) : null,
      createdAt: new Date().toISOString(),
    };

    const newPosts = [post, ...posts];
    setPosts(newPosts);

    // credit user $0.1 per post
    const updatedUsers = { ...users };
    const u = { ...updatedUsers[currentUser.username] };
    u.posts = [post, ...(u.posts || [])];
    u.balance = +(u.balance + 0.1).toFixed(2);
    updatedUsers[currentUser.username] = u;
    setUsers(updatedUsers);
    setCurrentUser(u);

    // reset form
    setPostText("");
    setSelectedFile(null);
    if (fileRef.current) fileRef.current.value = "";

    showMsg("پوسٹ کردی گئی — آپ کو $0.10 ملا");
  }

  // --- mining ---
  function startMining() {
    if (!currentUser) return showMsg("پہلے لوگن کریں");
    if (isMining) return;
    setIsMining(true);
    setMineProgress(0);
    mineIntervalRef.current = setInterval(() => {
      setMineProgress((p) => {
        const np = p + Math.random() * 10; // random progress
        if (np >= 100) {
          clearInterval(mineIntervalRef.current);
          // reward mined amount (small amount)
          const mined = +(0.05 + Math.random() * 0.15).toFixed(4); // e.g., 0.05-0.2
          const updatedUsers = { ...users };
          const u = { ...updatedUsers[currentUser.username] };
          u.minedAmount = +(u.minedAmount + mined).toFixed(4);
          updatedUsers[currentUser.username] = u;
          setUsers(updatedUsers);
          setCurrentUser(u);
          setIsMining(false);
          setMineProgress(0);
          showMsg(`Mining complete — ${mined}$ added to mined (claimable)`);
          return 0;
        }
        return np;
      });
    }, 800);
  }

  function claimMine() {
    if (!currentUser) return showMsg("پہلے لوگن کریں");
    const updatedUsers = { ...users };
    const u = { ...updatedUsers[currentUser.username] };
    if (!u.minedAmount || u.minedAmount <= 0) return showMsg("کوئی مائن کیا گیا بیلنس نہیں");
    u.balance = +(u.balance + u.minedAmount).toFixed(4);
    u.minedAmount = 0;
    updatedUsers[currentUser.username] = u;
    setUsers(updatedUsers);
    setCurrentUser(u);
    showMsg("مائننگ کلیم کرلیا گیا — بیلنس اپڈیٹ ہوگیا");
  }

  // --- payout ---
  function requestPayout() {
    if (!currentUser) return showMsg("پہلے لوگن کریں");
    if (currentUser.balance < 20) return showMsg("پے آؤٹ کے لیے کم از کم $20 درکار ہیں");
    // simulate payout and reset balance (in real app you'd call a backend/payment gateway)
    const updatedUsers = { ...users };
    const u = { ...updatedUsers[currentUser.username] };
    const payout = u.balance;
    u.payouts = [
      ...(u.payouts || []),
      { id: Date.now(), amount: +payout.toFixed(2), at: new Date().toISOString() },
    ];
    u.balance = 0;
    updatedUsers[currentUser.username] = u;
    setUsers(updatedUsers);
    setCurrentUser(u);
    showMsg(`Payout requested: $${payout.toFixed(2)} — simulated success`);
  }

  // --- referral invite link ---
  function getMyReferralLink() {
    if (!currentUser) return "";
    return `${window.location.origin}${window.location.pathname}?ref=${currentUser.referralCode}`;
  }

  // --- small helpers for pages ---
  function renderHome() {
    return (
      <div className="p-4">
        <h2 className="text-2xl font-semibold mb-3">Welcome, {currentUser.username}</h2>
        <div className="mb-4 text-sm">
          Balance: <strong>${(+currentUser.balance).toFixed(4)}</strong>
          <br /> Mined (claimable): <strong>${(+currentUser.minedAmount).toFixed(4)}</strong>
        </div>

        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div className="p-4 rounded-lg shadow-sm bg-white">
            <h3 className="font-medium mb-2">Create (post)</h3>
            <form onSubmit={handleCreatePost}>
              <textarea
                value={postText}
                onChange={(e) => setPostText(e.target.value)}
                placeholder="اپنی پوسٹ لکھیں..."
                className="w-full p-2 border rounded mb-2"
              />
              <input ref={fileRef} type="file" onChange={handleFileChange} className="mb-2" />
              <div className="flex gap-2">
                <button type="submit" className="px-3 py-1 rounded bg-blue-600 text-white">
                  پوسٹ کریں (آپ کو $0.10 ملے گا)
                </button>
                <button
                  type="button"
                  className="px-3 py-1 rounded border"
                  onClick={() => {
                    setPostText("");
                    setSelectedFile(null);
                    if (fileRef.current) fileRef.current.value = "";
                  }}
                >
                  صاف کریں
                </button>
              </div>
            </form>
          </div>

          <div className="p-4 rounded-lg shadow-sm bg-white">
            <h3 className="font-medium mb-2">Mining</h3>
            <div className="mb-2">Progress: {Math.round(mineProgress)}%</div>
            <div className="flex gap-2">
              <button onClick={startMining} className="px-3 py-1 rounded bg-green-600 text-white">
                Start Mining
              </button>
              <button onClick={claimMine} className="px-3 py-1 rounded border">
                Claim Mined
              </button>
            </div>
            <div className="mt-3 text-sm">
              Tip: mining yields a small random amount per run. Repeat to build up.
            </div>
          </div>
        </div>

        <div className="mt-6">
          <h3 className="font-medium mb-2">Recent posts</h3>
          <div className="space-y-3">
            {posts.map((p) => (
              <div key={p.id} className="p-3 rounded border bg-white">
                <div className="text-sm text-gray-500">{p.author} · {new Date(p.createdAt).toLocaleString()}</div>
                <div className="mt-1">{p.text}</div>
                {p.fileURL && (
                  <div className="mt-2">
                    <a href={p.fileURL} target="_blank" rel="noreferrer" className="underline">
                      {p.fileName}
                    </a>
                  </div>
                )}
              </div>
            ))}
            {posts.length === 0 && <div className="text-sm text-gray-500">No posts yet — be the first!</div>}
          </div>
        </div>
      </div>
    );
  }

  function renderBuySell() {
    return (
      <div className="p-4">
        <h2 className="text-xl font-semibold mb-3">Buy & Sell</h2>
        <div className="text-sm">یہ سیکشن ابھی سمولیشن ہے — آپ آئٹمز لسٹ کر سکتے ہیں، مگر ریئل پیمنٹس بیک اینڈ کے بغیر ممکن نہیں۔</div>
      </div>
    );
  }

  function renderEarn() {
    return (
      <div className="p-4">
        <h2 className="text-xl font-semibold mb-3">Earn</h2>
        <div className="mb-2">آپ کا بیلنس: <strong>${(+currentUser.balance).toFixed(4)}</strong></div>
        <div className="mb-2">Minimum payout: <strong>$20.00</strong></div>
        <div className="flex gap-2">
          <button onClick={requestPayout} className="px-3 py-1 rounded bg-indigo-600 text-white">
            Request Payout
          </button>
        </div>

        <div className="mt-4">
          <h4 className="font-medium">Payout history</h4>
          <div className="mt-2 space-y-2 text-sm">
            {currentUser.payouts && currentUser.payouts.length > 0 ? (
              currentUser.payouts.map((p) => (
                <div key={p.id} className="p-2 border rounded bg-white">
                  ${p.amount.toFixed(2)} — {new Date(p.at).toLocaleString()}
                </div>
              ))
            ) : (
              <div className="text-gray-500">No payouts yet</div>
            )}
          </div>
        </div>
      </div>
    );
  }

  function renderMinePage() {
    return (
      <div className="p-4">
        <h2 className="text-xl font-semibold mb-3">Mine</h2>
        <div className="mb-2">Mined (claimable): <strong>${(+currentUser.minedAmount).toFixed(4)}</strong></div>
        <div className="flex gap-2">
          <button onClick={startMining} className="px-3 py-1 rounded bg-green-600 text-white">
            Start Mining
          </button>
          <button onClick={claimMine} className="px-3 py-1 rounded border">
            Claim Now
          </button>
        </div>
      </div>
    );
  }

  function renderProfile() {
    return (
      <div className="p-4">
        <h2 className="text-xl font-semibold mb-3">Profile & Referral</h2>
        <div className="mb-2">Username: {currentUser.username}</div>
        <div className="mb-2">Referral Code: <strong>{currentUser.referralCode}</strong></div>
        <div className="mb-2">Referral Link (copy):</div>
        <input className="p-2 w-full border rounded mb-2" readOnly value={getMyReferralLink()} />
        <div className="mb-2">Referrals: {currentUser.referrals?.length || 0}</div>
        <div className="text-sm text-gray-600">When someone registers with your referral code they get registered and you get $1 credit (client-side simulation).</div>
      </div>
    );
  }

  // --- main render ---
  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gradient-to-r from-blue-50 to-indigo-50">
        <div className="flex flex-col items-center">
          <div className="w-24 h-24 rounded-full flex items-center justify-center mb-4 animate-bounce-slow" style={{background:'#3b82f6'}}>
            <div className="w-6 h-6 rounded-full bg-white" />
          </div>
          <div className="text-center text-gray-700">Loading site — please wait...</div>
        </div>
        <style>{`
          @keyframes bounce-slow { 0%,100% { transform: translateY(0) } 50% { transform: translateY(-24px) } }
          .animate-bounce-slow { animation: bounce-slow 1s infinite; }
        `}</style>
      </div>
    );
  }

  if (view === "auth") {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-50 p-4">
        <div className="w-full max-w-4xl grid md:grid-cols-2 gap-6">
          <div className="p-6 rounded-lg shadow bg-white">
            <h2 className="text-2xl mb-4 font-semibold">Login</h2>
            <form onSubmit={handleLogin}>
              <input
                className="w-full p-2 border rounded mb-2"
                placeholder="Username"
                value={loginUser}
                onChange={(e) => setLoginUser(e.target.value)}
              />
              <input
                className="w-full p-2 border rounded mb-2"
                placeholder="Password"
                type="password"
                value={loginPass}
                onChange={(e) => setLoginPass(e.target.value)}
              />
              <div className="flex gap-2">
                <button type="submit" className="px-3 py-1 bg-blue-600 text-white rounded">Login</button>
                <button type="button" className="px-3 py-1 border rounded" onClick={() => { setLoginUser('demo'); setLoginPass('demo123'); }}>Demo creds</button>
              </div>
            </form>
          </div>

          <div className="p-6 rounded-lg shadow bg-white">
            <h2 className="text-2xl mb-4 font-semibold">Register</h2>
            <form onSubmit={handleRegister}>
              <input
                className="w-full p-2 border rounded mb-2"
                placeholder="Choose username"
                value={regUser}
                onChange={(e) => setRegUser(e.target.value)}
              />
              <input
                className="w-full p-2 border rounded mb-2"
                placeholder="Choose password"
                type="password"
                value={regPass}
                onChange={(e) => setRegPass(e.target.value)}
              />
              <input
                className="w-full p-2 border rounded mb-2"
                placeholder="Referral code (optional)"
                value={regReferral}
                onChange={(e) => setRegReferral(e.target.value)}
              />
              <div className="flex gap-2">
                <button type="submit" className="px-3 py-1 bg-green-600 text-white rounded">Register & Start</button>
                <button type="button" className="px-3 py-1 border rounded" onClick={() => { setRegUser(''); setRegPass(''); setRegReferral(''); }}>Clear</button>
              </div>
            </form>
            <div className="mt-3 text-sm text-gray-600">نوٹ: یہ ایک کلائنٹ سائیڈ پیلاٹ پروجیکٹ ہے — ریئل پیمنٹس یا پے آؤٹ بیک اینڈ کے بغیر سمولیشن ہے۔</div>
          </div>
        </div>
        {message && (
          <div className="fixed bottom-6 left-1/2 transform -translate-x-1/2 bg-black text-white px-4 py-2 rounded">{message}</div>
        )}
      </div>
    );
  }

  // logged-in app UI
  return (
    <div className="min-h-screen bg-gray-100">
      <div className="flex">
        {/* sidebar */}
        <aside className="w-64 bg-white min-h-screen border-r">
          <div className="p-4 border-b">
            <div className="text-lg font-bold">Create to Earn</div>
            <div className="text-sm text-gray-500">Signed in as {currentUser.username}</div>
          </div>
          <nav className="p-4 space-y-2">
            <button onClick={() => setActivePage('home')} className={`w-full text-left p-2 rounded ${activePage==='home'?'bg-blue-50':''}`}>Home</button>
            <button onClick={() => setActivePage('create')} className={`w-full text-left p-2 rounded ${activePage==='create'?'bg-blue-50':''}`}>Create</button>
            <button onClick={() => setActivePage('buy-sell')} className={`w-full text-left p-2 rounded ${activePage==='buy-sell'?'bg-blue-50':''}`}>Buy & Sell</button>
            <button onClick={() => setActivePage('earn')} className={`w-full text-left p-2 rounded ${activePage==='earn'?'bg-blue-50':''}`}>Earn</button>
            <button onClick={() => setActivePage('mine')} className={`w-full text-left p-2 rounded ${activePage==='mine'?'bg-blue-50':''}`}>Mine</button>
            <button onClick={() => setActivePage('profile')} className={`w-full text-left p-2 rounded ${activePage==='profile'?'bg-blue-50':''}`}>Profile / Referral</button>
            <div className="pt-4 border-t mt-4">
              <button onClick={handleLogout} className="w-full text-left p-2 rounded text-red-600">Logout</button>
            </div>
          </nav>
        </aside>

        {/* main */}
        <main className="flex-1">
          <div className="p-4">
            {activePage === 'home' && renderHome()}
            {activePage === 'create' && renderHome() /* re-use create area inside home for simplicity */}
            {activePage === 'buy-sell' && renderBuySell()}
            {activePage === 'earn' && renderEarn()}
            {activePage === 'mine' && renderMinePage()}
            {activePage === 'profile' && renderProfile()}
          </div>
        </main>
      </div>

      {message && (
        <div className="fixed bottom-6 left-1/2 transform -translate-x-1/2 bg-black text-white px-4 py-2 rounded">{message}</div>
      )}
    </div>
  );
}
