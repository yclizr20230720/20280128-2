# 📚 Flask API Development - Advanced Training

## Advanced Backend Development with Flask

**Target Audience:** Experienced Backend Developers  
**Prerequisites:** Flask Basics, Python OOP, HTTP/REST APIs  
**Learning Time:** 3-4 days  
**Difficulty:** Advanced

---

## Table of Contents
1. [JWT Authentication](#jwt-authentication)
2. [Decorators and Middleware](#decorators-and-middleware)
3. [Database Integration](#database-integration)
4. [Service Layer Pattern](#service-layer-pattern)
5. [Advanced Error Handling](#advanced-error-handling)
6. [File Processing](#file-processing)
7. [API Versioning](#api-versioning)
8. [Security Best Practices](#security-best-practices)
9. [Performance Optimization](#performance-optimization)
10. [Testing Strategies](#testing-strategies)
11. [Deployment](#deployment)
12. [Monitoring and Logging](#monitoring-and-logging)

---

## JWT Authentication

### What is JWT?

**JSON Web Token (JWT)** is a compact, URL-safe token format for securely transmitting information between parties.

**Structure:**
```
header.payload.signature
```

**Example:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIn0.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### JWT Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Auth
    
    Client->>API: POST /login (username, password)
    API->>Auth: Verify credentials
    Auth-->>API: User data
    API->>API: Generate JWT token
    API-->>Client: Return token
    
    Client->>API: GET /protected (Authorization: Bearer token)
    API->>API: Verify token
    API-->>Client: Protected data
```


### Implementing JWT Authentication

**Install PyJWT:**
```bash
pip install PyJWT
```

### Token Generation

```python
# app/utils/auth.py
import jwt
import os
from datetime import datetime, timedelta

SECRET_KEY = os.getenv('JWT_SECRET_KEY', 'your-secret-key')
ALGORITHM = 'HS256'
TOKEN_EXPIRATION_HOURS = 24

def generate_token(user_data):
    """Generate JWT token for user"""
    payload = {
        'username': user_data['username'],
        'email': user_data['email'],
        'role': user_data['role'],
        'exp': datetime.utcnow() + timedelta(hours=TOKEN_EXPIRATION_HOURS),
        'iat': datetime.utcnow()  # Issued at
    }
    
    token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    return token
```

**Understanding the Payload:**

- **username, email, role:** User information
- **exp:** Expiration time (24 hours from now)
- **iat:** Issued at time (current time)
- **SECRET_KEY:** Used to sign the token (keep secret!)
- **ALGORITHM:** HS256 (HMAC with SHA-256)

### Token Verification

```python
def decode_token(token):
    """Decode and verify JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload, None
    except jwt.ExpiredSignatureError:
        return None, "Token has expired"
    except jwt.InvalidTokenError:
        return None, "Invalid token"
```

**What Can Go Wrong:**

1. **ExpiredSignatureError:** Token is too old
2. **InvalidTokenError:** Token is malformed or signature doesn't match
3. **DecodeError:** Token format is invalid

### Login Endpoint

```python
# app/routes/auth.py
from flask import Blueprint, request, jsonify
from app.models.user import UserModel
from app.utils.auth import generate_token

auth_bp = Blueprint('auth', __name__)
user_model = UserModel()

@auth_bp.route('/login', methods=['POST'])
def login():
    """Login user and return JWT token"""
    try:
        data = request.get_json()
        
        # Validate input
        if 'username' not in data or 'password' not in data:
            return jsonify({
                'error': 'Username and password are required'
            }), 400
        
        username = data['username'].strip()
        password = data['password']
        
        # Authenticate user
        user_data, error = user_model.authenticate(username, password)
        
        if error:
            return jsonify({'error': error}), 401
        
        # Generate JWT token
        token = generate_token(user_data)
        
        return jsonify({
            'success': True,
            'message': 'Login successful',
            'user': user_data,
            'token': token
        }), 200
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

---

## Decorators and Middleware

### Authentication Decorator

```python
# app/utils/auth.py
from functools import wraps
from flask import request, jsonify

def token_required(f):
    """Decorator to protect routes with JWT authentication"""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = None
        
        # Get token from Authorization header
        if 'Authorization' in request.headers:
            auth_header = request.headers['Authorization']
            try:
                # Format: "Bearer <token>"
                token = auth_header.split(' ')[1]
            except IndexError:
                return jsonify({'error': 'Invalid token format'}), 401
        
        if not token:
            return jsonify({'error': 'Token is missing'}), 401
        
        # Decode and verify token
        payload, error = decode_token(token)
        
        if error:
            return jsonify({'error': error}), 401
        
        # Add user info to request object
        request.user = payload
        
        # Call the original function
        return f(*args, **kwargs)
    
    return decorated
```

**How It Works:**

1. **@wraps(f):** Preserves original function metadata
2. **Extract token:** From Authorization header
3. **Verify token:** Decode and check validity
4. **Add user to request:** Make user data available in route
5. **Call original function:** If everything is valid

### Using the Decorator

```python
@auth_bp.route('/me', methods=['GET'])
@token_required
def get_current_user():
    """Get current user info (protected route)"""
    try:
        # Access user data from request.user
        username = request.user['username']
        user_data = user_model.get_user(username)
        
        if not user_data:
            return jsonify({'error': 'User not found'}), 404
        
        return jsonify({
            'success': True,
            'user': user_data
        }), 200
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

### Role-Based Access Control

```python
def admin_required(f):
    """Decorator to protect routes requiring admin role"""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = None
        
        # Get token from header
        if 'Authorization' in request.headers:
            auth_header = request.headers['Authorization']
            try:
                token = auth_header.split(' ')[1]
            except IndexError:
                return jsonify({'error': 'Invalid token format'}), 401
        
        if not token:
            return jsonify({'error': 'Token is missing'}), 401
        
        # Decode token
        payload, error = decode_token(token)
        
        if error:
            return jsonify({'error': error}), 401
        
        # Check if user is admin
        if payload.get('role') != 'admin':
            return jsonify({'error': 'Admin access required'}), 403
        
        # Add user info to request
        request.user = payload
        
        return f(*args, **kwargs)
    
    return decorated


# Usage
@auth_bp.route('/users', methods=['GET'])
@admin_required
def get_all_users():
    """Get all users (admin only)"""
    try:
        users = user_model.get_all_users()
        
        return jsonify({
            'success': True,
            'users': users,
            'count': len(users)
        }), 200
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

### Custom Decorators

```python
def validate_json(*expected_args):
    """Decorator to validate JSON request data"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            data = request.get_json()
            
            if not data:
                return jsonify({'error': 'No JSON data provided'}), 400
            
            # Check for required fields
            missing = [arg for arg in expected_args if arg not in data]
            if missing:
                return jsonify({
                    'error': f'Missing required fields: {", ".join(missing)}'
                }), 400
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator


# Usage
@app.route('/api/users', methods=['POST'])
@validate_json('username', 'email', 'password')
def create_user():
    data = request.get_json()
    # All required fields are guaranteed to exist
    username = data['username']
    email = data['email']
    password = data['password']
    # ...
```

### Request Logging Decorator

```python
import time
from functools import wraps

def log_request(f):
    """Decorator to log API requests"""
    @wraps(f)
    def decorated(*args, **kwargs):
        start_time = time.time()
        
        # Log request
        print(f"[{datetime.now()}] {request.method} {request.path}")
        print(f"Headers: {dict(request.headers)}")
        print(f"Body: {request.get_json()}")
        
        # Execute function
        response = f(*args, **kwargs)
        
        # Log response time
        duration = time.time() - start_time
        print(f"Response time: {duration:.3f}s")
        
        return response
    return decorated


# Usage
@app.route('/api/data', methods=['POST'])
@log_request
@token_required
def process_data():
    # Function implementation
    pass
```

---

## Database Integration

### User Model with JSON Storage

Our project uses JSON file storage for simplicity. Here's the complete implementation:

```python
# app/models/user.py
import json
import os
from datetime import datetime
import bcrypt

class UserModel:
    def __init__(self, users_file='users.json'):
        # Get the directory where this file is located
        base_dir = os.path.dirname(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
        self.users_file = os.path.join(base_dir, users_file)
        self._ensure_file_exists()
    
    def _ensure_file_exists(self):
        """Create users file if it doesn't exist"""
        if not os.path.exists(self.users_file):
            # Create default admin user
            default_users = {
                "admin": {
                    "username": "admin",
                    "email": "admin@vsmc.com",
                    "password": self._hash_password("admin123"),
                    "role": "admin",
                    "created_at": datetime.now().isoformat(),
                    "last_login": None
                }
            }
            with open(self.users_file, 'w') as f:
                json.dump(default_users, f, indent=2)
    
    def _hash_password(self, password):
        """Hash password using bcrypt"""
        return bcrypt.hashpw(
            password.encode('utf-8'),
            bcrypt.gensalt()
        ).decode('utf-8')
    
    def _verify_password(self, password, hashed):
        """Verify password against hash"""
        return bcrypt.checkpw(
            password.encode('utf-8'),
            hashed.encode('utf-8')
        )
    
    def _load_users(self):
        """Load users from file"""
        with open(self.users_file, 'r') as f:
            return json.load(f)
    
    def _save_users(self, users):
        """Save users to file"""
        with open(self.users_file, 'w') as f:
            json.dump(users, f, indent=2)
    
    def create_user(self, username, email, password, role='user'):
        """Create a new user"""
        users = self._load_users()
        
        # Check if user exists
        if username in users:
            return None, "Username already exists"
        
        # Check if email exists
        for user in users.values():
            if user['email'] == email:
                return None, "Email already exists"
        
        # Create user
        users[username] = {
            "username": username,
            "email": email,
            "password": self._hash_password(password),
            "role": role,
            "created_at": datetime.now().isoformat(),
            "last_login": None
        }
        
        self._save_users(users)
        
        # Return user without password
        user_data = users[username].copy()
        del user_data['password']
        return user_data, None
    
    def authenticate(self, username, password):
        """Authenticate user"""
        users = self._load_users()
        
        if username not in users:
            return None, "Invalid username or password"
        
        user = users[username]
        
        if not self._verify_password(password, user['password']):
            return None, "Invalid username or password"
        
        # Update last login
        users[username]['last_login'] = datetime.now().isoformat()
        self._save_users(users)
        
        # Return user without password
        user_data = user.copy()
        del user_data['password']
        return user_data, None
    
    def get_user(self, username):
        """Get user by username"""
        users = self._load_users()
        
        if username not in users:
            return None
        
        user_data = users[username].copy()
        del user_data['password']
        return user_data
    
    def get_all_users(self):
        """Get all users (admin only)"""
        users = self._load_users()
        result = []
        
        for user in users.values():
            user_data = user.copy()
            del user_data['password']
            result.append(user_data)
        
        return result
    
    def update_user(self, username, **kwargs):
        """Update user information"""
        users = self._load_users()
        
        if username not in users:
            return None, "User not found"
        
        # Update allowed fields
        allowed_fields = ['email', 'role']
        for field in allowed_fields:
            if field in kwargs:
                users[username][field] = kwargs[field]
        
        # Update password if provided
        if 'password' in kwargs:
            users[username]['password'] = self._hash_password(kwargs['password'])
        
        self._save_users(users)
        
        user_data = users[username].copy()
        del user_data['password']
        return user_data, None
    
    def delete_user(self, username):
        """Delete user"""
        users = self._load_users()
        
        if username not in users:
            return False, "User not found"
        
        if username == 'admin':
            return False, "Cannot delete admin user"
        
        del users[username]
        self._save_users(users)
        
        return True, None
```

**Key Concepts:**

1. **Password Hashing:** Never store plain passwords
2. **bcrypt:** Industry-standard password hashing
3. **File-based Storage:** Simple but not scalable
4. **CRUD Operations:** Create, Read, Update, Delete
5. **Error Handling:** Return (data, error) tuple

### Password Security with bcrypt

```python
import bcrypt

# Hash password
password = "admin123"
hashed = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())
# Result: b'$2b$12$...'

# Verify password
is_valid = bcrypt.checkpw(password.encode('utf-8'), hashed)
# Result: True or False
```

**Why bcrypt?**

1. **Slow by design:** Makes brute-force attacks impractical
2. **Salt included:** Each hash is unique
3. **Adaptive:** Can increase cost factor over time
4. **Industry standard:** Widely trusted and tested


### SQLAlchemy Integration (Alternative)

For production applications, use a proper database:

```python
# Install
pip install Flask-SQLAlchemy

# config.py
class Config:
    SQLALCHEMY_DATABASE_URI = 'sqlite:///app.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False

# models/user.py
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)
    role = db.Column(db.String(20), default='user')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime)
    
    def set_password(self, password):
        self.password_hash = bcrypt.hashpw(
            password.encode('utf-8'),
            bcrypt.gensalt()
        ).decode('utf-8')
    
    def check_password(self, password):
        return bcrypt.checkpw(
            password.encode('utf-8'),
            self.password_hash.encode('utf-8')
        )
    
    def to_dict(self):
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'role': self.role,
            'created_at': self.created_at.isoformat(),
            'last_login': self.last_login.isoformat() if self.last_login else None
        }

# app/__init__.py
from models.user import db

def create_app():
    app = Flask(__name__)
    app.config.from_object(config['development'])
    
    # Initialize database
    db.init_app(app)
    
    with app.app_context():
        db.create_all()
    
    return app

# Usage in routes
@auth_bp.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    
    # Create user
    user = User(
        username=data['username'],
        email=data['email'],
        role='user'
    )
    user.set_password(data['password'])
    
    db.session.add(user)
    db.session.commit()
    
    return jsonify(user.to_dict()), 201

@auth_bp.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    
    # Find user
    user = User.query.filter_by(username=data['username']).first()
    
    if not user or not user.check_password(data['password']):
        return jsonify({'error': 'Invalid credentials'}), 401
    
    # Update last login
    user.last_login = datetime.utcnow()
    db.session.commit()
    
    # Generate token
    token = generate_token(user.to_dict())
    
    return jsonify({
        'success': True,
        'user': user.to_dict(),
        'token': token
    }), 200
```

---

## Service Layer Pattern

### Why Service Layer?

**Separation of Concerns:**
- **Routes:** Handle HTTP requests/responses
- **Services:** Business logic
- **Models:** Data access

**Benefits:**
1. Easier to test
2. Reusable logic
3. Cleaner code
4. Better organization

### Service Example: Bossung Analysis

```python
# app/services/bossung_service.py
import pandas as pd
import numpy as np
from datetime import datetime
import os

class BossungService:
    """Service for Bossung curve analysis"""
    
    def __init__(self):
        self.results_cache = {}
    
    def analyze(self, filepath, target_cd, tolerance_percent):
        """
        Analyze Bossung curves and calculate process window metrics
        
        Parameters:
        -----------
        filepath : str
            Path to CSV data file
        target_cd : float
            Target critical dimension
        tolerance_percent : float
            Tolerance percentage
        
        Returns:
        --------
        dict : Analysis results
        """
        try:
            # Load data
            data = pd.read_csv(filepath)
            
            # Validate columns
            required_cols = ['Dose', 'Defocus', 'CD']
            if not all(col in data.columns for col in required_cols):
                raise ValueError(f'Missing required columns: {required_cols}')
            
            # Calculate tolerance
            tolerance = target_cd * (tolerance_percent / 100)
            cd_min = target_cd - tolerance
            cd_max = target_cd + tolerance
            
            # Find best focus for each dose
            best_focus_data = []
            for dose in data['Dose'].unique():
                dose_data = data[data['Dose'] == dose]
                
                # Find CD closest to target
                dose_data['cd_diff'] = abs(dose_data['CD'] - target_cd)
                best_idx = dose_data['cd_diff'].idxmin()
                best_row = dose_data.loc[best_idx]
                
                best_focus_data.append({
                    'dose': dose,
                    'best_focus': best_row['Defocus'],
                    'cd_at_best_focus': best_row['CD']
                })
            
            # Calculate process window
            in_spec_data = data[
                (data['CD'] >= cd_min) & (data['CD'] <= cd_max)
            ]
            
            if len(in_spec_data) > 0:
                dof = in_spec_data['Defocus'].max() - in_spec_data['Defocus'].min()
                el = in_spec_data['Dose'].max() - in_spec_data['Dose'].min()
                yield_percent = (len(in_spec_data) / len(data)) * 100
            else:
                dof = 0
                el = 0
                yield_percent = 0
            
            # Compile results
            results = {
                'target_cd': target_cd,
                'tolerance': tolerance,
                'cd_range': {
                    'min': cd_min,
                    'max': cd_max
                },
                'best_focus_data': best_focus_data,
                'process_window': {
                    'dof': round(dof, 3),
                    'el': round(el, 3),
                    'yield_percent': round(yield_percent, 2)
                },
                'data_summary': {
                    'total_points': len(data),
                    'in_spec_points': len(in_spec_data),
                    'dose_range': {
                        'min': float(data['Dose'].min()),
                        'max': float(data['Dose'].max())
                    },
                    'defocus_range': {
                        'min': float(data['Defocus'].min()),
                        'max': float(data['Defocus'].max())
                    }
                }
            }
            
            return results
            
        except Exception as e:
            raise Exception(f'Analysis failed: {str(e)}')
    
    def get_chart_data(self, filepath, target_cd, tolerance_percent):
        """
        Get data formatted for frontend charts
        
        Returns:
        --------
        dict : Chart data for various visualizations
        """
        try:
            data = pd.read_csv(filepath)
            
            # Bossung curves data
            bossung_data = []
            for dose in sorted(data['Dose'].unique()):
                dose_data = data[data['Dose'] == dose].sort_values('Defocus')
                bossung_data.append({
                    'dose': float(dose),
                    'points': [
                        {
                            'defocus': float(row['Defocus']),
                            'cd': float(row['CD'])
                        }
                        for _, row in dose_data.iterrows()
                    ]
                })
            
            # Process window data
            tolerance = target_cd * (tolerance_percent / 100)
            cd_min = target_cd - tolerance
            cd_max = target_cd + tolerance
            
            process_window_data = {
                'target_cd': target_cd,
                'cd_min': cd_min,
                'cd_max': cd_max,
                'points': [
                    {
                        'dose': float(row['Dose']),
                        'defocus': float(row['Defocus']),
                        'cd': float(row['CD']),
                        'in_spec': cd_min <= row['CD'] <= cd_max
                    }
                    for _, row in data.iterrows()
                ]
            }
            
            return {
                'bossung_curves': bossung_data,
                'process_window': process_window_data
            }
            
        except Exception as e:
            raise Exception(f'Chart data generation failed: {str(e)}')
    
    def generate_mock_data(self, output_folder, doses=9, defocus_points=17, target_cd=45.0):
        """
        Generate mock lithography data for testing
        
        Returns:
        --------
        str : Path to generated CSV file
        """
        try:
            # Generate dose and defocus arrays
            dose_array = np.linspace(20, 28, doses)
            defocus_array = np.linspace(-0.4, 0.4, defocus_points)
            
            # Generate data
            data_rows = []
            for dose in dose_array:
                for defocus in defocus_array:
                    # Simulate Bossung curve (parabolic)
                    cd = target_cd + 5 * (defocus ** 2) + np.random.normal(0, 0.5)
                    # Add dose effect
                    cd = cd - (dose - 24) * 0.5
                    
                    data_rows.append({
                        'Dose': round(dose, 2),
                        'Defocus': round(defocus, 3),
                        'CD': round(cd, 2),
                        'Wafer_ID': 'W001',
                        'Field': 'F01'
                    })
            
            # Create DataFrame
            df = pd.DataFrame(data_rows)
            
            # Save to file
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            filename = f'lithography_data_{timestamp}.csv'
            filepath = os.path.join(output_folder, filename)
            df.to_csv(filepath, index=False)
            
            return filepath
            
        except Exception as e:
            raise Exception(f'Mock data generation failed: {str(e)}')
```

**Service Layer Benefits:**

1. **Testable:** Can test without HTTP layer
2. **Reusable:** Use in multiple routes or even CLI
3. **Maintainable:** Business logic in one place
4. **Cacheable:** Can add caching easily

### Using Services in Routes

```python
# app/routes/edforest.py
from flask import Blueprint, request, jsonify, current_app
from app.services.bossung_service import BossungService

edforest_bp = Blueprint('edforest', __name__)
bossung_service = BossungService()

@edforest_bp.route('/analyze', methods=['POST'])
def analyze_bossung():
    """Analyze Bossung curves"""
    try:
        data = request.get_json()
        
        if not data:
            return jsonify({'error': 'No data provided'}), 400
        
        filepath = data.get('filepath')
        target_cd = data.get('target_cd', 45.0)
        tolerance_percent = data.get('tolerance_percent', 10)
        
        if not filepath or not os.path.exists(filepath):
            return jsonify({'error': 'Invalid file path'}), 400
        
        # Call service method
        results = bossung_service.analyze(filepath, target_cd, tolerance_percent)
        chart_data = bossung_service.get_chart_data(filepath, target_cd, tolerance_percent)
        
        return jsonify({
            'success': True,
            'results': results,
            'chart_data': chart_data
        }), 200
        
    except Exception as e:
        return jsonify({'error': f'Analysis failed: {str(e)}'}), 500
```

**Route Responsibilities:**

1. **Validate HTTP input:** Check request format
2. **Call service:** Delegate business logic
3. **Format response:** Convert to JSON
4. **Handle errors:** Return appropriate status codes

---

## Advanced Error Handling

### Custom Exception Classes

```python
# app/utils/exceptions.py
class APIException(Exception):
    """Base exception for API errors"""
    status_code = 500
    
    def __init__(self, message, status_code=None, payload=None):
        super().__init__()
        self.message = message
        if status_code is not None:
            self.status_code = status_code
        self.payload = payload
    
    def to_dict(self):
        rv = dict(self.payload or ())
        rv['error'] = self.message
        rv['status'] = self.status_code
        return rv


class ValidationError(APIException):
    """Validation error (400)"""
    status_code = 400


class AuthenticationError(APIException):
    """Authentication error (401)"""
    status_code = 401


class AuthorizationError(APIException):
    """Authorization error (403)"""
    status_code = 403


class NotFoundError(APIException):
    """Resource not found (404)"""
    status_code = 404


class ConflictError(APIException):
    """Resource conflict (409)"""
    status_code = 409
```

### Global Exception Handler

```python
# app/__init__.py
from app.utils.exceptions import APIException

def create_app():
    app = Flask(__name__)
    
    # Register exception handler
    @app.errorhandler(APIException)
    def handle_api_exception(error):
        response = jsonify(error.to_dict())
        response.status_code = error.status_code
        return response
    
    # Handle unexpected errors
    @app.errorhandler(Exception)
    def handle_unexpected_error(error):
        # Log the error
        app.logger.error(f'Unexpected error: {str(error)}')
        
        # Return generic error message
        return jsonify({
            'error': 'An unexpected error occurred',
            'status': 500
        }), 500
    
    return app
```

### Using Custom Exceptions

```python
from app.utils.exceptions import ValidationError, NotFoundError, AuthenticationError

@auth_bp.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    
    # Validation
    if not data:
        raise ValidationError('No data provided')
    
    if 'username' not in data:
        raise ValidationError('Username is required')
    
    if 'password' not in data:
        raise ValidationError('Password is required')
    
    # Authentication
    user_data, error = user_model.authenticate(
        data['username'],
        data['password']
    )
    
    if error:
        raise AuthenticationError(error)
    
    # Generate token
    token = generate_token(user_data)
    
    return jsonify({
        'success': True,
        'user': user_data,
        'token': token
    }), 200

@auth_bp.route('/users/<username>', methods=['GET'])
@admin_required
def get_user(username):
    user_data = user_model.get_user(username)
    
    if not user_data:
        raise NotFoundError(f'User {username} not found')
    
    return jsonify({
        'success': True,
        'user': user_data
    }), 200
```


---

## File Processing

### Secure File Upload

```python
# app/utils/file_handler.py
import os
from werkzeug.utils import secure_filename
from datetime import datetime

def allowed_file(filename, allowed_extensions):
    """Check if file extension is allowed"""
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in allowed_extensions


def save_upload_file(file, filename, upload_folder):
    """Save uploaded file with timestamp"""
    # Add timestamp to prevent conflicts
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    name, ext = os.path.splitext(filename)
    new_filename = f"{name}_{timestamp}{ext}"
    
    # Secure the filename
    safe_filename = secure_filename(new_filename)
    filepath = os.path.join(upload_folder, safe_filename)
    
    # Save file
    file.save(filepath)
    
    return filepath


def cleanup_old_files(folder, max_age_hours=24):
    """Clean up old files from folder"""
    import time
    
    now = time.time()
    max_age_seconds = max_age_hours * 3600
    
    for filename in os.listdir(folder):
        filepath = os.path.join(folder, filename)
        if os.path.isfile(filepath):
            file_age = now - os.path.getmtime(filepath)
            if file_age > max_age_seconds:
                try:
                    os.remove(filepath)
                    print(f'Deleted old file: {filename}')
                except Exception as e:
                    print(f'Failed to delete {filename}: {str(e)}')
```

### CSV Processing with Pandas

```python
import pandas as pd

def process_csv_file(filepath):
    """Process CSV file and return data summary"""
    try:
        # Read CSV
        df = pd.read_csv(filepath)
        
        # Validate columns
        required_columns = ['Dose', 'Defocus', 'CD']
        missing = [col for col in required_columns if col not in df.columns]
        
        if missing:
            raise ValueError(f'Missing columns: {", ".join(missing)}')
        
        # Data validation
        if df.isnull().any().any():
            raise ValueError('Data contains null values')
        
        # Get summary
        summary = {
            'rows': len(df),
            'columns': list(df.columns),
            'dose_range': {
                'min': float(df['Dose'].min()),
                'max': float(df['Dose'].max()),
                'unique': int(df['Dose'].nunique())
            },
            'defocus_range': {
                'min': float(df['Defocus'].min()),
                'max': float(df['Defocus'].max()),
                'unique': int(df['Defocus'].nunique())
            },
            'cd_stats': {
                'min': float(df['CD'].min()),
                'max': float(df['CD'].max()),
                'mean': float(df['CD'].mean()),
                'std': float(df['CD'].std())
            }
        }
        
        return df, summary
        
    except Exception as e:
        raise Exception(f'CSV processing failed: {str(e)}')
```

### File Download

```python
from flask import send_file

@app.route('/download/<filename>')
def download_file(filename):
    """Download file"""
    try:
        # Secure filename
        safe_filename = secure_filename(filename)
        filepath = os.path.join(app.config['OUTPUT_FOLDER'], safe_filename)
        
        # Check if file exists
        if not os.path.exists(filepath):
            raise NotFoundError('File not found')
        
        # Send file
        return send_file(
            filepath,
            as_attachment=True,
            download_name=safe_filename
        )
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

---

## API Versioning

### URL-Based Versioning

```python
# config.py
class Config:
    API_VERSION = 'v1'
    API_PREFIX = f'/api/{API_VERSION}'

# app/__init__.py
def create_app():
    app = Flask(__name__)
    
    api_prefix = app.config['API_PREFIX']
    
    # Register v1 blueprints
    app.register_blueprint(auth_bp, url_prefix=f'{api_prefix}/auth')
    app.register_blueprint(edforest_bp, url_prefix=f'{api_prefix}/edforest')
    
    return app
```

**Result:**
- `/api/v1/auth/login`
- `/api/v1/edforest/upload`

**Adding v2:**
```python
# app/routes/v2/auth.py
auth_v2_bp = Blueprint('auth_v2', __name__)

@auth_v2_bp.route('/login', methods=['POST'])
def login_v2():
    # New implementation with different features
    pass

# app/__init__.py
app.register_blueprint(auth_bp, url_prefix='/api/v1/auth')
app.register_blueprint(auth_v2_bp, url_prefix='/api/v2/auth')
```

### Header-Based Versioning

```python
from flask import request

@app.before_request
def check_api_version():
    """Check API version from header"""
    version = request.headers.get('API-Version', 'v1')
    
    if version not in ['v1', 'v2']:
        return jsonify({'error': 'Invalid API version'}), 400
    
    request.api_version = version
```

---

## Security Best Practices

### 1. Environment Variables

```python
# .env file
SECRET_KEY=your-secret-key-here
JWT_SECRET_KEY=your-jwt-secret-here
DATABASE_URL=postgresql://user:pass@localhost/db

# Load in application
from dotenv import load_dotenv
import os

load_dotenv()

SECRET_KEY = os.getenv('SECRET_KEY')
JWT_SECRET_KEY = os.getenv('JWT_SECRET_KEY')
```

### 2. CORS Configuration

```python
from flask_cors import CORS

# Restrictive CORS
CORS(app, resources={
    r"/api/*": {
        "origins": ["https://yourdomain.com"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "allow_headers": ["Content-Type", "Authorization"],
        "max_age": 3600
    }
})
```

### 3. Rate Limiting

```python
# Install
pip install Flask-Limiter

# Usage
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/api/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    # Login logic
    pass
```

### 4. Input Validation

```python
def validate_email(email):
    """Validate email format"""
    import re
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def validate_password(password):
    """Validate password strength"""
    if len(password) < 8:
        return False, "Password must be at least 8 characters"
    
    if not any(c.isupper() for c in password):
        return False, "Password must contain uppercase letter"
    
    if not any(c.islower() for c in password):
        return False, "Password must contain lowercase letter"
    
    if not any(c.isdigit() for c in password):
        return False, "Password must contain digit"
    
    return True, None

@auth_bp.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    
    # Validate email
    if not validate_email(data['email']):
        raise ValidationError('Invalid email format')
    
    # Validate password
    is_valid, error = validate_password(data['password'])
    if not is_valid:
        raise ValidationError(error)
    
    # Continue with registration...
```

### 5. SQL Injection Prevention

```python
# ❌ NEVER do this
query = f"SELECT * FROM users WHERE username = '{username}'"

# ✅ Use parameterized queries
from sqlalchemy import text

query = text("SELECT * FROM users WHERE username = :username")
result = db.session.execute(query, {'username': username})

# ✅ Or use ORM
user = User.query.filter_by(username=username).first()
```

### 6. XSS Prevention

```python
from markupsafe import escape

@app.route('/search')
def search():
    query = request.args.get('q', '')
    
    # Escape user input
    safe_query = escape(query)
    
    return f'Search results for: {safe_query}'
```

### 7. HTTPS Only

```python
from flask_talisman import Talisman

# Force HTTPS
Talisman(app, force_https=True)
```

### 8. Security Headers

```python
@app.after_request
def set_security_headers(response):
    """Add security headers to all responses"""
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    return response
```

---

## Performance Optimization

### 1. Response Caching

```python
from flask_caching import Cache

cache = Cache(app, config={
    'CACHE_TYPE': 'simple',
    'CACHE_DEFAULT_TIMEOUT': 300
})

@app.route('/api/data')
@cache.cached(timeout=60)
def get_data():
    # Expensive operation
    data = expensive_database_query()
    return jsonify(data)

# Cache with arguments
@app.route('/api/user/<int:user_id>')
@cache.cached(timeout=60, key_prefix='user_%s')
def get_user(user_id):
    user = User.query.get(user_id)
    return jsonify(user.to_dict())

# Clear cache
@app.route('/api/clear-cache')
@admin_required
def clear_cache():
    cache.clear()
    return jsonify({'message': 'Cache cleared'})
```

### 2. Database Query Optimization

```python
# ❌ N+1 Query Problem
users = User.query.all()
for user in users:
    print(user.posts)  # Separate query for each user!

# ✅ Use eager loading
from sqlalchemy.orm import joinedload

users = User.query.options(joinedload(User.posts)).all()
for user in users:
    print(user.posts)  # No additional queries!

# ✅ Use pagination
@app.route('/api/users')
def get_users():
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 20, type=int)
    
    pagination = User.query.paginate(
        page=page,
        per_page=per_page,
        error_out=False
    )
    
    return jsonify({
        'users': [u.to_dict() for u in pagination.items],
        'total': pagination.total,
        'pages': pagination.pages,
        'current_page': page
    })
```

### 3. Async Processing

```python
from threading import Thread

def async_task(app, func, *args, **kwargs):
    """Run function in background thread"""
    def wrapper():
        with app.app_context():
            func(*args, **kwargs)
    
    thread = Thread(target=wrapper)
    thread.start()
    return thread

@app.route('/api/process', methods=['POST'])
def process_data():
    data = request.get_json()
    
    # Start background processing
    async_task(app, heavy_processing, data)
    
    return jsonify({
        'message': 'Processing started',
        'status': 'pending'
    }), 202
```

### 4. Compression

```python
from flask_compress import Compress

Compress(app)

# All responses are now gzip compressed
```

### 5. Connection Pooling

```python
# config.py
class Config:
    SQLALCHEMY_ENGINE_OPTIONS = {
        'pool_size': 10,
        'pool_recycle': 3600,
        'pool_pre_ping': True
    }
```

---

## Testing Strategies

### Unit Testing

```python
# tests/test_auth.py
import unittest
from app import create_app
from app.models.user import UserModel

class AuthTestCase(unittest.TestCase):
    def setUp(self):
        """Set up test client"""
        self.app = create_app('testing')
        self.client = self.app.test_client()
        self.user_model = UserModel('test_users.json')
    
    def tearDown(self):
        """Clean up after tests"""
        import os
        if os.path.exists('test_users.json'):
            os.remove('test_users.json')
    
    def test_register(self):
        """Test user registration"""
        response = self.client.post('/api/v1/auth/register', json={
            'username': 'testuser',
            'email': 'test@example.com',
            'password': 'password123'
        })
        
        self.assertEqual(response.status_code, 201)
        data = response.get_json()
        self.assertTrue(data['success'])
        self.assertEqual(data['user']['username'], 'testuser')
    
    def test_login(self):
        """Test user login"""
        # Create user first
        self.user_model.create_user('testuser', 'test@example.com', 'password123')
        
        # Login
        response = self.client.post('/api/v1/auth/login', json={
            'username': 'testuser',
            'password': 'password123'
        })
        
        self.assertEqual(response.status_code, 200)
        data = response.get_json()
        self.assertTrue(data['success'])
        self.assertIn('token', data)
    
    def test_login_invalid_credentials(self):
        """Test login with invalid credentials"""
        response = self.client.post('/api/v1/auth/login', json={
            'username': 'nonexistent',
            'password': 'wrongpassword'
        })
        
        self.assertEqual(response.status_code, 401)
        data = response.get_json()
        self.assertIn('error', data)

if __name__ == '__main__':
    unittest.main()
```

### Integration Testing

```python
# tests/test_integration.py
class IntegrationTestCase(unittest.TestCase):
    def test_full_workflow(self):
        """Test complete user workflow"""
        # Register
        response = self.client.post('/api/v1/auth/register', json={
            'username': 'testuser',
            'email': 'test@example.com',
            'password': 'password123'
        })
        self.assertEqual(response.status_code, 201)
        
        # Login
        response = self.client.post('/api/v1/auth/login', json={
            'username': 'testuser',
            'password': 'password123'
        })
        token = response.get_json()['token']
        
        # Access protected route
        response = self.client.get('/api/v1/auth/me', headers={
            'Authorization': f'Bearer {token}'
        })
        self.assertEqual(response.status_code, 200)
        
        # Upload file
        data = {'file': (io.BytesIO(b'test data'), 'test.csv')}
        response = self.client.post('/api/v1/edforest/upload',
            data=data,
            content_type='multipart/form-data',
            headers={'Authorization': f'Bearer {token}'}
        )
        self.assertEqual(response.status_code, 200)
```

### Running Tests

```bash
# Run all tests
python -m unittest discover tests

# Run specific test file
python -m unittest tests.test_auth

# Run with coverage
pip install coverage
coverage run -m unittest discover tests
coverage report
coverage html
```


---

## Deployment

### Production Configuration

```python
# config.py
class ProductionConfig(Config):
    DEBUG = False
    TESTING = False
    
    # Use environment variables
    SECRET_KEY = os.environ.get('SECRET_KEY')
    JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY')
    DATABASE_URL = os.environ.get('DATABASE_URL')
    
    # Security
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'
    
    # Logging
    LOG_LEVEL = 'INFO'
```

### Using Gunicorn (Production Server)

```bash
# Install
pip install gunicorn

# Run
gunicorn -w 4 -b 0.0.0.0:5000 run:app

# With configuration file
gunicorn -c gunicorn_config.py run:app
```

**gunicorn_config.py:**
```python
# Gunicorn configuration
bind = '0.0.0.0:5000'
workers = 4
worker_class = 'sync'
worker_connections = 1000
timeout = 30
keepalive = 2

# Logging
accesslog = 'logs/access.log'
errorlog = 'logs/error.log'
loglevel = 'info'

# Process naming
proc_name = 'vsmc-litho-api'

# Server mechanics
daemon = False
pidfile = 'gunicorn.pid'
```

### Windows Service Deployment

```python
# install_service.py
import win32serviceutil
import win32service
import win32event
import servicemanager
import socket
import sys
import subprocess

class VSMCLithoService(win32serviceutil.ServiceFramework):
    _svc_name_ = "VSMCLithoAPI"
    _svc_display_name_ = "VSMC Litho Platform API"
    _svc_description_ = "Backend API for VSMC Lithography Analysis Platform"
    
    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.stop_event = win32event.CreateEvent(None, 0, 0, None)
        self.process = None
    
    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        win32event.SetEvent(self.stop_event)
        if self.process:
            self.process.terminate()
    
    def SvcDoRun(self):
        servicemanager.LogMsg(
            servicemanager.EVENTLOG_INFORMATION_TYPE,
            servicemanager.PYS_SERVICE_STARTED,
            (self._svc_name_, '')
        )
        self.main()
    
    def main(self):
        # Start Flask application
        self.process = subprocess.Popen([
            'python',
            'C:\\path\\to\\backend\\run.py'
        ])
        
        # Wait for stop event
        win32event.WaitForSingleObject(self.stop_event, win32event.INFINITE)

if __name__ == '__main__':
    if len(sys.argv) == 1:
        servicemanager.Initialize()
        servicemanager.PrepareToHostSingle(VSMCLithoService)
        servicemanager.StartServiceCtrlDispatcher()
    else:
        win32serviceutil.HandleCommandLine(VSMCLithoService)
```

**Install Service:**
```cmd
python install_service.py install
python install_service.py start
```

### Docker Deployment

**Dockerfile:**
```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create folders
RUN mkdir -p uploads outputs history

# Expose port
EXPOSE 5000

# Run with gunicorn
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "run:app"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=production
      - SECRET_KEY=${SECRET_KEY}
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
    volumes:
      - ./backend/uploads:/app/uploads
      - ./backend/outputs:/app/outputs
      - ./backend/history:/app/history
    restart: unless-stopped
  
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped
```

**Build and Run:**
```bash
docker-compose up -d
```

### Nginx Reverse Proxy

**nginx.conf:**
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    
    # Frontend
    location / {
        root /var/www/frontend/dist;
        try_files $uri $uri/ /index.html;
    }
    
    # Backend API
    location /api {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # File uploads
    client_max_body_size 16M;
}
```

---

## Monitoring and Logging

### Logging Configuration

```python
# app/__init__.py
import logging
from logging.handlers import RotatingFileHandler
import os

def create_app(config_name='development'):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    
    # Configure logging
    if not app.debug:
        # Create logs directory
        if not os.path.exists('logs'):
            os.mkdir('logs')
        
        # File handler
        file_handler = RotatingFileHandler(
            'logs/vsmc_api.log',
            maxBytes=10240000,  # 10MB
            backupCount=10
        )
        file_handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s '
            '[in %(pathname)s:%(lineno)d]'
        ))
        file_handler.setLevel(logging.INFO)
        app.logger.addHandler(file_handler)
        
        app.logger.setLevel(logging.INFO)
        app.logger.info('VSMC Litho Platform startup')
    
    return app
```

### Request Logging

```python
@app.before_request
def log_request():
    """Log incoming requests"""
    app.logger.info(f'{request.method} {request.path} - {request.remote_addr}')

@app.after_request
def log_response(response):
    """Log outgoing responses"""
    app.logger.info(f'{request.method} {request.path} - {response.status_code}')
    return response
```

### Error Logging

```python
@app.errorhandler(Exception)
def handle_exception(e):
    """Log and handle exceptions"""
    app.logger.error(f'Unhandled exception: {str(e)}', exc_info=True)
    
    return jsonify({
        'error': 'An unexpected error occurred',
        'status': 500
    }), 500
```

### Performance Monitoring

```python
import time
from functools import wraps

def monitor_performance(f):
    """Decorator to monitor endpoint performance"""
    @wraps(f)
    def decorated(*args, **kwargs):
        start_time = time.time()
        
        result = f(*args, **kwargs)
        
        duration = time.time() - start_time
        app.logger.info(f'{f.__name__} took {duration:.3f}s')
        
        # Alert if slow
        if duration > 1.0:
            app.logger.warning(f'Slow endpoint: {f.__name__} ({duration:.3f}s)')
        
        return result
    return decorated


@app.route('/api/analyze', methods=['POST'])
@monitor_performance
def analyze():
    # Analysis logic
    pass
```

### Health Check Endpoint

```python
# app/routes/health.py
from flask import Blueprint, jsonify
import psutil
import os

health_bp = Blueprint('health', __name__)

@health_bp.route('/', methods=['GET'])
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'version': '1.0.0'
    })

@health_bp.route('/detailed', methods=['GET'])
def detailed_health():
    """Detailed health check with system metrics"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'version': '1.0.0',
        'system': {
            'cpu_percent': psutil.cpu_percent(),
            'memory_percent': psutil.virtual_memory().percent,
            'disk_percent': psutil.disk_usage('/').percent
        },
        'application': {
            'uptime': get_uptime(),
            'requests_processed': get_request_count()
        }
    })
```

### Application Metrics

```python
from prometheus_flask_exporter import PrometheusMetrics

# Initialize metrics
metrics = PrometheusMetrics(app)

# Custom metrics
request_count = metrics.counter(
    'requests_total',
    'Total requests',
    labels={'method': lambda: request.method, 'endpoint': lambda: request.endpoint}
)

request_duration = metrics.histogram(
    'request_duration_seconds',
    'Request duration',
    labels={'method': lambda: request.method, 'endpoint': lambda: request.endpoint}
)

# Metrics endpoint: /metrics
```

---

## Best Practices Summary

### 1. Project Structure

```
backend/
├── app/
│   ├── __init__.py          # Application factory
│   ├── models/              # Data models
│   ├── routes/              # API endpoints (blueprints)
│   ├── services/            # Business logic
│   ├── utils/               # Helper functions
│   └── middleware/          # Custom middleware
├── tests/                   # Test files
├── logs/                    # Log files
├── config.py                # Configuration
├── run.py                   # Entry point
├── requirements.txt         # Dependencies
└── .env                     # Environment variables
```

### 2. Code Organization

- **Routes:** Handle HTTP only
- **Services:** Business logic
- **Models:** Data access
- **Utils:** Reusable helpers
- **Middleware:** Cross-cutting concerns

### 3. Security Checklist

- ✅ Use environment variables for secrets
- ✅ Hash passwords with bcrypt
- ✅ Implement JWT authentication
- ✅ Validate all inputs
- ✅ Use HTTPS in production
- ✅ Set security headers
- ✅ Implement rate limiting
- ✅ Use CORS properly
- ✅ Sanitize file uploads
- ✅ Log security events

### 4. Performance Checklist

- ✅ Use caching where appropriate
- ✅ Optimize database queries
- ✅ Implement pagination
- ✅ Use connection pooling
- ✅ Enable compression
- ✅ Monitor slow endpoints
- ✅ Use async for long tasks
- ✅ Set appropriate timeouts

### 5. Testing Checklist

- ✅ Write unit tests for services
- ✅ Write integration tests for APIs
- ✅ Test error handling
- ✅ Test authentication/authorization
- ✅ Test file uploads
- ✅ Measure code coverage
- ✅ Use CI/CD pipeline

### 6. Deployment Checklist

- ✅ Use production WSGI server (Gunicorn)
- ✅ Set DEBUG=False
- ✅ Configure logging
- ✅ Set up monitoring
- ✅ Use reverse proxy (Nginx)
- ✅ Enable HTTPS
- ✅ Set up backups
- ✅ Document deployment process

---

## Real-World Example: Complete API Endpoint

Here's a complete, production-ready endpoint from our project:

```python
# app/routes/history.py
from flask import Blueprint, request, jsonify, send_file
from werkzeug.utils import secure_filename
from app.utils.auth import token_required
from app.utils.exceptions import ValidationError, NotFoundError
import os
import json
from datetime import datetime

history_bp = Blueprint('history', __name__)

# Configuration
HISTORY_DIR = os.path.join(os.path.dirname(__file__), '../../history')
INDEX_FILE = os.path.join(HISTORY_DIR, 'index.json')

# Ensure directory exists
os.makedirs(HISTORY_DIR, exist_ok=True)


def load_index():
    """Load history index"""
    if os.path.exists(INDEX_FILE):
        try:
            with open(INDEX_FILE, 'r', encoding='utf-8') as f:
                return json.load(f)
        except Exception:
            return []
    return []


def save_index(index_data):
    """Save history index"""
    with open(INDEX_FILE, 'w', encoding='utf-8') as f:
        json.dump(index_data, f, indent=2, ensure_ascii=False)


@history_bp.route('/save', methods=['POST'])
@token_required
def save_analysis():
    """
    Save analysis results to history
    
    Expected JSON:
    {
        "username": "Daniel",
        "analysis_results": {...},
        "chart_data": {...},
        "parameters": {...}
    }
    
    Returns:
    --------
    JSON response with saved analysis ID
    """
    try:
        data = request.json
        
        # Validation
        if not data:
            raise ValidationError('No data provided')
        
        if not data.get('username'):
            raise ValidationError('Username is required')
        
        # Get username from token (more secure)
        username = request.user['username']
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        
        # Create filename
        filename = f"{secure_filename(username)}_{timestamp}.json"
        filepath = os.path.join(HISTORY_DIR, filename)
        
        # Prepare data
        save_data = {
            'metadata': {
                'username': username,
                'timestamp': timestamp,
                'datetime': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                'filename': filename
            },
            'parameters': data.get('parameters', {}),
            'analysis_results': data.get('analysis_results', {}),
            'chart_data': data.get('chart_data', {})
        }
        
        # Save analysis data
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(save_data, f, indent=2, ensure_ascii=False)
        
        # Update index
        index = load_index()
        index_entry = {
            'id': timestamp,
            'username': username,
            'timestamp': timestamp,
            'datetime': save_data['metadata']['datetime'],
            'filename': filename,
            'parameters': data.get('parameters', {}),
            'summary': {
                'dof': data.get('analysis_results', {}).get('process_window', {}).get('dof'),
                'el': data.get('analysis_results', {}).get('process_window', {}).get('el')
            }
        }
        index.append(index_entry)
        save_index(index)
        
        # Log success
        current_app.logger.info(f'Analysis saved: {filename} by {username}')
        
        return jsonify({
            'success': True,
            'message': 'Analysis saved successfully',
            'filename': filename,
            'id': timestamp
        }), 201
        
    except ValidationError as e:
        raise
    except Exception as e:
        current_app.logger.error(f'Save analysis failed: {str(e)}')
        raise Exception(f'Failed to save analysis: {str(e)}')


@history_bp.route('/list', methods=['GET'])
@token_required
def list_history():
    """
    Get list of saved analyses
    
    Query params:
    - username: Filter by username (optional)
    - limit: Number of results (optional)
    
    Returns:
    --------
    JSON response with analysis list
    """
    try:
        index = load_index()
        
        # Filter by username if provided
        username = request.args.get('username')
        if username:
            index = [item for item in index if item['username'].lower() == username.lower()]
        
        # Sort by timestamp (newest first)
        index.sort(key=lambda x: x['timestamp'], reverse=True)
        
        # Limit results
        limit = request.args.get('limit', type=int)
        if limit:
            index = index[:limit]
        
        return jsonify({
            'success': True,
            'count': len(index),
            'history': index
        }), 200
        
    except Exception as e:
        current_app.logger.error(f'List history failed: {str(e)}')
        raise Exception(f'Failed to list history: {str(e)}')


@history_bp.route('/load/<analysis_id>', methods=['GET'])
@token_required
def load_analysis(analysis_id):
    """Load specific analysis by ID"""
    try:
        # Find in index
        index = load_index()
        entry = next((item for item in index if item['id'] == analysis_id), None)
        
        if not entry:
            raise NotFoundError('Analysis not found')
        
        # Load full data
        filepath = os.path.join(HISTORY_DIR, entry['filename'])
        
        if not os.path.exists(filepath):
            raise NotFoundError('Analysis file not found')
        
        with open(filepath, 'r', encoding='utf-8') as f:
            data = json.load(f)
        
        return jsonify({
            'success': True,
            'data': data
        }), 200
        
    except (ValidationError, NotFoundError) as e:
        raise
    except Exception as e:
        current_app.logger.error(f'Load analysis failed: {str(e)}')
        raise Exception(f'Failed to load analysis: {str(e)}')
```

**This endpoint demonstrates:**

1. ✅ Authentication with JWT
2. ✅ Input validation
3. ✅ Custom exceptions
4. ✅ Secure file handling
5. ✅ Proper error handling
6. ✅ Logging
7. ✅ Clear documentation
8. ✅ Consistent response format
9. ✅ Appropriate status codes
10. ✅ Security best practices

---

## Conclusion

Advanced Flask development requires mastering:

1. **Authentication & Authorization:** JWT, decorators, role-based access
2. **Architecture:** Service layer, separation of concerns
3. **Security:** Input validation, password hashing, HTTPS
4. **Performance:** Caching, query optimization, async processing
5. **Testing:** Unit tests, integration tests, coverage
6. **Deployment:** Production servers, Docker, monitoring
7. **Logging:** Request logging, error tracking, metrics

The VSMC Litho Platform demonstrates these concepts in a real production application. Study the codebase, practice the exercises, and apply these patterns to your own projects.

**Happy coding! 🚀**

---

*Training Material Version: 1.0*  
*Last Updated: January 2026*  
*For: VSMC DAP Team*

