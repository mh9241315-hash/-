is_read = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

# ==================== إنشاء قاعدة البيانات ====================
with app.app_context():
    db.create_all()

# ==================== المساعدات (Helpers) ====================
def get_current_user():
    if 'username' in session:
        return User.query.filter_by(username=session['username']).first()
    return None

def add_notification(user_to, user_from, message):
    if user_to != user_from:
        new_note = Notification(user_to=user_to, user_from=user_from, message=message)
        db.session.add(new_note)
        db.session.commit()

# ==================== القوالب (UI Design) ====================
BASE_CSS = """
:root { --primary: #e50914; --dark: #141414; --card: #1f1f1f; --text: #ffffff; }
body { background-color: var(--dark); color: var(--text); font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; direction: rtl; margin: 0; }
.navbar { background: rgba(0,0,0,0.9); padding: 15px; position: sticky; top: 0; z-index: 1000; display: flex; justify-content: space-around; align-items: center; border-bottom: 1px solid #333; }
.navbar a { color: white; text-decoration: none; font-weight: bold; }
.container { max-width: 700px; margin: 20px auto; padding: 10px; }
.card { background: var(--card); border-radius: 12px; padding: 15px; margin-bottom: 20px; border: 1px solid #333; }
.story-container { display: flex; overflow-x: auto; gap: 10px; padding: 10px 0; }
.story-card { min-width: 100px; height: 150px; background: #333; border-radius: 10px; position: relative; overflow: hidden; border: 2px solid var(--primary); flex-shrink: 0; }
.story-card img { width: 100%; height: 100%; object-fit: cover; }
.story-author { position: absolute; bottom: 5px; right: 5px; font-size: 10px; background: rgba(0,0,0,0.5); padding: 2px 5px; }
.btn { background: var(--primary); color: white; border: none; padding: 8px 15px; border-radius: 5px; cursor: pointer; text-decoration: none; }
.reaction-bar { display: flex; gap: 10px; margin: 10px 0; font-size: 20px; }
.comment-section { border-top: 1px solid #333; padding-top: 10px; margin-top: 10px; font-size: 14px; }
.input-field { width: 100%; padding: 10px; margin: 10px 0; border-radius: 5px; border: 1px solid #333; background: #222; color: white; box-sizing: border-box; }
.badge { background: red; border-radius: 50%; padding: 2px 6px; font-size: 10px; }
"""

LAYOUT = """
<!DOCTYPE html>
<html>
<head>
    <title>Art Cinema</title>
    <style>""" + BASE_CSS + """</style>
</head>
<body>
    <div class="navbar">
        <a href="{{ url_for('index') }}">🎬 سينما الفن</a>
        <a href="{{ url_for('search') }}">🔍 بحث</a>
        {% if session.get('username') %}
            <a href="{{ url_for('notifications') }}">🔔 
                {% if unread_count %}<span class="badge">{{ unread_count }}</span>{% endif %}
            </a>
            <a href="{{ url_for('profile', username=session['username']) }}">👤 بروفيلي</a>
            <a href="{{ url_for('logout') }}">🚪 خروج</a>
        {% else %}
            <a href="{{ url_for('login') }}">تسجيل دخول</a>
        {% endif %}
    </div>
    <div class="container">
        {% with messages = get_flashed_messages() %}
          {% if messages %} {% for msg in messages %}<div class="card" style="border-color: yellow;">{{ msg }}</div>{% endfor %} {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </div>
</body>
</html>
"""

# ==================== المسارات (Routes) ====================

@app.context_processor
def inject_globals():
    user = get_current_user()
    unread_count = Notification.query.filter_by(user_to=user.username, is_read=False).count() if user else 0
    return dict(unread_count=unread_count, current_user=user)

@app.route('/')
def index():
    if 'username' not in session: return redirect(url_for('login'))
    
    # جلب القصص الصالحة (أقل من 24 ساعة)
    stories = Story.query.filter(Story.expires_at > datetime.utcnow()).order_by(Story.created_at.desc()).all()
    # جلب المنشورات
    posts = Post.query.order_by(Post.id.desc()).all()
    
    html = """
    {% extends "layout" %}
    {% block content %}
    <div class="story-container">
        <div class="story-card" style="display: flex; align-items: center; justify-content: center; background: #444;">
            <a href="{{ url_for('add_story') }}" style="color:white; text-decoration:none; font-size: 30px;">+</a>
        </div>
        {% for story in stories %}
        <div class="story-card">
            {% if story.image_filename %}
                <img src="{{ url_for('uploaded_file', filename=story.image_filename) }}">
            {% endif %}
            <div class="story-author">{{ story.author }}</div>
        </div>
        {% endfor %}
    </div>

    <div class="card">
        <form action="{{ url_for('add_post') }}" method="post" enctype="multipart/form-data">
            <textarea name="text" class="input-field" placeholder="بماذا تفكر في عالم السينما؟" required></textarea>
            <input type="text" name="hashtag" class="input-field" placeholder="هاشتاق (مثلاً: #نقد_فيلم)">
            <input type="file" name="image" accept="image/*">
            <button type="submit" class="btn">نشر</button>
        </form>
    </div>

    {% for post in posts %}
    <div class="card">
        <strong><a href="{{ url_for('profile', username=post.author) }}" style="color: var(--primary);">@{{ post.author }}</a></strong>
        <p>{{ post.text }}</p>
        {% if post.hashtag %}<small style="color: cyan;">{{ post.hashtag }}</small>{% endif %}
        
        {% if post.image_filename %}
            <img src="{{ url_for('uploaded_file', filename=post.image_filename) }}" style="width:100%; border-radius: 8px; margin-top:10px;">
        {% endif %}

        <div class="reaction-bar">
            <a href="{{ url_for('react', post_id=post.id, type='❤️') }}" style="text-decoration:none;">❤️ {{ post.reactions|selectattr('reaction_type', 'equalto', '❤️')|list|length }}</a>
            <a href="{{ url_for('react', post_id=post.id, type='👏') }}" style="text-decoration:none;">👏</a>
            <a href="{{ url_for('bookmark', post_id=post.id) }}" style="text-decoration:none; margin-right: auto;">🔖</a>
        </div>

        <div class="comment-section">
            {% for comment in post.comments %}
                <div><strong>{{ comment.author }}:</strong> {{ comment.text }}</div>
            {% endfor %}
            <form action="{{ url_for('add_comment', post_id=post.id) }}" method="post" style="display:flex; margin-top:10px;">
                <input type="text" name="comment_text" class="input-field" placeholder="أضف تعليقاً..." required>
                <button type="submit" class="btn" style="margin-right:5px;">ارسال</button>
            </form>
        </div>
    </div>
    {% endfor %}
    {% endblock %}
    """
    return render_template_string(html, stories=stories, posts=posts, layout=LAYOUT)

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username'].strip().lower()
        password = request.form['password']
        if User.query.filter_by(username=username).first():
            flash("اسم المستخدم موجود مسبقاً")
        else:
            new_user = User(username=username)
            new_user.set_password(password)
            db.session.add(new_user)
            db.session.commit()
            flash("تم التسجيل بنجاح! سجل دخولك الآن.")
            return redirect(url_for('login'))
    return render_template_string("""{% extends "layout" %} {% block content %} 
    <div class="card">
        <h2>إنشاء حساب جديد</h2>
        <form method="post">
            <input name="username" class="input-field" placeholder="اسم المستخدم" required>
            <input type="password" name="password" class="input-field" placeholder="كلمة المرور" required>
            <button type="submit" class="btn">تسجيل</button>
        </form>
    </div>{% endblock %}""", layout=LAYOUT)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        user = User.query.filter_by(username=request.form['username'].lower()).first()
        if user and user.check_password(request.form['password']):
            if user.is_banned:
                flash("هذا الحساب محظور.")
                return redirect(url_for('login'))
            session['username'] = user.username
            return redirect(url_for('index'))
        flash("خطأ في البيانات")
    return render_template_string("""{% extends "layout" %} {% block content %}
    <div class="card">
        <h2>تسجيل الدخول</h2>
        <form method="post">
            <input name="username" class="input-field" placeholder="اسم المستخدم" required>
            <input type="password" name="password" class="input-field" placeholder="كلمة المرور" required>
            <button type="submit" class="btn">دخول</button>
        </form>
        <p>ليس لديك حساب؟ <a href="{{ url_for('register') }}">سجل هنا</a></p>
    </div>{% endblock %}""", layout=LAYOUT)

@app.route('/logout')
def logout():
    session.pop('username', None)
    return redirect(url_for('login'))

@app.route('/add_post', methods=['POST'])
def add_post():
    if 'username' not in session: return redirect(url_for('login'))
    text = request.form.get('text')
    hashtag = request.form.get('hashtag')
    file = request.files.get('image')
    filename = None
    if file and allowed_file(file.filename, ALLOWED_IMAGE_EXT):
        filename = secure_filename(f"{datetime.now().timestamp()}_{file.filename}")
        file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
    
    new_post = Post(author=session['username'], text=text, hashtag=hashtag, image_filename=filename)
    db.session.add(new_post)
    db.session.commit()
    return redirect(url_for('index'))

@app.route('/add_story', methods=['GET', 'POST'])
def add_story():
    if 'username' not in session: return redirect(url_for('login'))
    if request.method == 'POST':
        file = request.files.get('image')
        text = request.form.get('text')
        filename = None
        if file and allowed_file(file.filename, ALLOWED_IMAGE_EXT):
            filename = secure_filename(f"story_{datetime.now().timestamp()}_{file.filename}")
            file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
        new_story = Story(author=session['username'], text=text, image_filename=filename)
        db.session.add(new_story)
        db.session.commit()
        return redirect(url_for('index'))
    return render_template_string("""{% extends "layout" %} {% block content %}
    <div class="card">
        <h2>إضافة قصة (Story)</h2>
        <form method="post" enctype="multipart/form-data">
            <input type="file" name="image" required>
            <input name="text" class="input-field" placeholder="كلمة قصيرة...">
            <button type="submit" class="btn">نشر الستوري</button>
        </form>
    </div>{% endblock %}""", layout=LAYOUT)

@app.route('/react/<int:post_id>/<type>')
def react(post_id, type):
    if 'username' not in session: return redirect(url_for('login'))
    post = Post.query.get_or_404(post_id)
    existing = PostReaction.query.filter_by(post_id=post_id, username=session['username']).first()
    if existing:
        db.session.delete(existing)
    else:
        new_reaction = PostReaction(post_id=post_id, username=session['username'], reaction_type=type)
        db.session.add(new_reaction)
        add_notification(post.author, session['username'], f"تفاعل مع منشورك بـ {type}")
    db.session.commit()
    return redirect(request.referrer or url_for('index'))

@app.route('/comment/<int:post_id>', methods=['POST'])
def add_comment(post_id):
    if 'username' not in session: return redirect(url_for('login'))
    post = Post.query.get_or_404(post_id)
    text = request.form.get('comment_text')
    new_comment = Comment(post_id=post_id, author=session['username'], text=text)
    db.session.add(new_comment)
    add_notification(post.author, session['username'], "علق على منشورك")
    db.session.commit()
    return redirect(request.referrer or url_for('index'))

@app.route('/profile/<username>')
def profile(username):
    user = User.query.filter_by(username=username).first_or_404()
    posts = Post.query.filter_by(author=username).order_by(Post.id.desc()).all()
    is_following = False
    if 'username' in session:
        curr = get_current_user()
        is_following = user in curr.followed
    return render_template_string("""{% extends "layout" %} {% block content %}
    <div class="card" style="text-align:center;">
        <h2>{{ user.username }}</h2>
        <p>{{ user.bio }}</p>
        <p>المتابعون: {{ user.followers.count() }} | يتابع: {{ user.followed.count() }}</p>
        {% if session.get('username') != user.username %}
            <a href="{{ url_for('follow', username=user.username) }}" class="btn">
                {{ 'إلغاء المتابعة' if is_following else 'متابعة' }}
            </a>
        {% endif %}
    </div>
    <h3>منشورات المستخدم</h3>
    {% for post in posts %}<div class="card">{{ post.text }}</div>{% endfor %}
    {% endblock %}""", user=user, posts=posts, is_following=is_following, layout=LAYOUT)

@app.route('/follow/<username>')
def follow(username):
    if 'username' not in session: return redirect(url_for('login'))
    user_to_follow = User.query.filter_by(username=username).first_or_404()
    me = get_current_user()
    if user_to_follow in me.followed:
        me.followed.remove(user_to_follow)
    else:
        me.followed.append(user_to_follow)
        add_notification(username, me.username, "بدأ بمتابعتك")
    db.session.commit()
    return redirect(request.referrer or url_for('profile', username=username))

@app.route('/bookmark/<int:post_id>')
def bookmark(post_id):
    if 'username' not in session: return redirect(url_for('login'))
    post = Post.query.get_or_404(post_id)
    me = get_current_user()
    me.toggle_bookmark(post)
    db.session.commit()
    flash("تم تحديث المحفوظات")
    return redirect(request.referrer or url_for('index'))

@app.route('/notifications')
def notifications():
    if 'username' not in session: return redirect(url_for('login'))
    notes = Notification.query.filter_by(user_to=session['username']).order_of(Notification.id.desc()).limit(20).all()
    for n in notes: n.is_read = True
    db.session.commit()
    return render_template_string("""{% extends "layout" %} {% block content %}
    <h2>الإشعارات</h2>
    {% for n in notes %}
        <div class="card" style="{{ 'opacity: 0.6;' if n.is_read else '' }}">
            <strong>{{ n.user_from }}</strong> {{ n.message }}
            <br><small>{{ n.created_at.strftime('%Y-%m-%d %H:%M') }}</small>
        </div>
    {% endfor %}
    {% endblock %}""", notes=notes, layout=LAYOUT)

@app.route('/search')
def search():
    query = request.args.get('q', '')
    users = User.query.filter(User.username.contains(query)).all() if query else []
    posts = Post.query.filter(Post.hashtag.contains(query)).all() if query else []
    return render_template_string("""{% extends "layout" %} {% block content %}
    <form action="/search" method="get">
        <input name="q" class="input-field" placeholder="ابحث عن مستخدم أو هاشتاق..." value="{{ query }}">
        <button type="submit" class="btn">بحث</button>
    </form>
    {% if query %}
        <h4>المستخدمون:</h4>
        {% for u in users %} <div><a href="/profile/{{ u.username }}">@{{ u.username }}</a></div> {% endfor %}
        <h4>الهاشتاقات:</h4>
        {% for p in posts %} <div class="card">{{ p.text }} ({{ p.hashtag }})</div> {% endfor %}
    {% endif %}
    {% endblock %}""", users=users, posts=posts, query=query, layout=LAYOUT)

@app.route('/uploads/<filename>')
def uploaded_file(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)

if __name__ == '__main__':
    app.run(debug=True)
