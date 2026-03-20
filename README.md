<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>NeuroVita Diamond Elite</title>
    
    <!-- Frameworks & Design -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@300;400;600;800&display=swap" rel="stylesheet">
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://unpkg.com/lucide@latest"></script>

    <!-- Firebase Engine -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, getDoc, updateDoc, onSnapshot, query } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const config = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'neurovita-prod';
        const app = initializeApp(config);
        
        window.NV_CORE = { 
            db: getFirestore(app), 
            auth: getAuth(app), 
            appId, 
            fs: { collection, doc, setDoc, getDoc, updateDoc, onSnapshot, query, signInAnonymously } 
        };
    </script>

    <style>
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #050505; color: #ffffff; -webkit-tap-highlight-color: transparent; overflow-x: hidden; }
        .glass-panel { background: rgba(255, 255, 255, 0.02); backdrop-filter: blur(25px); border: 1px solid rgba(255, 255, 255, 0.08); border-radius: 32px; }
        .btn-diamond { background: #ffffff; color: #000000; font-weight: 800; border-radius: 20px; transition: 0.2s; border-bottom: 5px solid #d1d5db; }
        .btn-diamond:active { transform: scale(0.96) translateY(3px); border-bottom-width: 1px; }
        .input-premium { background: rgba(255, 255, 255, 0.05); border: 1px solid rgba(255, 255, 255, 0.1); border-radius: 18px; padding: 20px; color: white; outline: none; width: 100%; transition: 0.3s; }
        .input-premium:focus { border-color: #10b981; }
        ::-webkit-scrollbar { display: none; }
        .animate-fade { animation: fadeIn 0.4s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
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
            const [toast, setToast] = useState(null);

            const showToast = (msg) => { setToast(msg); setTimeout(() => setToast(null), 3000); };

            // 1. Sessão e Autenticação Silenciosa
            useEffect(() => {
                const initApp = async () => {
                    try {
                        const { auth, fs } = window.NV_CORE;
                        await fs.signInAnonymously(auth);
                        
                        const savedSession = localStorage.getItem('nv_master_session');
                        if (savedSession) {
                            setUser(JSON.parse(savedSession));
                            setView('dashboard');
                        } else {
                            setTimeout(() => setView('landing'), 1500);
                        }
                    } catch (e) { setView('landing'); }
                };
                initApp();
            }, []);

            // 2. Sincronização em Tempo Real para Admin
            useEffect(() => {
                if (view === 'dashboard' && user?.role === 'ADMIN') {
                    const { db, appId, fs } = window.NV_CORE;
                    const q = fs.query(fs.collection(db, 'artifacts', appId, 'public', 'data', 'users'));
                    return fs.onSnapshot(q, (snap) => {
                        const list = snap.docs.map(d => ({ id: d.id, ...d.data() }));
                        setTherapists(list.filter(t => t.email !== ADMIN_EMAIL));
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

                try {
                    const { db, appId, fs } = window.NV_CORE;
                    
                    if (email === ADMIN_EMAIL && pass === ADMIN_PASS) {
                        const adminData = { email, name: 'CEO Hikári', role: 'ADMIN' };
                        setUser(adminData);
                        localStorage.setItem('nv_master_session', JSON.stringify(adminData));
                        setView('dashboard');
                        showToast("Acesso Mestre Ativado");
                    } else if (mode === 'register') {
                        const expiry = Date.now() + (3 * 24 * 60 * 60 * 1000); // 3 dias trial
                        const newUser = { email, pass, name, role: 'THERAPIST', status: 'Trial', expiry, createdAt: Date.now() };
                        await fs.setDoc(fs.doc(db, 'artifacts', appId, 'public', 'data', 'users', email), newUser);
                        setUser(newUser);
                        localStorage.setItem('nv_master_session', JSON.stringify(newUser));
                        setView('dashboard');
                        showToast("Bem-vindo! Tem 3 dias de Trial.");
                    } else {
                        const snap = await fs.getDoc(fs.doc(db, 'artifacts', appId, 'public', 'data', 'users', email));
                        if (snap.exists() && snap.data().pass === pass) {
                            setUser(snap.data());
                            localStorage.setItem('nv_master_session', JSON.stringify(snap.data()));
                            setView('dashboard');
                        } else { showToast("Credenciais Inválidas"); }
                    }
                } catch (err) { showToast("Erro de Ligação"); }
                setLoading(false);
            };

            const addTime = async (email, days) => {
                try {
                    const { db, appId, fs } = window.NV_CORE;
                    const docRef = fs.doc(db, 'artifacts', appId, 'public', 'data', 'users', email);
                    const snap = await fs.getDoc(docRef);
                    if (snap.exists()) {
                        const newExpiry = Math.max(snap.data().expiry, Date.now()) + (days * 86400000);
                        await fs.updateDoc(docRef, { expiry: newExpiry, status: 'Ativo' });
                        showToast(`+${days} dias adicionados.`);
                    }
                } catch (e) { showToast("Erro ao atualizar."); }
            };

            const logout = () => {
                localStorage.removeItem('nv_master_session');
                setUser(null);
                setView('landing');
            };

            const Icon = ({ name, size = 20 }) => {
                useEffect(() => { if(window.lucide) lucide.createIcons(); }, [view, activeTab]);
                return <i data-lucide={name} style={{ width: size, height: size }}></i>;
            };

            // --- RENDERIZADOR DE TELAS ---
            
            if (view === 'splash') return (
                <div className="h-screen flex flex-col items-center justify-center bg-black animate-fade">
                    <div className="w-20 h-20 bg-emerald-500/10 rounded-3xl flex items-center justify-center animate-pulse border border-emerald-500/20">
                        <Icon name="sparkles" size={40} />
                    </div>
                    <h1 className="text-3xl font-black italic mt-6 tracking-tighter">Neuro<span className="text-emerald-500 uppercase">Vita</span></h1>
                </div>
            );

            if (view === 'landing') return (
                <div className="min-h-screen p-10 flex flex-col items-center justify-center text-center animate-fade">
                    <h1 className="text-7xl font-black italic tracking-tighter leading-none mb-4 uppercase">Neuro<br/><span className="text-emerald-500 tracking-widest text-4xl">Vita</span></h1>
                    <p className="text-stone-600 font-bold uppercase tracking-[0.4em] text-[10px] mb-20">SaaS Management Elite 2026</p>
                    <div className="w-full max-w-xs space-y-4">
                        <button onClick={() => setView('login')} className="w-full btn-diamond py-7 text-xl uppercase tracking-widest shadow-2xl">Entrar</button>
                        <button onClick={() => setView('register')} className="w-full bg-white/5 py-6 rounded-2xl font-bold text-xs uppercase text-stone-500 hover:text-white transition-all">Novo Registo (3 Dias Trial)</button>
                    </div>
                    <footer className="absolute bottom-10 opacity-20 text-[8px] font-bold uppercase tracking-[1em]">Propriedade Hikári Fafe</footer>
                </div>
            );

            if (view === 'login' || view === 'register') return (
                <div className="min-h-screen p-6 flex flex-col items-center justify-center animate-fade">
                    <form onSubmit={(e) => handleAuth(e, view)} className="w-full max-w-md glass-panel p-10 shadow-2xl border-b-8 border-emerald-900/50">
                        <h2 className="text-3xl font-black mb-10 italic uppercase tracking-tighter">{view === 'login' ? 'Identificação' : 'Novo Legado'}</h2>
                        <div className="space-y-4">
                            {view === 'register' && <input name="name" placeholder="Nome do Colega / Clínica" className="input-premium" required />}
                            <input name="email" type="email" placeholder="E-mail" className="input-premium" required />
                            <input name="pass" type="password" placeholder="Chave Master" className="input-premium" required />
                            <button disabled={loading} className="w-full btn-diamond py-6 text-xl uppercase mt-4 flex justify-center">
                                {loading ? <div className="w-8 h-8 border-4 border-black border-t-transparent rounded-full animate-spin"></div> : 'Confirmar'}
                            </button>
                            <button type="button" onClick={() => setView('landing')} className="w-full text-stone-600 font-bold text-[10px] uppercase mt-6 tracking-widest">Voltar atrás</button>
                        </div>
                    </form>
                </div>
            );

            if (view === 'dashboard') return (
                <div className="min-h-screen flex flex-col md:flex-row bg-[#050505] animate-fade">
                    <aside className="w-full md:w-80 p-8 border-r border-white/5 bg-black/50 flex flex-col glass-panel !rounded-none">
                        <div className="flex items-center gap-4 mb-16">
                            <div className="w-10 h-10 bg-white rounded-xl flex items-center justify-center text-black font-black italic">NV</div>
                            <span className="font-black text-xl italic tracking-tighter uppercase">NeuroVita</span>
                        </div>
                        <nav className="flex-1 space-y-2">
                            <button onClick={() => setActiveTab('home')} className={`w-full text-left p-5 rounded-2xl font-black flex items-center gap-4 transition-all ${activeTab === 'home' ? 'bg-emerald-600 text-white shadow-xl shadow-emerald-500/10' : 'text-stone-600'}`}><Icon name="layout-dashboard" /> Início</button>
                            <button onClick={() => setActiveTab('users')} className={`w-full text-left p-5 rounded-2xl font-black flex items-center gap-4 transition-all ${activeTab === 'users' ? 'bg-emerald-600 text-white shadow-xl shadow-emerald-500/10' : 'text-stone-600'}`}><Icon name="users" /> {user.role === 'ADMIN' ? 'Terapeutas' : 'Pacientes'}</button>
                        </nav>
                        <button onClick={logout} className="p-5 text-stone-700 font-black text-xs uppercase flex items-center gap-4 mt-10 hover:text-red-500 transition-all border-t border-white/5 pt-10"><Icon name="log-out" /> Sair do Sistema</button>
                    </aside>

                    <main className="flex-1 p-8 md:p-16 overflow-y-auto">
                        <header className="mb-16 flex justify-between items-end">
                            <div>
                                <h2 className="text-5xl font-black italic tracking-tighter leading-none mb-2">Olá, {user.name.split(' ')[0]}</h2>
                                <p className="text-stone-600 text-[10px] font-black uppercase tracking-[0.4em]">{user.role === 'ADMIN' ? 'Centro de Comando' : `Trial: ${Math.max(0, Math.ceil((user.expiry - Date.now()) / 86400000))} Dias`}</p>
                            </div>
                            <div className="w-16 h-16 rounded-3xl bg-emerald-600 flex items-center justify-center font-black text-white shadow-2xl border-b-4 border-emerald-800 uppercase leading-none">{user.name.charAt(0)}</div>
                        </header>

                        {activeTab === 'home' && (
                            <div className="grid gap-8 grid-cols-1 md:grid-cols-2">
                                <div className="glass-panel p-10 border-l-8 border-emerald-500 relative group overflow-hidden">
                                    <div className="absolute -right-10 -bottom-10 opacity-5 group-hover:scale-110 transition-transform"><Icon name="activity" size={240}/></div>
                                    <p className="text-stone-600 text-[10px] font-black uppercase tracking-widest mb-4">Total na Rede</p>
                                    <h3 className="text-8xl font-black tracking-tighter leading-none">{user.role === 'ADMIN' ? therapists.length : '12'}</h3>
                                    <p className="text-emerald-500 text-[10px] font-black mt-4 uppercase tracking-widest">Utilizadores Ativos</p>
                                </div>
                                <div className="glass-panel p-10 flex flex-col justify-center border border-white/5">
                                    <h3 className="text-2xl font-black uppercase tracking-tighter mb-2 italic">Estado da Conta</h3>
                                    <p className="text-stone-500 font-bold uppercase text-[10px] tracking-widest leading-relaxed">
                                        {user.role === 'ADMIN' ? 'CEO MASTER VITALÍCIO' : `VÁLIDO ATÉ: ${new Date(user.expiry).toLocaleDateString('pt-PT')}`}
                                    </p>
                                </div>
                            </div>
                        )}

                        {activeTab === 'users' && user.role === 'ADMIN' && (
                            <div className="space-y-4 animate-fade">
                                <h3 className="text-2xl font-black italic mb-10 tracking-tighter uppercase">Gestão de Licenciados</h3>
                                {therapists.length === 0 ? (
                                    <div className="text-center py-20 opacity-30 italic">Sem terapeutas registados até ao momento.</div>
                                ) : (
                                    therapists.map(t => (
                                        <div key={t.email} className="glass-panel p-8 flex flex-col md:flex-row items-center justify-between gap-8 hover:bg-white/[0.04] transition-all border border-white/5">
                                            <div className="flex-1">
                                                <h4 className="text-2xl font-black italic uppercase leading-none mb-1">{t.name}</h4>
                                                <p className="text-stone-600 text-[10px] font-bold uppercase tracking-widest">{t.email}</p>
                                                <div className="mt-4 flex items-center gap-3">
                                                    <span className={`px-4 py-1.5 rounded-full text-[9px] font-black uppercase tracking-widest border ${t.expiry > Date.now() ? 'bg-emerald-500/10 text-emerald-500 border-emerald-500/20' : 'bg-red-500/10 text-red-500 border-red-500/20'}`}>{t.expiry > Date.now() ? 'Ativo' : 'Expirado'}</span>
                                                    <p className="text-[10px] font-bold text-stone-500 uppercase tracking-widest">Expira: {new Date(t.expiry).toLocaleDateString()}</p>
                                                </div>
                                            </div>
                                            <div className="flex gap-3 w-full md:w-auto">
                                                <button onClick={() => addTime(t.email, 7)} className="flex-1 md:flex-none bg-stone-900 text-white font-black px-8 py-5 rounded-2xl text-[10px] uppercase border border-white/5 hover:bg-stone-800 transition-all">+7 Dias</button>
                                                <button onClick={() => addTime(t.email, 30)} className="flex-1 md:flex-none bg-white text-black font-black px-8 py-5 rounded-2xl text-[10px] uppercase shadow-xl hover:bg-emerald-500 hover:text-white transition-all shadow-emerald-500/10">+30 Dias</button>
                                            </div>
                                        </div>
                                    ))
                                )}
                            </div>
                        )}
                        
                        {activeTab === 'users' && user.role === 'THERAPIST' && (
                            <div className="text-center py-32 glass-panel border-dashed border-2 border-white/10 animate-fade">
                                <Icon name="user-plus" className="mx-auto mb-8 text-stone-800" size={120} />
                                <h3 className="text-5xl font-black mb-6 tracking-tighter italic uppercase leading-none">Novo Paciente</h3>
                                <p className="text-stone-600 max-w-sm mx-auto text-xs font-bold tracking-widest uppercase leading-relaxed opacity-60">Inicie o mapeamento biológico e comece a tratar a raiz dos sintomas do seu paciente.</p>
                                <button className="mt-12 btn-diamond px-16 py-7 text-sm uppercase tracking-widest shadow-2xl">Adicionar à Rede</button>
                            </div>
                        )}
                    </main>

                    {toast && (
                        <div className="fixed bottom-10 left-1/2 -translate-x-1/2 glass-panel px-10 py-5 rounded-2xl font-black text-[10px] uppercase tracking-[0.3em] shadow-2xl z-[100] border-emerald-500/50 text-emerald-500 animate-fade text-center">
                            {toast}
                        </div>
                    )}
                </div>
            );

            return null; // Logic is handled inside components
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>

