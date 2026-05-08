# python
code for making star
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_cors import CORS
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime
import jwt
import os

# =========================================
# VibeOn Backend Server
# Your Vibe. Your Space.
# =========================================

app = Flask(__name__)
CORS(app)

# =========================================
# Configuration
# =========================================

app.config['SECRET_KEY'] = 'vibeon_secret_key_2026'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///vibeon.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

UPLOAD_FOLDER = 'uploads'
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

# =========================================
# Database
# =========================================

db = SQLAlchemy(app)

# =========================================
# Models
# =========================================

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), unique=True)
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(300))
    bio = db.Column(db.String(300), default='')
    profile_image = db.Column(db.String(300), default='')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    caption = db.Column(db.String(500))
    image_url = db.Column(db.String(500))
    likes = db.Column(db.Integer, default=0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


class Comment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer)
    post_id = db.Column(db.Integer)
    text = db.Column(db.String(500))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


# =========================================
# Create Database
# =========================================

with app.app_context():
    db.create_all()


# =========================================
# Home Route
# =========================================

@app.route('/')
def home():
    return jsonify({
        'app': 'VibeOn',
        'tagline': 'Your Vibe. Your Space.',
        'status': 'Running Successfully'
    })


# =========================================
# Register User
# =========================================

@app.route('/register', methods=['POST'])
def register():
    data = request.json

    username = data.get('username')
    email = data.get('email')
    password = data.get('password')

    existing_user = User.query.filter_by(email=email).first()

    if existing_user:
        return jsonify({'message': 'User already exists'}), 400

    hashed_password = generate_password_hash(password)

    new_user = User(
        username=username,
        email=email,
        password=hashed_password
    )

    db.session.add(new_user)
    db.session.commit()

    return jsonify({'message': 'User registered successfully'})


# =========================================
# Login User
# =========================================

@app.route('/login', methods=['POST'])
def login():
    data = request.json

    email = data.get('email')
    password = data.get('password')

    user = User.query.filter_by(email=email).first()

    if not user:
        return jsonify({'message': 'User not found'}), 404

    if not check_password_hash(user.password, password):
        return jsonify({'message': 'Wrong password'}), 401

    token = jwt.encode(
        {
            'user_id': user.id,
            'username': user.username
        },
        app.config['SECRET_KEY'],
        algorithm='HS256'
    )

    return jsonify({
        'message': 'Login successful',
        'token': token,
        'username': user.username
    })


# =========================================
# Create Post
# =========================================

@app.route('/create-post', methods=['POST'])
def create_post():
    data = request.json

    user_id = data.get('user_id')
    caption = data.get('caption')
    image_url = data.get('image_url')

    post = Post(
        user_id=user_id,
        caption=caption,
        image_url=image_url
    )

    db.session.add(post)
    db.session.commit()

    return jsonify({'message': 'Post uploaded successfully'})


# =========================================
# Get All Posts
# =========================================

@app.route('/posts', methods=['GET'])
def get_posts():
    posts = Post.query.order_by(Post.created_at.desc()).all()

    output = []

    for post in posts:
        user = User.query.get(post.user_id)

        output.append({
            'id': post.id,
            'username': user.username,
            'caption': post.caption,
            'image_url': post.image_url,
            'likes': post.likes,
            'created_at': post.created_at
        })

    return jsonify(output)


# =========================================
# Like Post
# =========================================

@app.route('/like-post/<int:post_id>', methods=['POST'])
def like_post(post_id):
    post = Post.query.get(post_id)

    if not post:
        return jsonify({'message': 'Post not found'}), 404

    post.likes += 1
    db.session.commit()

    return jsonify({'message': 'Post liked'})


# =========================================
# Add Comment
# =========================================

@app.route('/comment', methods=['POST'])
def add_comment():
    data = request.json

    comment = Comment(
        user_id=data.get('user_id'),
        post_id=data.get('post_id'),
        text=data.get('text')
    )

    db.session.add(comment)
    db.session.commit()

    return jsonify({'message': 'Comment added'})


# =========================================
# Get Comments
# =========================================

@app.route('/comments/<int:post_id>', methods=['GET'])
def get_comments(post_id):
    comments = Comment.query.filter_by(post_id=post_id).all()

    output = []

    for comment in comments:
        user = User.query.get(comment.user_id)

        output.append({
            'username': user.username,
            'text': comment.text,
            'created_at': comment.created_at
        })

    return jsonify(output)


# =========================================
# Profile Route
# =========================================

@app.route('/profile/<username>', methods=['GET'])
def profile(username):
    user = User.query.filter_by(username=username).first()

    if not user:
        return jsonify({'message': 'User not found'}), 404

    posts = Post.query.filter_by(user_id=user.id).all()

    return jsonify({
        'username': user.username,
        'bio': user.bio,
        'profile_image': user.profile_image,
        'posts_count': len(posts)
    })


# =========================================
# Search Users
# =========================================

@app.route('/search')
def search_users():
    query = request.args.get('q')

    users = User.query.filter(User.username.contains(query)).all()

    result = []

    for user in users:
        result.append({
            'id': user.id,
            'username': user.username
        })

    return jsonify(result)


# =========================================
# Delete Post
# =========================================

@app.route('/delete-post/<int:post_id>', methods=['DELETE'])
def delete_post(post_id):
    post = Post.query.get(post_id)

    if not post:
        return jsonify({'message': 'Post not found'}), 404

    db.session.delete(post)
    db.session.commit()

    return jsonify({'message': 'Post deleted'})


# =========================================
# Trending Posts
# =========================================

@app.route('/trending', methods=['GET'])
def trending_posts():
    posts = Post.query.order_by(Post.likes.desc()).limit(10).all()

    result = []

    for post in posts:
        user = User.query.get(post.user_id)

        result.append({
            'username': user.username,
            'caption': post.caption,
            'image_url': post.image_url,
            'likes': post.likes
        })

    return jsonify(result)


# =========================================
# Privacy / Ghost Mode Example
# =========================================

@app.route('/ghost-mode/<username>', methods=['POST'])
def ghost_mode(username):
    return jsonify({
        'message': f'Ghost mode enabled for {username}'
    })


# =========================================
# Run Server
# =========================================

if __name__ == '__main__':
    app.run(debug=True)

# =========================================
# INSTALLATION
# =========================================

# pip install flask
# pip install flask_sqlalchemy
# pip install flask_cors
# pip install pyjwt
# pip install werkzeug

# =========================================
# RUN PROJECT
# =========================================

# python app.py

# =========================================
# DEPLOYMENT
# =========================================

# Upload on:
# https://render.com
# https://railway.app
# https://vercel.com
# https://pythonanywhere.com

# =========================================
# FUTURE FEATURES
# =========================================

# - Reels
# - Stories
# - Messaging
# - Notifications
# - AI captions
# - Voice chat
# - Live streaming
# - Dark/light themes
# - Referral rewards
# - Creator monetization
