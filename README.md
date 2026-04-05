<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Content Machine PRO</title>
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
    <style>
        :root { 
            --bg-main: #0b0b10; --bg-panel: #16161e; --bg-input: #20202b; 
            --text-main: #e2e8f0; --text-muted: #64748b; 
            --accent-glow: #c084fc; --btn-light: #d8b4fe; --btn-text: #2e1065; 
        }

        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Inter', system-ui, sans-serif; }
        body { background: var(--bg-main); color: var(--text-main); padding: 40px; }

        .flex-center { height: 80vh; display: flex; justify-content: center; align-items: center; }
        .auth-box { background: var(--bg-panel); padding: 30px; border-radius: 16px; width: 340px; border: 1px solid #2a2a3f; }

        .container { display: none; max-width: 1100px; margin: 0 auto; grid-template-columns: 1.2fr 1fr; gap: 30px; }
        .panel { background: var(--bg-panel); padding: 25px; border-radius: 16px; border: 1px solid #2a2a3f; }
        
        h1, h2 { margin-bottom: 20px; font-weight: 600; }
        label { display: block; font-size: 12px; color: var(--text-muted); margin-bottom: 8px; margin-top: 15px; text-transform: uppercase; letter-spacing: 1px; }

        textarea, input { 
            width: 100%; padding: 15px; border-radius: 10px; border: 1px solid #2a2a3f; 
            background: var(--bg-input); color: white; outline: none; transition: 0.2s; 
        }
        textarea:focus, input:focus { border-color: var(--accent-glow); }

        button { width: 100%; padding: 15px; border: none; border-radius: 10px; font-weight: bold; cursor: pointer; transition: 0.2s; }
        button.primary { background: var(--btn-light); color: var(--btn-text); margin-top: 25px; box-shadow: 0 4px 15px rgba(192, 132, 252, 0.2); }
        button.primary:hover { transform: translateY(-2px); filter: brightness(1.1); }
        button.secondary { background: #2a2a3f; color: white; margin-top: 10px; font-size: 12px; }

        .platforms { display: flex; gap: 8px; margin-top: 5px; }
        .plat-btn {
            flex: 1; padding: 10px; background: var(--bg-input); border: 1px solid #2a2a3f;
            border-radius: 8px; color: var(--text-muted); cursor: pointer; text-align: center; font-size: 12px; font-weight: bold;
        }
        .plat-btn.active { background: rgba(192, 132, 252, 0.2); border-color: var(--accent-glow); color: var(--btn-light); }

        .queue-item { background: var(--bg-input); padding: 15px; border-radius: 12px; margin-bottom: 12px; border-left: 4px solid var(--accent-glow); }
        .q-date { font-size: 11px; color: var(--accent-glow); font-weight: bold; margin-bottom: 5px; }
        .q-text { font-size: 14px; line-height: 1.5; }
        .q-plats { margin-top: 8px; display: flex; gap: 5px; }
        .badge { font-size: 9px; padding: 2px 6px; background: #2a2a3f; border-radius: 4px; color: #aaa; }
    </style>
</head>
<body>

    <div id="auth-screen" class="flex-center">
        <div class="auth-box">
            <h2 style="text-align:center; color: var(--btn-light)">Content Login</h2>
            <input id="email" type="email" placeholder="Email" style="margin-top:20px">
            <input id="pass" type="password" placeholder="Пароль">
            <button class="primary" onclick="handleAuth()">Войти в систему</button>
            <button class="secondary" onclick="logout()">Выйти</button>
        </div>
    </div>

    <div id="main-app" class="container">
        <div class="panel">
            <h2>Создать пост</h2>
            <label>Текст публикации</label>
            <textarea id="postText" rows="8" placeholder="О чем сегодня расскажем?"></textarea>

            <label>Куда отправляем?</label>
            <div class="platforms" id="platContainer">
                <div class="plat-btn active" data-name="TG">Telegram</div>
                <div class="plat-btn" data-name="YT">YouTube</div>
                <div class="plat-btn" data-name="INST">Instagram</div>
                <div class="plat-btn" data-name="TT">TikTok</div>
            </div>

            <label>Дата и время выхода</label>
            <input type="datetime-local" id="postDate">

            <button class="primary" onclick="schedulePost()">Запланировать в очередь</button>
        </div>

        <div class="panel">
            <h2>Очередь постов</h2>
            <div id="queueList"></div>
        </div>
    </div>

    <script>
        const SUPABASE_URL = "https://ltrglnjbsyoelkagqbbh.supabase.co";
        const SUPABASE_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Imx0cmdsbmpic3lvZWxrYWdxYmJoIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzUxNzgyNDksImV4cCI6MjA5MDc1NDI0OX0.VhZiD9OVDfv3nlfYysBNXPsax4RkevErwWImMlqW9BM";
        const db = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

        let currentUser = null;

        // Переключение кнопок соцсетей
        document.querySelectorAll('.plat-btn').forEach(btn => {
            btn.onclick = () => btn.classList.toggle('active');
        });

        // Авторизация
        async function handleAuth() {
            const email = document.getElementById('email').value;
            const password = document.getElementById('pass').value;
            const { data, error } = await db.auth.signInWithPassword({ email, password });
            if (error) alert("Ошибка: " + error.message);
        }

        db.auth.onAuthStateChange((event, session) => {
            if (session) {
                currentUser = session.user;
                document.getElementById('auth-screen').style.display = 'none';
                document.getElementById('main-app').style.display = 'grid';
                loadPosts();
            } else {
                document.getElementById('auth-screen').style.display = 'flex';
                document.getElementById('main-app').style.display = 'none';
            }
        });

        function logout() { db.auth.signOut(); }

        // Загрузка постов
        async function loadPosts() {
            const { data, error } = await db.from('posts')
                .select('*')
                .eq('user_id', currentUser.id)
                .order('post_date', { ascending: true });

            const list = document.getElementById('queueList');
            list.innerHTML = '';
            
            if (data) {
                data.forEach(p => {
                    const item = document.createElement('div');
                    item.className = 'queue-item';
                    const date = new Date(p.post_date).toLocaleString('ru-RU', { day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit' });
                    const platforms = p.platforms.map(pl => `<span class="badge">${pl}</span>`).join(' ');
                    
                    item.innerHTML = `
                        <div class="q-date">${date}</div>
                        <div class="q-text">${p.text}</div>
                        <div class="q-plats">${platforms}</div>
                    `;
                    list.appendChild(item);
                });
            }
        }

        // Сохранение поста
        async function schedulePost() {
            const text = document.getElementById('postText').value;
            const date = document.getElementById('postDate').value;
            const platforms = [];
            document.querySelectorAll('.plat-btn.active').forEach(b => platforms.push(b.dataset.name));

            if (!text || !date || platforms.length === 0) {
                alert("Заполни все поля и выбери хотя бы одну соцсеть!");
                return;
            }

            const { error } = await db.from('posts').insert([{
                user_id: currentUser.id,
                text: text,
                post_date: date,
                platforms: platforms,
                status: 'pending'
            }]);

            if (!error) {
                document.getElementById('postText').value = '';
                loadPosts();
            } else {
                alert("Ошибка сохранения: " + error.message);
            }
        }
    </script>
</body>
</html>


# content-machine
