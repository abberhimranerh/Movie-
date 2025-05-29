
## 1. Backend Implementation (Express.js + MongoDB)

### File Structure
```
backend/
  ├── config/
  │   └── db.js
  ├── controllers/
  │   ├── authController.js
  │   ├── movieController.js
  │   └── userController.js
  ├── middleware/
  │   └── auth.js
  ├── models/
  │   ├── User.js
  │   └── Movie.js
  ├── routes/
  │   ├── authRoutes.js
  │   ├── movieRoutes.js
  │   └── userRoutes.js
  ├── .env
  ├── server.js
  └── package.json
```

### server.js
```javascript
require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const authRoutes = require('./routes/authRoutes');
const movieRoutes = require('./routes/movieRoutes');
const userRoutes = require('./routes/userRoutes');

const app = express();
const PORT = process.env.PORT || 5000;

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/movies', movieRoutes);
app.use('/api/users', userRoutes);

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### User Model (models/User.js)
```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  watchlists: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Watchlist' }],
  favorites: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Movie' }],
  ratings: [{
    movieId: { type: mongoose.Schema.Types.ObjectId, ref: 'Movie' },
    score: { type: Number, min: 1, max: 5 }
  }]
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Generate JWT token
userSchema.methods.generateAuthToken = function() {
  return jwt.sign(
    { id: this._id, username: this.username },
    process.env.JWT_SECRET,
    { expiresIn: '30d' }
  );
};

module.exports = mongoose.model('User', userSchema);
```

### Auth Controller (controllers/authController.js)
```javascript
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Register new user
exports.register = async (req, res) => {
  try {
    const { username, email, password } = req.body;
    
    // Check if user exists
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }

    // Create user
    const user = await User.create({ username, email, password });
    
    // Generate token
    const token = user.generateAuthToken();
    
    res.status(201).json({ token, user: { id: user._id, username: user.username, email: user.email } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Login user
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Check if user exists
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    // Check password
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    // Generate token
    const token = user.generateAuthToken();
    
    res.json({ token, user: { id: user._id, username: user.username, email: user.email } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## 2. Frontend Implementation (React)

### File Structure
```
frontend/
  ├── public/
  ├── src/
  │   ├── api/
  │   │   ├── auth.js
  │   │   ├── movies.js
  │   │   └── users.js
  │   ├── components/
  │   │   ├── auth/
  │   │   ├── movies/
  │   │   └── shared/
  │   ├── context/
  │   │   └── AuthContext.js
  │   ├── pages/
  │   │   ├── Auth/
  │   │   ├── Dashboard/
  │   │   ├── MovieDetail/
  │   │   └── Search/
  │   ├── App.js
  │   ├── index.js
  │   └── styles/
  ├── package.json
  └── README.md
```

### Auth Context (context/AuthContext.js)
```javascript
import React, { createContext, useState, useEffect } from 'react';
import { loginUser, registerUser, getCurrentUser } from '../api/auth';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const checkAuth = async () => {
      try {
        if (token) {
          const userData = await getCurrentUser(token);
          setUser(userData);
          setIsAuthenticated(true);
        }
      } catch (error) {
        console.error('Auth check failed:', error);
        logout();
      } finally {
        setLoading(false);
      }
    };
    
    checkAuth();
  }, [token]);

  const login = async (credentials) => {
    const { token, user } = await loginUser(credentials);
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    setIsAuthenticated(true);
  };

  const register = async (userData) => {
    const { token, user } = await registerUser(userData);
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    setIsAuthenticated(true);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    setIsAuthenticated(false);
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        token,
        isAuthenticated,
        loading,
        login,
        register,
        logout
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};
```

### Movie API (api/movies.js)
```javascript
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

// TMDB API integration
const TMDB_API_KEY = process.env.REACT_APP_TMDB_API_KEY;

export const searchMovies = async (query) => {
  try {
    const response = await axios.get(
      `https://api.themoviedb.org/3/search/movie?api_key=${TMDB_API_KEY}&query=${query}`
    );
    return response.data.results;
  } catch (error) {
    console.error('Error searching movies:', error);
    throw error;
  }
};

export const getMovieDetails = async (movieId) => {
  try {
    const response = await axios.get(
      `https://api.themoviedb.org/3/movie/${movieId}?api_key=${TMDB_API_KEY}`
    );
    return response.data;
  } catch (error) {
    console.error('Error fetching movie details:', error);
    throw error;
  }
};

export const getRecommendations = async (movieId) => {
  try {
    const response = await axios.get(
      `https://api.themoviedb.org/3/movie/${movieId}/recommendations?api_key=${TMDB_API_KEY}`
    );
    return response.data.results;
  } catch (error) {
    console.error('Error fetching recommendations:', error);
    throw error;
  }
};
```

## 3. Deployment

### Frontend (Vercel)
1. Install Vercel CLI: `npm install -g vercel`
2. Login: `vercel login`
3. Deploy: `vercel`

### Backend (Render)
1. Create a new Web Service on Render
2. Connect your GitHub repository
3. Set environment variables:
   - `MONGODB_URI`
   - `JWT_SECRET`
   - `PORT`
4. Deploy

## 4. Environment Variables

### Backend (.env)
```env
MONGODB_URI=mongodb+srv://<username>:<password>@cluster0.mongodb.net/movie-app?retryWrites=true&w=majority
JWT_SECRET=your_jwt_secret
PORT=5000
TMDB_API_KEY=your_tmdb_api_key
```

### Frontend (.env)
```env
REACT_APP_API_URL=https://your-render-backend-url.onrender.com
REACT_APP_TMDB_API_KEY=your_tmdb_api_key
```

## 5. Complete Features

1. **User Authentication**
   - Secure JWT-based registration/login
   - Protected routes
   - Password hashing

2. **Movie Discovery**
   - TMDB API integration
   - Search functionality
   - Detailed movie pages
   - Recommendations

3. **User Features**
   - Favorite movies
   - Watchlists
   - Ratings and reviews
   - Profile management

4. **Deployment**
   - Vercel for frontend
   - Render for backend
   - MongoDB Atlas for database

## 6. Stretch Goals Implementation

```javascript
// Social features
export const followUser = async (userId, token) => {
  const response = await axios.post(
    `${API_URL}/users/follow/${userId}`,
    {},
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Trailer integration
export const getMovieTrailer = async (movieId) => {
  const response = await axios.get(
    `https://api.themoviedb.org/3/movie/${movieId}/videos?api_key=${TMDB_API_KEY}`
  );
  return response.data.results.find(video => video.type === 'Trailer');
};

// PWA setup (in public/manifest.json)
{
  "short_name": "MovieApp",
  "name": "Movie Recommendation App",
  "icons": [
    {
      "src": "/icons/icon-192x192.png",
      "type": "image/png",
      "sizes": "192x192"
    },
    {
      "src": "/icons/icon-512x512.png",
      "type": "image/png",
      "sizes": "512x512"
    }
  ],
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#000000",
  "background_color": "#ffffff"
}
```
