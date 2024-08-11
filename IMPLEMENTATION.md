The code I provided earlier includes a basic user registration route, but it does not fully implement a user sign-in and login feature. To add user authentication, including sign-in and login, we need to expand the application with the following components:

1. **User Registration**: A route to register new users.
2. **User Login**: A route to allow existing users to log in.
3. **Password Hashing**: Securely storing user passwords using hashing.
4. **Session Management**: Managing user sessions to keep them logged in across requests.
5. **Protected Routes**: Routes that require users to be authenticated.

I'll add these features to the existing application below.

### **1. `models.py` (User Model Update)**
```python
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password = db.Column(db.String(128), nullable=False)

class Event(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(500))
    date = db.Column(db.DateTime, nullable=False)

class RSVP(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    event_id = db.Column(db.Integer, db.ForeignKey('event.id'), nullable=False)
    user_name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(120), nullable=False)  
    response = db.Column(db.String(10), nullable=False)
    event = db.relationship('Event', backref=db.backref('rsvps', lazy=True))
```

### **2. `routes.py` (User Authentication)**
```python
from flask import Flask, request, jsonify, session
from flask_mail import Mail, Message
from flask_socketio import SocketIO
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
from models import db, User, Event, RSVP

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///events.db'
app.config['SECRET_KEY'] = 'your-secret-key'  # For session management
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = 'your-email@gmail.com'
app.config['MAIL_PASSWORD'] = 'your-email-password'

db.init_app(app)
mail = Mail(app)
socketio = SocketIO(app)

# User Registration Route
@app.route('/register', methods=['POST'])
def register():
    data = request.json
    hashed_password = generate_password_hash(data['password'], method='sha256')
    new_user = User(email=data['email'], password=hashed_password)
    db.session.add(new_user)
    db.session.commit()
    return jsonify({'message': 'User registered successfully'}), 201

# User Login Route
@app.route('/login', methods=['POST'])
def login():
    data = request.json
    user = User.query.filter_by(email=data['email']).first()
    if not user or not check_password_hash(user.password, data['password']):
        return jsonify({'message': 'Login failed! Check your credentials and try again.'}), 401
    
    session['user_id'] = user.id
    return jsonify({'message': 'Login successful!'}), 200

# Protected Route Example
@app.route('/protected', methods=['GET'])
def protected():
    if 'user_id' not in session:
        return jsonify({'message': 'You are not logged in!'}), 401

    return jsonify({'message': f'Welcome User {session["user_id"]}!'}), 200

@app.route('/logout', methods=['POST'])
def logout():
    session.pop('user_id', None)
    return jsonify({'message': 'You have logged out successfully!'}), 200

# RSVP Route with User Session
@app.route('/rsvp', methods=['POST'])
def rsvp():
    if 'user_id' not in session:
        return jsonify({'message': 'You must be logged in to RSVP!'}), 401

    data = request.json
    new_rsvp = RSVP(event_id=data['event_id'], user_name=data['user_name'], email=data['email'], response=data['response'])
    db.session.add(new_rsvp)
    db.session.commit()

    # Send email notification
    send_rsvp_notification(data['event_id'], data['user_name'], data['email'], data['response'])

    socketio.emit('rsvp_update', {'event_id': data['event_id'], 'user_name': data['user_name'], 'response': data['response']})
    return jsonify({'message': 'RSVP recorded'}), 201

def send_rsvp_notification(event_id, user_name, email, response):
    msg = Message(f"RSVP Update for Event {event_id}", sender="your-email@gmail.com", recipients=[email])
    msg.body = f"Hello {user_name},\n\nYou have successfully RSVP'd with '{response}' for Event ID: {event_id}.\n\nThank you!"
    mail.send(msg)

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    socketio.run(app, debug=True)
```

### **3. Frontend (`login_form.js`)**
```javascript
import React, { useState } from 'react';
import axios from 'axios';

function LoginForm() {
  const [formData, setFormData] = useState({ email: '', password: '' });

  const handleChange = e => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleSubmit = e => {
    e.preventDefault();
    axios.post('/login', formData)
      .then(response => console.log(response.data))
      .catch(error => console.log(error));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" name="email" placeholder="Email" onChange={handleChange} required />
      <input type="password" name="password" placeholder="Password" onChange={handleChange} required />
      <button type="submit">Login</button>
    </form>
  );
}

export default LoginForm;
```

### **4. Frontend (`register_form.js`)**
```javascript
import React, { useState } from 'react';
import axios from 'axios';

function RegisterForm() {
  const [formData, setFormData] = useState({ email: '', password: '' });

  const handleChange = e => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleSubmit = e => {
    e.preventDefault();
    axios.post('/register', formData)
      .then(response => console.log(response.data))
      .catch(error => console.log(error));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" name="email" placeholder="Email" onChange={handleChange} required />
      <input type="password" name="password" placeholder="Password" onChange={handleChange} required />
      <button type="submit">Register</button>
    </form>
  );
}

export default RegisterForm;
```

### **5. `requirements.txt`**
```text
Flask==2.1.2
Flask-SQLAlchemy==2.5.1
Flask-SocketIO==5.2.0
Flask-Mail==0.9.1
Flask-Bcrypt==0.7.1
Flask-Session==0.4.0  # Add this for session management
eventlet==0.33.0
```

### **Summary of Features Added:**
1. **User Registration**: Users can create accounts with an email and password.
2. **User Login**: Users can log in to their accounts.
3. **Session Management**: Users stay logged in across requests, enabling access to protected routes.
4. **Protected Routes**: Certain routes (like RSVP) require users to be logged in.

With these additions, your Real-Time Event Management System now supports full user authentication, allowing users to register, log in, and securely RSVP to events.
