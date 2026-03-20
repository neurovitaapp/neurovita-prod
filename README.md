<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>NeuroVita Diamond Elite</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@300;400;600;800&display=swap" rel="stylesheet">
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://unpkg.com/lucide@latest"></script>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, getDoc, updateDoc, onSnapshot, query } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const config = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'neurovita-prod';
        const app = initializeApp(config);
        window.NV = { db: getFirestore(app), auth: getAuth(app), appId, fs: { collection, doc, setDoc, getDoc, updateDoc, onSnapshot, query, signInAnonymously } };
    </script>

    <style>
        body { font-family: 'Plus Jakarta Sans', sans-serif; background: #050505; color: #fff; -webkit-tap-highlight-color: transparent; }
        .glass { background: rgba(255,255,255,0.03); backdrop-filter: blur(20px); border: 1px solid rgba(255,255,255,0.08); border-radius: 24px; }
        .btn-premium { background: #fff; color: #000; font-weight: 800; border-radius: 16px; transition: 0.2s; }
        .btn-premium:active { transform: scale(0.95); }
        .input-premium { background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.1); border-radius: 14px; padding: 16px; color: white; outline: none; width: 100%; }
        .input-premium:focus { border-color: #10b981; }
    </style>
</head>
<body>
    <div id="root"></div>
    <script type="text/babel">
        const { useState, useEffect } = React;
        const ADMIN_EMAIL = 'neurovita.app@gmail.com';
        const ADMIN_PASS = 'Ricardo.21';

        function App() {
            const [view, setView] = useState('splash');
            const [user, setUser] = useState(null);
            const [therapists, setTherapists] = useState([]);
            const [loading, setLoading] = useState(false);
            const [activeTab, setActiveTab] = useState('home');

            useEffect(() => {
                const init = async () => {
                    try {
                        await window.NV.fs.signInAnonymously(window.NV.auth);
                        const saved = localStorage.getItem('nv_session');
                        if (saved) { setUser(JSON.parse(saved)); setView('dashboard'); }
                        else { setTimeout(() => setView('landing'), 1000); }
                    } catch (e) { setView('landing'); }
                };
                init();
            }, []);

            useEffect(() => {
                if (view === 'dashboard' && user?.role === 'ADMIN') {
                    const q = window.NV.fs.query(window.NV.fs.collection(window.NV.db, 'artifacts', window.NV.appId, 'public', 'data', 'users'));
                    return window.NV.fs.onSnapshot(q, (snap) => {
                        setTherapists(snap.docs.map(d => ({ id: d.id, ...d.data() })).filter(t => t.email !== ADMIN_EMAIL));
                    });
                }
                if (window.lucide) lucide.createIcons();
            }, [view, activeTab]);

            const handleAuth = async (e, mode) => {
                e.preventDefault();
                setLoading(true);
                const email = e.target.email.value.trim().toLowerCase();
                const pass = e.target.pass.value;
                const name = e.target.name?.value || 'Terapeuta';

                if (email === ADMIN_EMAIL && pass === ADMIN_PASS) {
                    const data = { email, name: 'CEO Hikári', role: 'ADMIN' };
                    setUser(data); localStorage.setItem('nv_session', JSON.stringify(data));
                    setView('dashboard');
                } else if (mode === 'register') {
                    const expiry = Date.now() + (3 * 86400000);
                    const newUser = { email, pass, name, role: 'THERAPIST', expiry, status: 'Trial' };
                    await window.NV.fs.setDoc(window.NV.fs.doc(window.NV.db, 'artifacts', window.NV.appId, 'public', 'data', 'users', email), newUser);
                    setUser(newUser); localStorage.setItem('nv_session', JSON.stringify(newUser));
                    setView('dashboard');
                } else {
                    const snap = await window.NV.fs.getDoc(window.NV.fs.doc(window.NV.db, 'artifacts', window.NV.appId, 'public', 'data', 'users', email));
                    if (snap.exists() && snap.data().pass === pass) {
                        setUser(snap.data()); localStorage.setItem('nv_session', JSON.stringify(snap.data()));
                        setView('dashboard');
                    } else { alert("Dados incorretos."); }
                }
                setLoading(false);
            };

            const addTime = async (email, days) => {
                const docRef = window.NV.fs.doc(window.NV.db, 'artifacts', window.NV.appId, 'public', 'data', 'users', email);
                const snap = await window.NV.fs.getDoc(docRef);
                if (snap.exists()) {
                    const newExp = Math.max(snap.data().expiry, Date.now()) + (days * 86400000);
                    await window.NV.fs.updateDoc(docRef, { expiry: newExp, status: 'Ativo' });
                }
            };

            if (view === 'splash') return <div className="h-screen flex items-center justify-center bg-black"><h1 className="text-2xl font-black italic animate-pulse text-emerald-500 uppercase">NeuroVita</h1></div>;

            if (view === 'landing') return (
                <div className="min-h-screen flex flex-col items-center justify-center p-10 text-center">
                    <h1 className="text-6xl font-black italic mb-4 leading-none uppercase">Neuro<br/><span className="text-emerald-500">Vita</span></h1>
                    <p className="text-stone-500 font-bold uppercase text-[10px] tracking-[0.4em] mb-12">Diamond Edition 2026</p>
                    <div className="w-full max-w-xs space-y-4">
                        <button onClick={() => setView('login')} className="w-full btn-premium py-6 text-xl uppercase tracking-widest">Entrar</button>
                        <button onClick={() => setView('register')} className="w-full bg-white/5 py-6 rounded-2xl font-bold text-xs uppercase text-stone-500">Registo (3 Dias Grátis)</button>
                    </div>
                </div>
            );

            if (view === 'login' || view === 'register') return (
                <div className="min-h-screen flex items-center justify-center p-6">
                    <form onSubmit={(e) => handleAuth(e, view)} className="w-full max-w-md glass p-10 shadow-2xl">
                        <h2 className="text-3xl font-black mb-8 italic uppercase">{view === 'login' ? 'Entrar' : 'Registar'}</h2>
                        <div className="space-y-4">
                            {view === 'register' && <input name="name" placeholder="Nome / Clínica" className="input-premium" required />}
                            <input name="email" type="email" placeholder="E-mail" className="input-premium" required />
                            <input name="pass" type="password" placeholder="Chave" className="input-premium" required />
                            <button disabled={loading} className="w-full btn-premium py-6 text-xl uppercase mt-4">{loading ? '...' : 'Confirmar'}</button>
                            <button type="button" onClick={() => setView('landing')} className="w-full text-stone-600 font-bold text-[10px] uppercase mt-4">Voltar</button>
                        </div>
                    </form>
                </div>
            );

            if (view === 'dashboard') return (
                <div className="min-h-screen flex flex-col md:flex-row">
                    <aside className="w-full md:w-80 p-8 border-r border-white/5 flex flex-col glass !rounded-none">
                        <div className="flex items-center gap-3 mb-12"><div className="w-10 h-10 bg-white rounded-lg flex items-center justify-center text-black font-black italic">NV</div><span className="font-black text-xl italic uppercase">NeuroVita</span></div>
                        <nav className="flex-1 space-y-2">
                            <button onClick={() => setActiveTab('home')} className={`w-full text-left p-4 rounded-xl font-bold flex items-center gap-3 ${activeTab === 'home' ? 'bg-emerald-600' : 'text-stone-500'}`}><i data-lucide="home"></i> Início</button>
                            <button onClick={() => setActiveTab('users')} className={`w-full text-left p-4 rounded-xl font-bold flex items-center gap-3 ${activeTab === 'users' ? 'bg-emerald-600' : 'text-stone-500'}`}><i data-lucide="users"></i> {user.role === 'ADMIN' ? 'Terapeutas' : 'Pacientes'}</button>
                        </nav>
                        <button onClick={() => { localStorage.removeItem('nv_session'); setView('landing'); }} className="p-4 text-stone-600 font-bold text-xs uppercase flex items-center gap-3 mt-10 hover:text-red-500 transition-all border-t border-white/5 pt-10"><i data-lucide="log-out"></i> Sair</button>
                    </aside>
                    <main className="flex-1 p-8 md:p-12 overflow-y-auto">
                        <header className="mb-12">
                            <h2 className="text-4xl font-black italic tracking-tighter leading-none mb-1 uppercase">Olá, {user.name.split(' ')[0]}</h2>
                            <p className="text-stone-600 text-[10px] font-black uppercase tracking-[0.4em]">{user.role === 'ADMIN' ? 'Centro de Comando Fundadora' : `Licença Trial: ${Math.max(0, Math.ceil((user.expiry - Date.now()) / 86400000))} Dias`}</p>
                        </header>
                        {activeTab === 'home' && (
                            <div className="grid gap-6 grid-cols-1 md:grid-cols-2">
                                <div className="glass p-10 border-l-4 border-emerald-500"><p className="text-stone-600 text-[10px] font-black uppercase mb-2">Na Rede</p><h3 className="text-7xl font-black">{user.role === 'ADMIN' ? therapists.length : '12'}</h3></div>
                                <div className="glass p-10 flex flex-col justify-center border border-white/5"><h3 className="text-xl font-bold uppercase mb-1">Estado da Conta</h3><p className="text-stone-500 font-bold uppercase text-[10px]">{user.role === 'ADMIN' ? 'LIGAÇÃO MASTER ATIVA' : `VALIDADE: ${new Date(user.expiry).toLocaleDateString()}`}</p></div>
                            </div>
                        )}
                        {activeTab === 'users' && user.role === 'ADMIN' && (
                            <div className="space-y-4">
                                <h3 className="text-xl font-bold italic mb-6 uppercase">Gestão de Licenciados</h3>
                                {therapists.map(t => (
                                    <div key={t.email} className="glass p-6 flex flex-col md:flex-row items-center justify-between gap-6 border border-white/5">
                                        <div><h4 className="text-lg font-bold uppercase mb-1">{t.name}</h4><p className="text-stone-600 text-[10px] font-bold uppercase">{t.email} • Exp: {new Date(t.expiry).toLocaleDateString()}</p></div>
                                        <div className="flex gap-2"><button onClick={() => addTime(t.email, 7)} className="bg-stone-900 px-4 py-3 rounded-xl text-[10px] font-bold uppercase border border-white/5">+7 Dias</button><button onClick={() => addTime(t.email, 30)} className="bg-white text-black px-4 py-3 rounded-xl text-[10px] font-bold uppercase shadow-xl">+30 Dias</button></div>
                                    </div>
                                ))}
                            </div>
                        )}
                        {activeTab === 'users' && user.role === 'THERAPIST' && <div className="text-center py-24 glass border-dashed border-2 border-white/10"><i data-lucide="user-plus" className="mx-auto mb-6 text-stone-800" size={64} /><h3 className="text-2xl font-black mb-4 uppercase">Novo Paciente</h3><button className="btn-premium px-10 py-4 text-xs uppercase shadow-xl">Adicionar à Rede</button></div>}
                    </main>
                </div>
            );
        }
        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>

