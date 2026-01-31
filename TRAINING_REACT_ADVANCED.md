# 📚 React Programming Training - ADVANCED

## For Engineers Ready to Level Up

**Target Audience:** Engineers who completed React Basics  
**Prerequisites:** Understanding of useState, useEffect, components, props  
**Learning Time:** 2-3 weeks  
**Difficulty:** Intermediate to Advanced

---

## Table of Contents
1. [Context API - Global State](#context-api---global-state)
2. [Custom Hooks](#custom-hooks)
3. [React Router - Navigation](#react-router---navigation)
4. [Protected Routes](#protected-routes)
5. [API Integration Patterns](#api-integration-patterns)
6. [Advanced useEffect](#advanced-useeffect)
7. [Performance Optimization](#performance-optimization)
8. [Component Composition](#component-composition)
9. [Error Boundaries](#error-boundaries)
10. [Best Practices](#best-practices)

---

## Context API - Global State

### The Problem

Passing props through many levels is tedious:

```jsx
// ❌ Prop Drilling Problem
<App user={user}>
  <Header user={user}>
    <Navigation user={user}>
      <UserMenu user={user} />  {/* Finally used here! */}
    </Navigation>
  </Header>
</App>
```

### The Solution: Context API

Context lets you share data across the entire component tree without passing props manually.

### Real Example: AuthContext

Let's analyze our authentication context:

```jsx
// frontend/src/context/AuthContext.jsx
import { createContext, useContext, useState, useEffect } from 'react'
import { useNavigate } from 'react-router-dom'

// Step 1: Create Context
const AuthContext = createContext()

// Step 2: Create Custom Hook for Easy Access
export const useAuth = () => {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}

// Step 3: Create Provider Component
export const AuthProvider = ({ children }) => {
  // State
  const [user, setUser] = useState(null)
  const [token, setToken] = useState(null)
  const [loading, setLoading] = useState(true)
  const navigate = useNavigate()

  // Load saved session on mount
  useEffect(() => {
    const storedToken = localStorage.getItem('token')
    const storedUser = localStorage.getItem('user')
    
    if (storedToken && storedUser) {
      setToken(storedToken)
      setUser(JSON.parse(storedUser))
    }
    
    setLoading(false)
  }, [])

  // Login function
  const login = async (username, password) => {
    try {
      const response = await fetch('http://localhost:5000/api/v1/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ username, password }),
      })

      const data = await response.json()

      if (!response.ok) {
        throw new Error(data.error || 'Login failed')
      }

      // Save to localStorage
      localStorage.setItem('token', data.token)
      localStorage.setItem('user', JSON.stringify(data.user))
      
      // Update state
      setToken(data.token)
      setUser(data.user)

      return { success: true, user: data.user }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  // Logout function
  const logout = () => {
    localStorage.removeItem('token')
    localStorage.removeItem('user')
    setToken(null)
    setUser(null)
    navigate('/login')
  }

  // Helper functions
  const isAuthenticated = () => !!token && !!user
  const isAdmin = () => user?.role === 'admin'

  // Value object - everything we want to share
  const value = {
    user,
    token,
    loading,
    login,
    register,
    logout,
    isAuthenticated,
    isAdmin,
  }

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  )
}
```

**Understanding Each Part:**

### 1. Creating Context

```jsx
const AuthContext = createContext()
```

- Creates a "container" for shared data
- Like a global variable, but better
- Components can "subscribe" to it

### 2. Custom Hook Pattern

```jsx
export const useAuth = () => {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

**Why This Pattern?**

- Makes it easier to use: `const { user } = useAuth()`
- Provides helpful error if used incorrectly
- Standard pattern in React community

### 3. Provider Component

```jsx
export const AuthProvider = ({ children }) => {
  // ... state and functions ...
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  )
}
```

**Key Points:**

- `children` = all components inside the provider
- `value` = data/functions to share
- Any component inside can access `value`

### 4. Using the Context

**In App.jsx:**
```jsx
import { AuthProvider } from './context/AuthContext'

function App() {
  return (
    <AuthProvider>
      {/* All these components can use useAuth() */}
      <Header />
      <Main />
      <Footer />
    </AuthProvider>
  )
}
```

**In Any Component:**
```jsx
import { useAuth } from '../context/AuthContext'

function Header() {
  const { user, logout, isAdmin } = useAuth()
  
  return (
    <header>
      <p>Welcome, {user?.username}!</p>
      {isAdmin() && <span>Admin</span>}
      <button onClick={logout}>Logout</button>
    </header>
  )
}
```

### Real Example: ThemeContext

Our theme context is simpler but follows the same pattern:

```jsx
// frontend/src/context/ThemeContext.jsx
const ThemeContext = createContext()

export const useTheme = () => {
  const context = useContext(ThemeContext)
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider')
  }
  return context
}

export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState(() => {
    // Load from localStorage or system preference
    const savedTheme = localStorage.getItem('theme')
    if (savedTheme) return savedTheme
    
    if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
      return 'dark'
    }
    return 'light'
  })

  // Save to localStorage when theme changes
  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme)
    localStorage.setItem('theme', theme)
  }, [theme])

  const toggleTheme = () => {
    setTheme(prevTheme => prevTheme === 'light' ? 'dark' : 'light')
  }

  const value = {
    theme,
    toggleTheme,
    isDark: theme === 'dark'
  }

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  )
}
```

**Usage:**
```jsx
function ThemeToggle() {
  const { theme, toggleTheme } = useTheme()
  
  return (
    <button onClick={toggleTheme}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  )
}
```

### When to Use Context?

✅ **Good Use Cases:**
- User authentication
- Theme (dark/light mode)
- Language/localization
- Shopping cart
- Any data needed by many components

❌ **Don't Use For:**
- Data that changes frequently (use state)
- Data only needed by 2-3 components (use props)
- Everything (causes unnecessary re-renders)

---

## Custom Hooks

### What are Custom Hooks?

Custom hooks let you extract component logic into reusable functions.

### Rules of Hooks

1. ✅ Must start with "use" (useAuth, useTheme, useData)
2. ✅ Only call at top level (not in loops/conditions)
3. ✅ Only call from React functions

### Real Example: useAuth Hook

We already saw this! It's a custom hook:

```jsx
export const useAuth = () => {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

**Why It's Useful:**

```jsx
// Without custom hook
const context = useContext(AuthContext)
if (!context) throw new Error('...')
const { user, login, logout } = context

// With custom hook
const { user, login, logout } = useAuth()  // Much cleaner!
```

### Creating Your Own Custom Hook

Let's create a `useLocalStorage` hook:

```jsx
import { useState, useEffect } from 'react'

function useLocalStorage(key, initialValue) {
  // Get from localStorage or use initial value
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key)
      return item ? JSON.parse(item) : initialValue
    } catch (error) {
      console.error(error)
      return initialValue
    }
  })

  // Save to localStorage when value changes
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue))
    } catch (error) {
      console.error(error)
    }
  }, [key, storedValue])

  return [storedValue, setStoredValue]
}

// Usage
function MyComponent() {
  const [name, setName] = useLocalStorage('name', 'Guest')
  
  return (
    <div>
      <input 
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <p>Hello, {name}!</p>
    </div>
  )
}
```

**Benefits:**

- Reusable across components
- Encapsulates complex logic
- Easier to test
- Cleaner component code

### Another Example: useFetch Hook

```jsx
function useFetch(url) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true)
        const response = await fetch(url)
        const json = await response.json()
        setData(json)
      } catch (err) {
        setError(err.message)
      } finally {
        setLoading(false)
      }
    }

    fetchData()
  }, [url])

  return { data, loading, error }
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`)
  
  if (loading) return <p>Loading...</p>
  if (error) return <p>Error: {error}</p>
  
  return <div>{user.name}</div>
}
```

---


## React Router - Navigation

### What is React Router?

React Router enables navigation between different pages/views in your single-page application (SPA).

### Real Example: Our App Router

Let's analyze our App.jsx routing setup:

```jsx
// frontend/src/App.jsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom'
import { ThemeProvider } from './context/ThemeContext'
import { AuthProvider } from './context/AuthContext'
import Header from './components/Header'
import Footer from './components/Footer'
import ProtectedRoute from './components/ProtectedRoute'
import Home from './pages/Home'
import EDForest from './pages/EDForest'
import History from './pages/History'
import About from './pages/About'
import Login from './pages/Login'
import Register from './pages/Register'
import NotFound from './pages/NotFound'

function App() {
  return (
    <ThemeProvider>
      <Router>
        <AuthProvider>
          <div className="flex flex-col min-h-screen">
            <Routes>
              {/* Public routes - No authentication required */}
              <Route path="/login" element={<Login />} />
              <Route path="/register" element={<Register />} />
              
              {/* Protected routes - Authentication required */}
              <Route path="/*" element={
                <>
                  <Header />
                  <main className="flex-grow">
                    <Routes>
                      <Route path="/" element={
                        <ProtectedRoute>
                          <Home />
                        </ProtectedRoute>
                      } />
                      <Route path="/edforest" element={
                        <ProtectedRoute>
                          <EDForest />
                        </ProtectedRoute>
                      } />
                      <Route path="/history" element={
                        <ProtectedRoute>
                          <History />
                        </ProtectedRoute>
                      } />
                      <Route path="/about" element={
                        <ProtectedRoute>
                          <About />
                        </ProtectedRoute>
                      } />
                      <Route path="*" element={<NotFound />} />
                    </Routes>
                  </main>
                  <Footer />
                </>
              } />
            </Routes>
          </div>
        </AuthProvider>
      </Router>
    </ThemeProvider>
  )
}
```

**Understanding the Structure:**

### 1. Router Component

```jsx
<Router>
  {/* All routing happens inside here */}
</Router>
```

- Wraps your entire app
- Enables routing functionality
- Must be at the top level

### 2. Routes and Route

```jsx
<Routes>
  <Route path="/login" element={<Login />} />
  <Route path="/home" element={<Home />} />
  <Route path="/about" element={<About />} />
</Routes>
```

- `<Routes>` = Container for all routes
- `<Route>` = Individual route definition
- `path` = URL path
- `element` = Component to render

### 3. Navigation with Link

From our Header component:

```jsx
import { Link, useLocation } from 'react-router-dom'

function Header() {
  const location = useLocation()
  
  const isActive = (path) => location.pathname === path
  
  return (
    <nav>
      <Link
        to="/"
        className={isActive('/') ? 'active' : ''}
      >
        Home
      </Link>
      
      <Link
        to="/edforest"
        className={isActive('/edforest') ? 'active' : ''}
      >
        EDForest
      </Link>
      
      <Link
        to="/history"
        className={isActive('/history') ? 'active' : ''}
      >
        History
      </Link>
    </nav>
  )
}
```

**Key Points:**

- Use `<Link>` instead of `<a>` tags
- `to` prop = destination path
- No page reload (SPA behavior)
- `useLocation()` = get current path

### 4. Programmatic Navigation

```jsx
import { useNavigate } from 'react-router-dom'

function LoginForm() {
  const navigate = useNavigate()
  
  const handleLogin = async () => {
    const success = await login(username, password)
    
    if (success) {
      navigate('/')  // Redirect to home
    }
  }
  
  return <button onClick={handleLogin}>Login</button>
}
```

**When to Use:**

- After form submission
- After successful login
- Conditional redirects
- Back button functionality

### 5. Route Parameters

```jsx
// Define route with parameter
<Route path="/user/:userId" element={<UserProfile />} />

// Access parameter in component
import { useParams } from 'react-router-dom'

function UserProfile() {
  const { userId } = useParams()
  
  return <div>User ID: {userId}</div>
}

// Navigate to: /user/123
// userId will be "123"
```

---

## Protected Routes

### The Problem

Some pages should only be accessible to logged-in users.

### Real Example: ProtectedRoute Component

```jsx
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom'
import { useAuth } from '../context/AuthContext'
import { Loader } from 'lucide-react'

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { isAuthenticated, isAdmin, loading } = useAuth()

  // Show loading while checking authentication
  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-center">
          <Loader className="w-12 h-12 text-primary-600 animate-spin mx-auto mb-4" />
          <p className="text-gray-600">Loading...</p>
        </div>
      </div>
    )
  }

  // Not authenticated? Redirect to login
  if (!isAuthenticated()) {
    return <Navigate to="/login" replace />
  }

  // Admin-only route but user is not admin?
  if (adminOnly && !isAdmin()) {
    return <Navigate to="/" replace />
  }

  // All checks passed, show the protected content
  return children
}

export default ProtectedRoute
```

**Understanding the Logic:**

### 1. Loading State

```jsx
if (loading) {
  return <Loader />
}
```

- While checking if user is logged in
- Prevents flash of wrong content
- Shows loading spinner

### 2. Authentication Check

```jsx
if (!isAuthenticated()) {
  return <Navigate to="/login" replace />
}
```

- If not logged in, redirect to login
- `replace` = don't add to history (can't go back)
- User must login to continue

### 3. Admin Check

```jsx
if (adminOnly && !isAdmin()) {
  return <Navigate to="/" replace />
}
```

- For admin-only pages
- Regular users redirected to home
- Prevents unauthorized access

### 4. Render Children

```jsx
return children
```

- All checks passed
- Show the protected content
- `children` = the wrapped component

### Usage

```jsx
// Protect a single route
<Route path="/dashboard" element={
  <ProtectedRoute>
    <Dashboard />
  </ProtectedRoute>
} />

// Admin-only route
<Route path="/admin" element={
  <ProtectedRoute adminOnly={true}>
    <AdminPanel />
  </ProtectedRoute>
} />
```

---

## API Integration Patterns

### Centralized API Service

Instead of calling `fetch()` everywhere, create a service layer.

### Real Example: api.js

```jsx
// frontend/src/services/api.js

const API_BASE_URL = 'http://localhost:5000/api/v1'

// Helper function to get auth token
const getAuthHeader = () => {
  const token = localStorage.getItem('token')
  return token ? { 'Authorization': `Bearer ${token}` } : {}
}

// Generic request function
const request = async (endpoint, options = {}) => {
  const url = `${API_BASE_URL}${endpoint}`
  
  const config = {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...getAuthHeader(),
      ...options.headers,
    },
  }

  try {
    const response = await fetch(url, config)
    const data = await response.json()

    if (!response.ok) {
      throw new Error(data.error || 'Request failed')
    }

    return data
  } catch (error) {
    console.error('API Error:', error)
    throw error
  }
}

// Authentication APIs
export const login = async (username, password) => {
  return request('/auth/login', {
    method: 'POST',
    body: JSON.stringify({ username, password }),
  })
}

export const register = async (username, email, password) => {
  return request('/auth/register', {
    method: 'POST',
    body: JSON.stringify({ username, email, password }),
  })
}

// EDForest APIs
export const uploadData = async (file) => {
  const formData = new FormData()
  formData.append('file', file)

  const token = localStorage.getItem('token')
  const response = await fetch(`${API_BASE_URL}/edforest/upload`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
    },
    body: formData,
  })

  return response.json()
}

export const analyzeBossung = async (filepath, targetCD, tolerance) => {
  return request('/edforest/analyze', {
    method: 'POST',
    body: JSON.stringify({ filepath, target_cd: targetCD, tolerance }),
  })
}

// History APIs
export const saveAnalysis = async (username, results, chartData, parameters) => {
  return request('/history/save', {
    method: 'POST',
    body: JSON.stringify({
      username,
      analysis_results: results,
      chart_data: chartData,
      parameters,
    }),
  })
}

export const getHistory = async () => {
  return request('/history/list')
}

export const getHistoryStats = async () => {
  return request('/history/stats')
}

export const deleteHistory = async (id) => {
  return request(`/history/${id}`, {
    method: 'DELETE',
  })
}
```

**Benefits of This Pattern:**

1. **Centralized Configuration:**
   - Base URL in one place
   - Auth headers automatically added
   - Easy to change API endpoint

2. **Error Handling:**
   - Consistent error handling
   - Easy to add logging
   - Can add retry logic

3. **Type Safety:**
   - Clear function signatures
   - Easy to document
   - IDE autocomplete works

4. **Reusability:**
   - Use same functions everywhere
   - No duplicate code
   - Easy to test

### Using the API Service

```jsx
import { login, getHistory, saveAnalysis } from '../services/api'

function MyComponent() {
  const handleLogin = async () => {
    try {
      const result = await login(username, password)
      console.log('Logged in:', result.user)
    } catch (error) {
      console.error('Login failed:', error.message)
    }
  }

  const loadHistory = async () => {
    try {
      const data = await getHistory()
      setHistory(data.history)
    } catch (error) {
      console.error('Failed to load history:', error.message)
    }
  }

  return (
    <div>
      <button onClick={handleLogin}>Login</button>
      <button onClick={loadHistory}>Load History</button>
    </div>
  )
}
```

---

## Advanced useEffect

### Dependency Array Deep Dive

The dependency array controls when useEffect runs.

### Pattern 1: Run Once on Mount

```jsx
useEffect(() => {
  console.log('Component mounted!')
  fetchData()
}, [])  // Empty array = run once
```

**Use Cases:**
- Fetch initial data
- Set up subscriptions
- Initialize third-party libraries

### Pattern 2: Run When Specific Values Change

```jsx
useEffect(() => {
  console.log('User changed:', userId)
  fetchUserData(userId)
}, [userId])  // Run when userId changes
```

**Use Cases:**
- Fetch data based on prop/state
- Update based on user selection
- Sync with external data

### Pattern 3: Run on Every Render

```jsx
useEffect(() => {
  console.log('Component rendered')
})  // No array = run every render
```

**⚠️ Usually avoid this!** Can cause performance issues.

### Real Example: Multiple Dependencies

From our EDForest page:

```jsx
const [targetCD, setTargetCD] = useState(45.0)
const [tolerance, setTolerance] = useState(10)
const [analysisResults, setAnalysisResults] = useState(null)

// Recalculate when parameters change
useEffect(() => {
  if (analysisResults) {
    // Recalculate with new parameters
    const updated = recalculateMetrics(analysisResults, targetCD, tolerance)
    setAnalysisResults(updated)
  }
}, [targetCD, tolerance])  // Run when either changes
```

### Cleanup Functions

Some effects need cleanup (timers, subscriptions, etc.)

```jsx
useEffect(() => {
  // Setup
  const timer = setInterval(() => {
    console.log('Tick')
  }, 1000)
  
  // Cleanup function
  return () => {
    clearInterval(timer)
    console.log('Cleanup!')
  }
}, [])
```

**When Cleanup Runs:**
- Before effect runs again
- When component unmounts
- Prevents memory leaks

### Real Example: Event Listeners

```jsx
useEffect(() => {
  const handleResize = () => {
    console.log('Window resized:', window.innerWidth)
  }
  
  // Add listener
  window.addEventListener('resize', handleResize)
  
  // Remove listener on cleanup
  return () => {
    window.removeEventListener('resize', handleResize)
  }
}, [])
```

### Common Mistakes

**Mistake 1: Missing Dependencies**

```jsx
// ❌ Wrong - missing dependency
useEffect(() => {
  fetchData(userId)
}, [])  // Should include userId!

// ✅ Correct
useEffect(() => {
  fetchData(userId)
}, [userId])
```

**Mistake 2: Infinite Loop**

```jsx
// ❌ Wrong - causes infinite loop
const [data, setData] = useState([])

useEffect(() => {
  setData([...data, 'new item'])
}, [data])  // Changes data, which triggers effect again!

// ✅ Correct - use functional update
useEffect(() => {
  setData(prev => [...prev, 'new item'])
}, [])  // Only run once
```

**Mistake 3: Async in useEffect**

```jsx
// ❌ Wrong - can't make useEffect async
useEffect(async () => {
  const data = await fetchData()
}, [])

// ✅ Correct - create async function inside
useEffect(() => {
  const loadData = async () => {
    const data = await fetchData()
    setData(data)
  }
  loadData()
}, [])
```

---

## Performance Optimization

### React.memo - Prevent Unnecessary Re-renders

`React.memo` prevents component from re-rendering if props haven't changed.

### Real Example: ToolCard Optimization

```jsx
import { memo } from 'react'

const ToolCard = memo(({ icon: Icon, title, description, features, status }) => {
  console.log('ToolCard rendered:', title)
  
  return (
    <div className="bg-white rounded-lg p-6">
      <Icon className="w-12 h-12" />
      <h3>{title}</h3>
      <p>{description}</p>
      <ul>
        {features.map((feature, index) => (
          <li key={index}>{feature}</li>
        ))}
      </ul>
    </div>
  )
})

export default ToolCard
```

**When to Use:**
- Component renders often
- Props don't change frequently
- Rendering is expensive
- Pure component (same props = same output)

### useMemo - Memoize Expensive Calculations

```jsx
import { useMemo } from 'react'

function DataTable({ data, filter }) {
  // Only recalculate when data or filter changes
  const filteredData = useMemo(() => {
    console.log('Filtering data...')
    return data.filter(item => item.name.includes(filter))
  }, [data, filter])
  
  return (
    <table>
      {filteredData.map(item => (
        <tr key={item.id}>
          <td>{item.name}</td>
        </tr>
      ))}
    </table>
  )
}
```

**Without useMemo:**
- Filters data on every render
- Even if data/filter didn't change
- Wastes CPU cycles

**With useMemo:**
- Only filters when data or filter changes
- Caches result
- Much faster!

### useCallback - Memoize Functions

```jsx
import { useCallback } from 'react'

function Parent() {
  const [count, setCount] = useState(0)
  
  // Without useCallback - new function every render
  const handleClick = () => {
    console.log('Clicked!')
  }
  
  // With useCallback - same function reference
  const handleClickMemo = useCallback(() => {
    console.log('Clicked!')
  }, [])  // Dependencies
  
  return (
    <div>
      <Child onClick={handleClickMemo} />
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
    </div>
  )
}

const Child = memo(({ onClick }) => {
  console.log('Child rendered')
  return <button onClick={onClick}>Click</button>
})
```

**Why It Matters:**
- Without `useCallback`, Child re-renders every time
- With `useCallback`, Child only renders when needed
- Important for optimized components

### When to Optimize?

✅ **Optimize When:**
- Performance issues noticed
- Component renders frequently
- Expensive calculations
- Large lists

❌ **Don't Optimize:**
- Premature optimization
- Simple components
- No performance issues
- Makes code harder to read

**Rule:** Measure first, optimize second!

---


## Component Composition

### What is Composition?

Composition is building complex components from simpler ones. It's like LEGO - combine small pieces to make something big.

### Pattern 1: Children Prop

The most basic composition pattern:

```jsx
function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  )
}

// Usage
<Card>
  <h2>Title</h2>
  <p>Content goes here</p>
  <button>Click me</button>
</Card>
```

### Pattern 2: Render Props

Pass a function as a prop:

```jsx
function DataFetcher({ url, render }) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data)
        setLoading(false)
      })
  }, [url])
  
  return render({ data, loading })
}

// Usage
<DataFetcher
  url="/api/users"
  render={({ data, loading }) => (
    loading ? <Spinner /> : <UserList users={data} />
  )}
/>
```

### Pattern 3: Compound Components

Components that work together:

```jsx
// Tabs component
function Tabs({ children }) {
  const [activeTab, setActiveTab] = useState(0)
  
  return (
    <div className="tabs">
      {React.Children.map(children, (child, index) =>
        React.cloneElement(child, {
          isActive: index === activeTab,
          onClick: () => setActiveTab(index)
        })
      )}
    </div>
  )
}

function Tab({ label, isActive, onClick, children }) {
  return (
    <div>
      <button 
        className={isActive ? 'active' : ''}
        onClick={onClick}
      >
        {label}
      </button>
      {isActive && <div>{children}</div>}
    </div>
  )
}

// Usage
<Tabs>
  <Tab label="Upload">
    <UploadForm />
  </Tab>
  <Tab label="Analyze">
    <AnalyzeForm />
  </Tab>
  <Tab label="Results">
    <Results />
  </Tab>
</Tabs>
```

### Real Example: Our EDForest Tabs

From our EDForest page:

```jsx
const [activeTab, setActiveTab] = useState('upload')

return (
  <div className="bg-white rounded-xl shadow-lg">
    {/* Tab Navigation */}
    <div className="border-b border-gray-200">
      <nav className="flex space-x-8 px-6">
        {['upload', 'analyze', 'results'].map((tab) => (
          <button
            key={tab}
            onClick={() => setActiveTab(tab)}
            className={`py-4 px-1 border-b-2 font-medium text-sm capitalize ${
              activeTab === tab
                ? 'border-primary-600 text-primary-600'
                : 'border-transparent text-gray-500'
            }`}
          >
            {tab}
          </button>
        ))}
      </nav>
    </div>

    {/* Tab Content */}
    <div className="p-6">
      {activeTab === 'upload' && <UploadTab />}
      {activeTab === 'analyze' && <AnalyzeTab />}
      {activeTab === 'results' && <ResultsTab />}
    </div>
  </div>
)
```

**Benefits:**
- Reusable tab system
- Clean separation of concerns
- Easy to add new tabs
- Consistent styling

---

## Error Boundaries

### What are Error Boundaries?

Error boundaries catch JavaScript errors in child components and display a fallback UI.

### Creating an Error Boundary

```jsx
import React from 'react'

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props)
    this.state = { hasError: false, error: null }
  }

  static getDerivedStateFromError(error) {
    // Update state so next render shows fallback UI
    return { hasError: true, error }
  }

  componentDidCatch(error, errorInfo) {
    // Log error to error reporting service
    console.error('Error caught by boundary:', error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      // Fallback UI
      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      )
    }

    return this.props.children
  }
}

// Usage
<ErrorBoundary>
  <MyComponent />
</ErrorBoundary>
```

**Note:** Error boundaries must be class components (for now).

### Where to Place Error Boundaries

```jsx
function App() {
  return (
    <ErrorBoundary>
      <Header />
      <ErrorBoundary>
        <Main />
      </ErrorBoundary>
      <Footer />
    </ErrorBoundary>
  )
}
```

**Strategy:**
- Top level: Catch all errors
- Around routes: Isolate page errors
- Around complex components: Prevent cascade failures

---

## Best Practices

### 1. Component Organization

**File Structure:**
```
src/
├── components/          # Reusable components
│   ├── Button.jsx
│   ├── Card.jsx
│   └── Header.jsx
├── pages/              # Page components
│   ├── Home.jsx
│   ├── Login.jsx
│   └── Dashboard.jsx
├── context/            # Context providers
│   ├── AuthContext.jsx
│   └── ThemeContext.jsx
├── services/           # API services
│   └── api.js
├── hooks/              # Custom hooks
│   └── useLocalStorage.js
└── utils/              # Helper functions
    └── formatDate.js
```

### 2. Naming Conventions

**Components:**
```jsx
// ✅ PascalCase for components
function UserProfile() {}
function ToolCard() {}

// ❌ Wrong
function userProfile() {}
function tool_card() {}
```

**Functions and Variables:**
```jsx
// ✅ camelCase for functions/variables
const handleClick = () => {}
const userData = {}

// ❌ Wrong
const HandleClick = () => {}
const UserData = {}
```

**Constants:**
```jsx
// ✅ UPPER_SNAKE_CASE for constants
const API_BASE_URL = 'http://localhost:5000'
const MAX_FILE_SIZE = 16 * 1024 * 1024

// ❌ Wrong
const apiBaseUrl = 'http://localhost:5000'
```

### 3. Props Destructuring

```jsx
// ✅ Destructure props
function UserCard({ name, email, avatar }) {
  return (
    <div>
      <img src={avatar} />
      <h3>{name}</h3>
      <p>{email}</p>
    </div>
  )
}

// ❌ Don't use props object
function UserCard(props) {
  return (
    <div>
      <img src={props.avatar} />
      <h3>{props.name}</h3>
      <p>{props.email}</p>
    </div>
  )
}
```

### 4. Conditional Rendering

```jsx
// ✅ Use && for simple conditions
{isLoggedIn && <UserMenu />}

// ✅ Use ternary for if/else
{isLoggedIn ? <UserMenu /> : <LoginButton />}

// ✅ Use early return for complex conditions
if (!data) return <Loading />
if (error) return <Error />
return <Content data={data} />

// ❌ Don't use if/else in JSX
{if (isLoggedIn) { return <UserMenu /> }}  // Syntax error!
```

### 5. State Management

```jsx
// ✅ Keep state close to where it's used
function TodoList() {
  const [todos, setTodos] = useState([])
  // todos only used in this component
}

// ✅ Lift state up when needed by multiple components
function App() {
  const [user, setUser] = useState(null)
  return (
    <>
      <Header user={user} />
      <Profile user={user} />
    </>
  )
}

// ✅ Use Context for global state
<AuthProvider>
  <App />
</AuthProvider>
```

### 6. useEffect Dependencies

```jsx
// ✅ Include all dependencies
useEffect(() => {
  fetchData(userId, filter)
}, [userId, filter])

// ❌ Missing dependencies
useEffect(() => {
  fetchData(userId, filter)
}, [userId])  // Missing filter!

// ✅ Use ESLint plugin to catch this
// npm install eslint-plugin-react-hooks
```

### 7. Key Prop in Lists

```jsx
// ✅ Use unique ID
{users.map(user => (
  <UserCard key={user.id} {...user} />
))}

// ⚠️ Use index only if no unique ID
{items.map((item, index) => (
  <Item key={index} {...item} />
))}

// ❌ Never use random values
{items.map(item => (
  <Item key={Math.random()} {...item} />
))}
```

### 8. Avoid Inline Functions in JSX

```jsx
// ❌ Creates new function on every render
<button onClick={() => handleClick(id)}>Click</button>

// ✅ Use useCallback for optimization
const handleClickMemo = useCallback(() => {
  handleClick(id)
}, [id])

<button onClick={handleClickMemo}>Click</button>

// ✅ Or just pass the function if no parameters
<button onClick={handleClick}>Click</button>
```

### 9. PropTypes or TypeScript

```jsx
// Option 1: PropTypes
import PropTypes from 'prop-types'

function UserCard({ name, age, email }) {
  return <div>{name}</div>
}

UserCard.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
  email: PropTypes.string.isRequired,
}

// Option 2: TypeScript (better!)
interface UserCardProps {
  name: string
  age?: number
  email: string
}

function UserCard({ name, age, email }: UserCardProps) {
  return <div>{name}</div>
}
```

### 10. Code Splitting

```jsx
import { lazy, Suspense } from 'react'

// ✅ Lazy load heavy components
const HeavyChart = lazy(() => import('./components/HeavyChart'))

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyChart />
    </Suspense>
  )
}
```

---

## Real-World Example: Complete Component

Let's build a complete, production-ready component using all best practices:

```jsx
// components/UserProfile.jsx
import { useState, useEffect, useCallback, memo } from 'react'
import { useNavigate } from 'react-router-dom'
import { User, Mail, Phone, Edit, Loader } from 'lucide-react'
import { getUserProfile, updateUserProfile } from '../services/api'
import { useAuth } from '../context/AuthContext'

/**
 * UserProfile Component
 * Displays and allows editing of user profile information
 * 
 * @param {string} userId - The ID of the user to display
 * @param {boolean} editable - Whether the profile can be edited
 */
const UserProfile = memo(({ userId, editable = false }) => {
  // State
  const [profile, setProfile] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)
  const [isEditing, setIsEditing] = useState(false)
  
  // Hooks
  const { user: currentUser } = useAuth()
  const navigate = useNavigate()
  
  // Check if current user can edit this profile
  const canEdit = editable && currentUser?.id === userId
  
  // Load profile data
  useEffect(() => {
    const loadProfile = async () => {
      try {
        setLoading(true)
        setError(null)
        const data = await getUserProfile(userId)
        setProfile(data)
      } catch (err) {
        setError(err.message)
      } finally {
        setLoading(false)
      }
    }
    
    loadProfile()
  }, [userId])
  
  // Handle edit toggle
  const handleEditToggle = useCallback(() => {
    setIsEditing(prev => !prev)
  }, [])
  
  // Handle save
  const handleSave = useCallback(async (updatedData) => {
    try {
      setLoading(true)
      const updated = await updateUserProfile(userId, updatedData)
      setProfile(updated)
      setIsEditing(false)
    } catch (err) {
      setError(err.message)
    } finally {
      setLoading(false)
    }
  }, [userId])
  
  // Loading state
  if (loading && !profile) {
    return (
      <div className="flex items-center justify-center p-8">
        <Loader className="w-8 h-8 animate-spin text-primary-600" />
      </div>
    )
  }
  
  // Error state
  if (error) {
    return (
      <div className="bg-red-50 border border-red-200 rounded-lg p-4">
        <p className="text-red-700">Error: {error}</p>
        <button 
          onClick={() => navigate(-1)}
          className="mt-2 text-red-600 hover:underline"
        >
          Go Back
        </button>
      </div>
    )
  }
  
  // No data state
  if (!profile) {
    return (
      <div className="text-center p-8">
        <p className="text-gray-500">Profile not found</p>
      </div>
    )
  }
  
  // Main render
  return (
    <div className="bg-white rounded-lg shadow-lg p-6">
      {/* Header */}
      <div className="flex items-center justify-between mb-6">
        <h2 className="text-2xl font-bold text-gray-900">
          User Profile
        </h2>
        {canEdit && (
          <button
            onClick={handleEditToggle}
            className="flex items-center space-x-2 px-4 py-2 bg-primary-600 text-white rounded-lg hover:bg-primary-700"
          >
            <Edit className="w-4 h-4" />
            <span>{isEditing ? 'Cancel' : 'Edit'}</span>
          </button>
        )}
      </div>
      
      {/* Profile Content */}
      {isEditing ? (
        <ProfileEditForm 
          profile={profile}
          onSave={handleSave}
          onCancel={handleEditToggle}
        />
      ) : (
        <ProfileView profile={profile} />
      )}
    </div>
  )
})

// Display name for debugging
UserProfile.displayName = 'UserProfile'

export default UserProfile

// Sub-components
function ProfileView({ profile }) {
  return (
    <div className="space-y-4">
      <div className="flex items-center space-x-3">
        <User className="w-5 h-5 text-gray-400" />
        <div>
          <p className="text-sm text-gray-500">Name</p>
          <p className="font-medium text-gray-900">{profile.name}</p>
        </div>
      </div>
      
      <div className="flex items-center space-x-3">
        <Mail className="w-5 h-5 text-gray-400" />
        <div>
          <p className="text-sm text-gray-500">Email</p>
          <p className="font-medium text-gray-900">{profile.email}</p>
        </div>
      </div>
      
      <div className="flex items-center space-x-3">
        <Phone className="w-5 h-5 text-gray-400" />
        <div>
          <p className="text-sm text-gray-500">Phone</p>
          <p className="font-medium text-gray-900">{profile.phone || 'Not provided'}</p>
        </div>
      </div>
    </div>
  )
}

function ProfileEditForm({ profile, onSave, onCancel }) {
  const [formData, setFormData] = useState(profile)
  
  const handleSubmit = (e) => {
    e.preventDefault()
    onSave(formData)
  }
  
  const handleChange = (field, value) => {
    setFormData(prev => ({ ...prev, [field]: value }))
  }
  
  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Name
        </label>
        <input
          type="text"
          value={formData.name}
          onChange={(e) => handleChange('name', e.target.value)}
          className="w-full px-4 py-2 border border-gray-300 rounded-lg"
          required
        />
      </div>
      
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Email
        </label>
        <input
          type="email"
          value={formData.email}
          onChange={(e) => handleChange('email', e.target.value)}
          className="w-full px-4 py-2 border border-gray-300 rounded-lg"
          required
        />
      </div>
      
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Phone
        </label>
        <input
          type="tel"
          value={formData.phone || ''}
          onChange={(e) => handleChange('phone', e.target.value)}
          className="w-full px-4 py-2 border border-gray-300 rounded-lg"
        />
      </div>
      
      <div className="flex space-x-3">
        <button
          type="submit"
          className="flex-1 bg-primary-600 text-white py-2 rounded-lg hover:bg-primary-700"
        >
          Save Changes
        </button>
        <button
          type="button"
          onClick={onCancel}
          className="flex-1 bg-gray-200 text-gray-700 py-2 rounded-lg hover:bg-gray-300"
        >
          Cancel
        </button>
      </div>
    </form>
  )
}
```

**What Makes This Component Good:**

1. ✅ **Proper imports** - Organized and clear
2. ✅ **JSDoc comments** - Documents props
3. ✅ **memo** - Optimized for performance
4. ✅ **Custom hooks** - useAuth for authentication
5. ✅ **useCallback** - Memoized functions
6. ✅ **Error handling** - Proper error states
7. ✅ **Loading states** - Good UX
8. ✅ **Conditional rendering** - Clean and readable
9. ✅ **Sub-components** - Separated concerns
10. ✅ **Accessibility** - Proper labels and semantics

---

## Summary - Key Takeaways

### Advanced Concepts Mastered:

1. ✅ **Context API** - Global state management
2. ✅ **Custom Hooks** - Reusable logic
3. ✅ **React Router** - Navigation and routing
4. ✅ **Protected Routes** - Authentication guards
5. ✅ **API Integration** - Centralized service layer
6. ✅ **Advanced useEffect** - Dependencies and cleanup
7. ✅ **Performance** - memo, useMemo, useCallback
8. ✅ **Composition** - Building complex UIs
9. ✅ **Error Boundaries** - Error handling
10. ✅ **Best Practices** - Production-ready code

### Next Steps:

1. **Practice** - Build your own features
2. **Read Code** - Study our project thoroughly
3. **Experiment** - Try different patterns
4. **Optimize** - Measure and improve performance
5. **Test** - Learn React Testing Library
6. **TypeScript** - Add type safety
7. **State Management** - Learn Redux/Zustand
8. **Server State** - Learn React Query

---

## Additional Resources

### Official Documentation
- React Docs: https://react.dev
- React Router: https://reactrouter.com
- Vite: https://vitejs.dev

### Learning Platforms
- React Tutorial: https://react.dev/learn
- FreeCodeCamp React Course
- Scrimba React Course

### Community
- React Discord
- Stack Overflow
- Reddit r/reactjs

---

**Congratulations!** 🎉

You've completed the Advanced React Training! You now have the skills to build production-ready React applications. Keep practicing and building!

**Remember:** The best way to learn is by doing. Start building your own features in this project!
