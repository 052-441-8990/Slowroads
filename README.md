# SLOW ROADS SOURCE CODE VER 1.01
### By Things
<br>
** THIS IS NOT THE OFFICIAL SOURCE AND IS OUTDATED! **
<hr>
This is the source code to Slow Roads. I looked through the code and developer console and downloaded the code and all dependancies.
<br>
Visit the full preview at thingsinreverse.github.io/slowroads
<br>
All credits go to Anslo, visit the original at slowroads.io
<br>
Clarified from Anslo, this is under a http://creativecommons.org/licenses/by-nc-nd/4.0/ license, which means you can copy the original but not change the program; but it is stated this will be changed soon. 
<br>
VER: 1.01
// Slow Roads Premium API - Node.js/Express Implementation
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');

const app = express();
const PORT = process.env.PORT || 3000;
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

// Middleware
app.use(helmet());
app.use(express.json());

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});
app.use('/api/', limiter);

// Mock database (in production, use proper database)
const users = [];
const premiumFeatures = [];
const gameStats = [];

// Authentication middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ error: 'Invalid token' });
    req.user = user;
    next();
  });
};

;
// Slow Roads Premium API - Node.js/Express Implementation
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3000;
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});
app.use('/api/', limiter);

// Mock database (in production, use proper database)
const users = new Map();
const savedRoutes = new Map();
const gameStats = new Map();

// Authentication middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ error: 'Invalid token' });
    req.user = user;
    next();
  });
};

// Premium subscription middleware
const requirePremium = (req, res, next) => {
  if (!req.user.isPremium) {
    return res.status(403).json({ 
      error: 'Premium subscription required',
      upgradeUrl: '/api/premium/upgrade'
    });
  }
  next();
};

// === AUTH ENDPOINTS ===

// User registration
app.post('/api/auth/register', async (req, res) => {
  try {
    const { username, email, password } = req.body;
    
    // Input validation
    if (!username || !email || !password) {
      return res.status(400).json({ error: 'Username, email, and password are required' });
    }
    
    if (password.length < 6) {
      return res.status(400).json({ error: 'Password must be at least 6 characters long' });
    }
    
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      return res.status(400).json({ error: 'Invalid email format' });
    }
    
    // Check if user exists
    const existingUser = Array.from(users.values()).find(u => u.email === email);
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    // Hash password
    const saltRounds = 10;
    const hashedPassword = await bcrypt.hash(password, saltRounds);

    // Create user
    const userId = Date.now().toString() + Math.random().toString(36).substr(2, 9);
    const user = {
      id: userId,
      username: username.trim(),
      email: email.toLowerCase().trim(),
      password: hashedPassword,
      isPremium: false,
      premiumPlan: null,
      premiumStartDate: null,
      createdAt: new Date().toISOString(),
      lastLogin: new Date().toISOString(),
      gameStats: {
        totalDistance: 0,
        totalPlayTime: 0,
        highestSpeed: 0,
        roadsGenerated: 0
      }
    };

    users.set(userId, user);

    // Generate JWT
    const token = jwt.sign(
      { userId: user.id, email: user.email, isPremium: user.isPremium },
      JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.status(201).json({
      message: 'User created successfully',
      token,
      user: {
        id: user.id,
        username: user.username,
        email: user.email,
        isPremium: user.isPremium,
        createdAt: user.createdAt
      }
    });

  } catch (error) {
    console.error('Registration error:', error);
    res.status(500).json({ error: 'Server error during registration' });
  }
});

// User login
app.post('/api/auth/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Input validation
    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password are required' });
    }

    const user = Array.from(users.values()).find(u => u.email === email.toLowerCase().trim());
    if (!user) {
      return res.status(400).json({ error: 'Invalid credentials' });
    }

    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(400).json({ error: 'Invalid credentials' });
    }

    // Update last login
    user.lastLogin = new Date().toISOString();

    const token = jwt.sign(
      { userId: user.id, email: user.email, isPremium: user.isPremium },
      JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({
      message: 'Login successful',
      token,
      user: {
        id: user.id,
        username: user.username,
        email: user.email,
        isPremium: user.isPremium,
        lastLogin: user.lastLogin
      }
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Server error during login' });
  }
});

// === PREMIUM ENDPOINTS ===

// Get premium features
app.get('/api/premium/features', authenticateToken, (req, res) => {
  const features = {
    free: [
      'Basic road generation',
      'Standard weather conditions',
      'Basic car models (3 available)',
      'Day/night cycle'
    ],
    premium: [
      'Advanced road generation with hills and curves',
      'Weather effects (rain, snow, fog)',
      'Premium car models (15+ available)',
      'Custom car colors and modifications',
      'Photo mode with filters',
      'Time-lapse recording',
      'Seasonal environments',
      'Radio stations',
      'Save/load custom routes',
      'Cloud sync for game progress',
      'No ads'
    ]
  };

  res.json({
    features,
    userStatus: req.user.isPremium ? 'premium' : 'free',
    pricing: {
      monthly: '$4.99',
      yearly: '$39.99'
    }
  });
});

// Upgrade to premium
app.post('/api/premium/upgrade', authenticateToken, (req, res) => {
  try {
    const { plan, paymentMethod } = req.body;
    
    // Validate plan
    if (!plan || !['monthly', 'yearly'].includes(plan)) {
      return res.status(400).json({ error: 'Invalid plan. Choose monthly or yearly' });
    }
    
    // Validate payment method
    if (!paymentMethod) {
      return res.status(400).json({ error: 'Payment method is required' });
    }
    
    const user = users.get(req.user.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    if (user.isPremium) {
      return res.status(400).json({ error: 'User already has premium subscription' });
    }
    
    // In real implementation, integrate with payment processor (Stripe, PayPal, etc.)
    // For demo purposes, we'll simulate successful payment
    
    user.isPremium = true;
    user.premiumPlan = plan;
    user.premiumStartDate = new Date().toISOString();
    
    // Update user in map
    users.set(req.user.userId, user);

    res.json({
      message: 'Premium upgrade successful',
      plan,
      startDate: user.premiumStartDate,
      features: [
        'Advanced road generation',
        'Weather effects',
        'Premium cars',
        'Photo mode',
        'Custom routes',
        'Cloud sync',
        'No ads'
      ]
    });
    
  } catch (error) {
    console.error('Premium upgrade error:', error);
    res.status(500).json({ error: 'Server error during premium upgrade' });
  }
});

// === GAME ENDPOINTS ===

// Get user game stats
app.get('/api/game/stats', authenticateToken, (req, res) => {
  try {
    const user = users.get(req.user.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    const achievements = [
      { 
        id: 1,
        name: 'First Drive', 
        description: 'Complete your first drive', 
        unlocked: user.gameStats.totalDistance > 0,
        icon: '🚗'
      },
      { 
        id: 2,
        name: 'Speed Demon', 
        description: 'Reach 100 mph', 
        unlocked: user.gameStats.highestSpeed >= 100,
        icon: '⚡'
      },
      { 
        id: 3,
        name: 'Explorer', 
        description: 'Generate 10 different roads', 
        unlocked: user.gameStats.roadsGenerated >= 10,
        icon: '🗺️'
      },
      { 
        id: 4,
        name: 'Long Hauler', 
        description: 'Drive 1000 miles total', 
        unlocked: user.gameStats.totalDistance >= 1000,
        icon: '🛣️'
      },
      { 
        id: 5,
        name: 'Time Traveler', 
        description: 'Play for 10 hours total', 
        unlocked: user.gameStats.totalPlayTime >= 600, // 10 hours in minutes
        icon: '⏰'
      }
    ];
    
    res.json({
      stats: user.gameStats,
      achievements,
      level: Math.floor(user.gameStats.totalDistance / 100) + 1,
      nextLevelDistance: ((Math.floor(user.gameStats.totalDistance / 100) + 1) * 100) - user.gameStats.totalDistance
    });
    
  } catch (error) {
    console.error('Get stats error:', error);
    res.status(500).json({ error: 'Server error retrieving stats' });
  }
});

// Update game stats
app.post('/api/game/stats', authenticateToken, (req, res) => {
  try {
    const { distance, playTime, maxSpeed, roadsGenerated } = req.body;
    
    // Validate input
    if (distance < 0 || playTime < 0 || maxSpeed < 0 || roadsGenerated < 0) {
      return res.status(400).json({ error: 'Stats values cannot be negative' });
    }
    
    const user = users.get(req.user.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    // Update stats
    user.gameStats.totalDistance += distance || 0;
    user.gameStats.totalPlayTime += playTime || 0;
    user.gameStats.highestSpeed = Math.max(user.gameStats.highestSpeed, maxSpeed || 0);
    user.gameStats.roadsGenerated += roadsGenerated || 0;
    
    // Save updated user
    users.set(req.user.userId, user);

    res.json({
      message: 'Stats updated successfully',
      stats: user.gameStats,
      newLevel: Math.floor(user.gameStats.totalDistance / 100) + 1
    });
    
  } catch (error) {
    console.error('Update stats error:', error);
    res.status(500).json({ error: 'Server error updating stats' });
  }
});

// Get available cars
app.get('/api/game/cars', authenticateToken, (req, res) => {
  const basicCars = [
    { id: 1, name: 'Classic Sedan', type: 'sedan', available: true },
    { id: 2, name: 'Compact Car', type: 'compact', available: true },
    { id: 3, name: 'SUV', type: 'suv', available: true }
  ];

  const premiumCars = [
    { id: 4, name: 'Sports Car', type: 'sports', available: req.user.isPremium },
    { id: 5, name: 'Luxury Sedan', type: 'luxury', available: req.user.isPremium },
    { id: 6, name: 'Electric Car', type: 'electric', available: req.user.isPremium },
    { id: 7, name: 'Vintage Car', type: 'vintage', available: req.user.isPremium },
    { id: 8, name: 'Truck', type: 'truck', available: req.user.isPremium }
  ];

  res.json({
    cars: [...basicCars, ...premiumCars],
    customization: {
      colors: req.user.isPremium ? 
        ['red', 'blue', 'green', 'yellow', 'black', 'white', 'silver', 'gold'] :
        ['red', 'blue', 'white'],
      modifications: req.user.isPremium
    }
  });
});

// Generate road (premium features)
app.post('/api/game/generate-road', authenticateToken, (req, res) => {
  const { 
    length = 'medium', 
    terrain = 'flat', 
    weather = 'clear', 
    season = 'summer',
    timeOfDay = 'day' 
  } = req.body;

  // Premium terrain types
  const premiumTerrain = ['hills', 'mountains', 'desert', 'forest'];
  if (premiumTerrain.includes(terrain) && !req.user.isPremium) {
    return res.status(403).json({ 
      error: 'Premium subscription required for advanced terrain',
      available: ['flat', 'slight_curves']
    });
  }

  // Premium weather effects
  const premiumWeather = ['rain', 'snow', 'fog', 'storm'];
  if (premiumWeather.includes(weather) && !req.user.isPremium) {
    return res.status(403).json({ 
      error: 'Premium subscription required for weather effects',
      available: ['clear', 'cloudy']
    });
  }

  // Generate road data
  const roadData = {
    id: Math.random().toString(36).substr(2, 9),
    length,
    terrain,
    weather,
    season,
    timeOfDay,
    seed: Math.random(),
    waypoints: generateWaypoints(length, terrain),
    environment: {
      temperature: getTemperature(season, weather),
      visibility: getVisibility(weather),
      windSpeed: getWindSpeed(weather)
    }
  };

  res.json({
    message: 'Road generated successfully',
    road: roadData
  });
});

// Save custom route (premium feature)
app.post('/api/game/save-route', authenticateToken, requirePremium, (req, res) => {
  try {
    const { name, roadData, screenshot } = req.body;
    
    // Validate input
    if (!name || !roadData) {
      return res.status(400).json({ error: 'Route name and road data are required' });
    }
    
    if (name.length > 50) {
      return res.status(400).json({ error: 'Route name must be 50 characters or less' });
    }
    
    const routeId = Date.now().toString() + Math.random().toString(36).substr(2, 9);
    
    const route = {
      id: routeId,
      userId: req.user.userId,
      name: name.trim(),
      roadData,
      screenshot: screenshot || null,
      createdAt: new Date().toISOString(),
      plays: 0,
      isPublic: false
    };

    // Get user's existing routes
    const userRoutes = savedRoutes.get(req.user.userId) || [];
    
    // Check route limit (max 20 routes per user)
    if (userRoutes.length >= 20) {
      return res.status(400).json({ error: 'Maximum 20 saved routes allowed' });
    }
    
    userRoutes.push(route);
    savedRoutes.set(req.user.userId, userRoutes);

    res.json({
      message: 'Route saved successfully',
      routeId: route.id,
      route: {
        id: route.id,
        name: route.name,
        createdAt: route.createdAt,
        plays: route.plays
      }
    });
    
  } catch (error) {
    console.error('Save route error:', error);
    res.status(500).json({ error: 'Server error saving route' });
  }
});

// Get user's saved routes
app.get('/api/game/saved-routes', authenticateToken, requirePremium, (req, res) => {
  try {
    const userRoutes = savedRoutes.get(req.user.userId) || [];
    
    const routes = userRoutes.map(route => ({
      id: route.id,
      name: route.name,
      preview: route.screenshot ? route.screenshot.substring(0, 100) + '...' : null,
      createdAt: route.createdAt,
      plays: route.plays,
      isPublic: route.isPublic
    }));

    res.json({ 
      routes,
      totalRoutes: routes.length,
      maxRoutes: 20
    });
    
  } catch (error) {
    console.error('Get saved routes error:', error);
    res.status(500).json({ error: 'Server error retrieving saved routes' });
  }
});

// Helper functions
function generateWaypoints(length, terrain) {
  const numPoints = length === 'short' ? 10 : length === 'medium' ? 20 : 40;
  const waypoints = [];
  
  for (let i = 0; i < numPoints; i++) {
    waypoints.push({
      x: i * 100,
      y: terrain === 'hills' ? Math.sin(i * 0.5) * 50 : 0,
      z: i * 100
    });
  }
  
  return waypoints;
}

function getTemperature(season, weather) {
  const seasonTemps = { 
    spring: 65, 
    summer: 80, 
    autumn: 55, 
    winter: 35 
  };
  
  const weatherModifier = {
    clear: 0,
    cloudy: -5,
    rain: -10,
    snow: -20,
    fog: -8,
    storm: -15
  };
  
  const baseTemp = seasonTemps[season] || 65;
  const modifier = weatherModifier[weather] || 0;
  
  return Math.max(baseTemp + modifier, -10); // Minimum -10°F
}

function getVisibility(weather) {
  const visibilityMap = { 
    clear: 10, 
    cloudy: 8, 
    rain: 5, 
    fog: 2, 
    snow: 3, 
    storm: 1 
  };
  return visibilityMap[weather] || 10;
}

function getWindSpeed(weather) {
  const windSpeedMap = { 
    clear: 5, 
    cloudy: 10, 
    rain: 15, 
    fog: 3, 
    snow: 12, 
    storm: 25 
  };
  return windSpeedMap[weather] || 5;
}

// Add health check endpoint
app.get('/api/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: '1.0.0'
  });
});

// Add user profile endpoint
app.get('/api/user/profile', authenticateToken, (req, res) => {
  try {
    const user = users.get(req.user.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json({
      id: user.id,
      username: user.username,
      email: user.email,
      isPremium: user.isPremium,
      premiumPlan: user.premiumPlan,
      createdAt: user.createdAt,
      lastLogin: user.lastLogin,
      gameStats: user.gameStats
    });
    
  } catch (error) {
    console.error('Profile error:', error);
    res.status(500).json({ error: 'Server error retrieving profile' });
  }
});

// === LEADERBOARD ENDPOINTS ===

// Get global leaderboard
app.get('/api/leaderboard', authenticateToken, (req, res) => {
  try {
    const { type = 'distance', limit = 10 } = req.query;
    
    // Validate type
    if (!['distance', 'speed', 'time'].includes(type)) {
      return res.status(400).json({ error: 'Invalid leaderboard type. Use: distance, speed, or time' });
    }
    
    // Validate limit
    const limitNum = parseInt(limit);
    if (isNaN(limitNum) || limitNum < 1 || limitNum > 100) {
      return res.status(400).json({ error: 'Limit must be between 1 and 100' });
    }
    
    // Get all users and sort by specified type
    const allUsers = Array.from(users.values());
    let sortedUsers;
    
    switch (type) {
      case 'distance':
        sortedUsers = allUsers.sort((a, b) => b.gameStats.totalDistance - a.gameStats.totalDistance);
        break;
      case 'speed':
        sortedUsers = allUsers.sort((a, b) => b.gameStats.highestSpeed - a.gameStats.highestSpeed);
        break;
      case 'time':
        sortedUsers = allUsers.sort((a, b) => b.gameStats.totalPlayTime - a.gameStats.totalPlayTime);
        break;
    }
    
    // Create leaderboard
    const leaderboard = sortedUsers.slice(0, limitNum).map((user, index) => ({
      rank: index + 1,
      username: user.username,
      score: type === 'distance' ? user.gameStats.totalDistance : 
             type === 'speed' ? user.gameStats.highestSpeed : 
             user.gameStats.totalPlayTime,
      isPremium: user.isPremium
    }));
    
    // Find current user's rank
    const currentUser = users.get(req.user.userId);
    let userRank = 0;
    let userScore = 0;
    
    if (currentUser) {
      userRank = sortedUsers.findIndex(u => u.id === req.user.userId) + 1;
      userScore = type === 'distance' ? currentUser.gameStats.totalDistance : 
                  type === 'speed' ? currentUser.gameStats.highestSpeed : 
                  currentUser.gameStats.totalPlayTime;
    }

    res.json({
      leaderboard,
      type,
      userRank,
      userScore,
      totalPlayers: allUsers.length
    });
    
  } catch (error) {
    console.error('Leaderboard error:', error);
    res.status(500).json({ error: 'Server error retrieving leaderboard' });
  }
});100) + 1,
    userScore: Math.floor(Math.random() * 10000)
  });
});

// Error handling middleware
app.use((error, req, res, next) => {
  console.error(error);
  res.status(500).json({ error: 'Internal server error' });
});

// Start server
app.listen(PORT, () => {
  console.log(`Slow Roads Premium API running on port ${PORT}`);
});

module.exports = app;
