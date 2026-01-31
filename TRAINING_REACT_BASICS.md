# 📚 React Programming Training - BASICS

## For Engineers with No Experience

**Target Audience:** Complete Beginners, No React Experience  
**Prerequisites:** Basic JavaScript (variables, functions, arrays)  
**Learning Time:** 2-3 weeks  
**Difficulty:** Beginner

---

## Table of Contents
1. [What is React?](#what-is-react)
2. [Setting Up Your First Component](#setting-up-your-first-component)
3. [JSX - JavaScript XML](#jsx---javascript-xml)
4. [Components and Props](#components-and-props)
5. [State Management with useState](#state-management-with-usestate)
6. [Event Handling](#event-handling)
7. [Conditional Rendering](#conditional-rendering)
8. [Lists and Keys](#lists-and-keys)
9. [Forms and Inputs](#forms-and-inputs)
10. [useEffect Hook](#useeffect-hook)

---

## What is React?

React is a **JavaScript library** for building user interfaces. Think of it like building with LEGO blocks:
- Each LEGO block = A React Component
- You combine blocks = Build complex UIs
- Reuse blocks = Write less code

### Why React?
- ✅ **Component-Based:** Break UI into reusable pieces
- ✅ **Declarative:** Describe what you want, React handles how
- ✅ **Fast:** Virtual DOM makes updates efficient
- ✅ **Popular:** Huge community and job market

---

## Setting Up Your First Component

### Understanding main.jsx (Entry Point)

Let's look at our project's entry point:

```jsx
// frontend/src/main.jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

**Line-by-Line Explanation:**

1. **Import React:**
   ```jsx
   import { StrictMode } from 'react'
   ```
   - `StrictMode` helps catch bugs during development
   - Like a safety net for your code

2. **Import ReactDOM:**
   ```jsx
   import { createRoot } from 'react-dom/client'
   ```
   - `createRoot` connects React to your HTML
   - It "mounts" your React app to the DOM

3. **Import CSS:**
   ```jsx
   import './index.css'
   ```
   - Global styles for your entire app
   - Loaded once at the start

4. **Import Main Component:**
   ```jsx
   import App from './App.jsx'
   ```
   - Your main application component
   - The root of your component tree

5. **Render to DOM:**
   ```jsx
   createRoot(document.getElementById('root')).render(
     <StrictMode>
       <App />
     </StrictMode>,
   )
   ```
   - Finds `<div id="root">` in your HTML
   - Renders your React app inside it
   - Everything starts here!

### Your First Component

The simplest component is a function that returns JSX:

```jsx
function Welcome() {
  return <h1>Hello, World!</h1>
}
```

**That's it!** A component is just a function that returns what you want to show.

---

## JSX - JavaScript XML

JSX looks like HTML but it's actually JavaScript. React transforms it into real HTML.

### Basic JSX Rules

**Rule 1: Must return ONE parent element**

❌ **Wrong:**
```jsx
function MyComponent() {
  return (
    <h1>Title</h1>
    <p>Paragraph</p>
  )
}
```

✅ **Correct:**
```jsx
function MyComponent() {
  return (
    <div>
      <h1>Title</h1>
      <p>Paragraph</p>
    </div>
  )
}
```

**Rule 2: Use `className` instead of `class`**

❌ **Wrong:**
```jsx
<div class="container">Content</div>
```

✅ **Correct:**
```jsx
<div className="container">Content</div>
```

**Rule 3: Close all tags**

❌ **Wrong:**
```jsx
<img src="photo.jpg">
<input type="text">
```

✅ **Correct:**
```jsx
<img src="photo.jpg" />
<input type="text" />
```

### JavaScript in JSX

Use curly braces `{}` to write JavaScript inside JSX:

```jsx
function Greeting() {
  const name = "Daniel"
  const age = 25
  
  return (
    <div>
      <h1>Hello, {name}!</h1>
      <p>You are {age} years old</p>
      <p>Next year you'll be {age + 1}</p>
    </div>
  )
}
```

**Output:**
```
Hello, Daniel!
You are 25 years old
Next year you'll be 26
```

---

## Components and Props

### What are Props?

Props (properties) are how you pass data to components. Think of them like function parameters.

### Real Example from Our Project: ToolCard

Let's look at our `ToolCard` component:

```jsx
// frontend/src/components/ToolCard.jsx
const ToolCard = ({ icon: Icon, title, description, features, link, status, caseCount, userCount }) => {
  return (
    <div className="bg-white rounded-lg p-6 shadow-md">
      {/* Icon */}
      <div className="mb-4">
        <Icon className="w-12 h-12 text-primary-600" />
      </div>
      
      {/* Title */}
      <h3 className="text-xl font-bold text-gray-900 mb-2">
        {title}
      </h3>
      
      {/* Description */}
      <p className="text-gray-600 text-sm mb-4">
        {description}
      </p>
      
      {/* Features List */}
      <ul className="space-y-2 mb-4">
        {features.map((feature, index) => (
          <li key={index} className="text-sm text-gray-700">
            • {feature}
          </li>
        ))}
      </ul>
      
      {/* Statistics */}
      {status === 'active' && (
        <div className="flex space-x-4 text-xs text-gray-500 mb-4">
          <span>Cases: {caseCount}</span>
          <span>Users: {userCount}</span>
        </div>
      )}
    </div>
  )
}
```

**How to Use This Component:**

```jsx
// In Home.jsx
<ToolCard
  icon={BarChart3}
  title="EDForest - Bossung Curve Analysis"
  description="Generate professional Bossung curves"
  features={[
    'Upload CSV data',
    'Interactive charts',
    'Export results'
  ]}
  link="/edforest"
  status="active"
  caseCount={42}
  userCount={5}
/>
```

**Understanding Props:**

1. **Destructuring Props:**
   ```jsx
   const ToolCard = ({ icon: Icon, title, description }) => {
   ```
   - Instead of `props.title`, we use `title` directly
   - `icon: Icon` renames `icon` to `Icon` (capital I)
   - This is called "destructuring"

2. **Using Props:**
   ```jsx
   <h3>{title}</h3>
   <p>{description}</p>
   ```
   - Props are just variables you can use in JSX
   - Wrap them in `{}` to display their values

3. **Props are Read-Only:**
   ```jsx
   // ❌ Never do this!
   title = "New Title"  // ERROR!
   
   // ✅ Props can't be changed
   // They come from the parent component
   ```

---

## State Management with useState

### What is State?

State is data that can change over time. When state changes, React re-renders your component.

### Real Example: Login Form

Let's look at our Login page:

```jsx
// frontend/src/pages/Login.jsx (simplified)
import { useState } from 'react'

const Login = () => {
  // Declare state variables
  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [loading, setLoading] = useState(false)

  const handleSubmit = async (e) => {
    e.preventDefault()
    setLoading(true)
    setError('')
    
    // Try to login
    const result = await login(username, password)
    
    if (result.success) {
      // Success! Navigate to home
      navigate('/')
    } else {
      // Show error message
      setError(result.error)
    }
    
    setLoading(false)
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* Show error if exists */}
      {error && (
        <div className="error-message">
          {error}
        </div>
      )}
      
      {/* Username input */}
      <input
        type="text"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
        placeholder="Username"
      />
      
      {/* Password input */}
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      
      {/* Submit button */}
      <button type="submit" disabled={loading}>
        {loading ? 'Logging in...' : 'Login'}
      </button>
    </form>
  )
}
```

**Understanding useState:**

1. **Declaring State:**
   ```jsx
   const [username, setUsername] = useState('')
   ```
   - `username` = current value (starts as empty string '')
   - `setUsername` = function to update the value
   - `useState('')` = initial value

2. **Reading State:**
   ```jsx
   <input value={username} />
   <p>You typed: {username}</p>
   ```
   - Just use the variable name
   - React automatically updates when it changes

3. **Updating State:**
   ```jsx
   setUsername('Daniel')  // Set to specific value
   setLoading(true)       // Set to true
   setError('')           // Clear error (empty string)
   ```
   - Call the setter function
   - React re-renders the component
   - UI updates automatically!

4. **State is Isolated:**
   ```jsx
   // Each component has its own state
   <Login />  // Has its own username/password
   <Login />  // Different username/password
   ```

### Common State Patterns

**Pattern 1: Toggle Boolean**
```jsx
const [isOpen, setIsOpen] = useState(false)

// Toggle on/off
setIsOpen(!isOpen)  // If true, becomes false. If false, becomes true.
```

**Pattern 2: Update from Input**
```jsx
const [text, setText] = useState('')

<input 
  value={text}
  onChange={(e) => setText(e.target.value)}
/>
```

**Pattern 3: Multiple State Variables**
```jsx
const [name, setName] = useState('')
const [age, setAge] = useState(0)
const [email, setEmail] = useState('')

// Each has its own setter
setName('Daniel')
setAge(25)
setEmail('daniel@vsmc.com')
```

---


## Event Handling

### What are Events?

Events are things that happen: clicks, typing, hovering, etc. React handles them similar to HTML but with some differences.

### Real Example: Theme Toggle Button

Let's look at our ThemeToggle component:

```jsx
// frontend/src/components/ThemeToggle.jsx
import { Sun, Moon } from 'lucide-react'
import { useTheme } from '../context/ThemeContext'

const ThemeToggle = () => {
  const { theme, toggleTheme } = useTheme()

  return (
    <button
      onClick={toggleTheme}
      className="p-2 rounded-lg bg-gray-100"
      title={`Switch to ${theme === 'light' ? 'dark' : 'light'} mode`}
    >
      {theme === 'light' ? (
        <Moon className="w-5 h-5" />
      ) : (
        <Sun className="w-5 h-5" />
      )}
    </button>
  )
}
```

**Understanding Events:**

1. **onClick Event:**
   ```jsx
   <button onClick={toggleTheme}>
   ```
   - `onClick` = React's click event (camelCase!)
   - `toggleTheme` = function to call when clicked
   - **No parentheses!** `onClick={toggleTheme}` not `onClick={toggleTheme()}`

2. **Why No Parentheses?**
   ```jsx
   // ❌ Wrong - calls function immediately
   <button onClick={toggleTheme()}>
   
   // ✅ Correct - passes function reference
   <button onClick={toggleTheme}>
   
   // ✅ Also correct - arrow function
   <button onClick={() => toggleTheme()}>
   ```

3. **Event with Parameters:**
   ```jsx
   // If you need to pass parameters, use arrow function
   <button onClick={() => deleteUser(userId)}>
     Delete
   </button>
   ```

### Common Events

**onClick - Button Clicks**
```jsx
<button onClick={handleClick}>Click Me</button>
```

**onChange - Input Changes**
```jsx
<input 
  value={text}
  onChange={(e) => setText(e.target.value)}
/>
```

**onSubmit - Form Submission**
```jsx
<form onSubmit={handleSubmit}>
  <button type="submit">Submit</button>
</form>
```

**onMouseEnter/onMouseLeave - Hover**
```jsx
<div 
  onMouseEnter={() => setHovered(true)}
  onMouseLeave={() => setHovered(false)}
>
  Hover me!
</div>
```

### Real Example: Form Submission

From our Login page:

```jsx
const handleSubmit = async (e) => {
  // Prevent page reload (default form behavior)
  e.preventDefault()
  
  // Your logic here
  setLoading(true)
  const result = await login(username, password)
  setLoading(false)
}

return (
  <form onSubmit={handleSubmit}>
    <input type="text" value={username} onChange={(e) => setUsername(e.target.value)} />
    <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
    <button type="submit">Login</button>
  </form>
)
```

**Key Points:**

1. **e.preventDefault():**
   - Stops default behavior (page reload for forms)
   - Always use this for form submissions

2. **e.target.value:**
   - Gets the current value from input
   - `e` = event object
   - `target` = the element that triggered the event
   - `value` = current value of that element

---

## Conditional Rendering

### Show/Hide Based on Conditions

React lets you show different things based on conditions.

### Real Example: Error Messages

From our Login page:

```jsx
const [error, setError] = useState('')

return (
  <div>
    {/* Only show if error exists */}
    {error && (
      <div className="bg-red-50 border border-red-200 rounded-lg p-4">
        <AlertCircle className="w-5 h-5 text-red-600" />
        <p className="text-sm text-red-700">{error}</p>
      </div>
    )}
    
    {/* Rest of form */}
  </div>
)
```

**Understanding `&&` Operator:**

```jsx
{error && <div>Error message</div>}
```

- If `error` is empty string (''), nothing shows
- If `error` has text, the div shows
- This is called "short-circuit evaluation"

### Ternary Operator (? :)

Show one thing OR another:

```jsx
// From ThemeToggle.jsx
{theme === 'light' ? (
  <Moon className="w-5 h-5" />
) : (
  <Sun className="w-5 h-5" />
)}
```

**Reads as:** "If theme is light, show Moon icon, otherwise show Sun icon"

### Real Example: Loading States

From our Login button:

```jsx
const [loading, setLoading] = useState(false)

<button type="submit" disabled={loading}>
  {loading ? (
    <>
      <Loader className="w-5 h-5 mr-2 animate-spin" />
      Logging in...
    </>
  ) : (
    <>
      <LogIn className="w-5 h-5 mr-2" />
      Login
    </>
  )}
</button>
```

**What's Happening:**

1. When `loading` is false: Shows "Login" with LogIn icon
2. When `loading` is true: Shows "Logging in..." with spinning Loader
3. Button is disabled when loading (can't click twice)

### Multiple Conditions

```jsx
{!loading && !error && <SuccessMessage />}
{loading && <Spinner />}
{error && <ErrorMessage />}
```

---

## Lists and Keys

### Rendering Arrays

React can render arrays of elements automatically.

### Real Example: Tool Cards

From our Home page:

```jsx
const tools = [
  {
    icon: BarChart3,
    title: 'EDForest - Bossung Curve Analysis',
    description: 'Generate professional Bossung curves',
    features: ['Upload CSV', 'Interactive charts', 'Export results'],
    link: '/edforest',
    status: 'active'
  },
  {
    icon: Target,
    title: 'Focus-Exposure Matrix (FEM)',
    description: 'Analyze dose-focus matrices',
    features: ['Contour maps', 'Optimal point detection'],
    link: '/fem',
    status: 'coming-soon'
  },
  // ... more tools
]

return (
  <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
    {tools.map((tool, index) => (
      <ToolCard 
        key={index}
        icon={tool.icon}
        title={tool.title}
        description={tool.description}
        features={tool.features}
        link={tool.link}
        status={tool.status}
      />
    ))}
  </div>
)
```

**Understanding .map():**

1. **What is .map()?**
   ```jsx
   array.map((item, index) => {
     // Return JSX for each item
   })
   ```
   - Loops through each item in array
   - Returns new array of JSX elements
   - React renders all of them

2. **The Key Prop:**
   ```jsx
   <ToolCard key={index} ... />
   ```
   - **Required** for list items
   - Helps React track which items changed
   - Should be unique for each item
   - Usually use `id` or `index`

3. **Why Keys Matter:**
   ```jsx
   // ❌ Without key - React gets confused
   {tools.map(tool => <ToolCard {...tool} />)}
   
   // ✅ With key - React knows which is which
   {tools.map((tool, index) => <ToolCard key={index} {...tool} />)}
   ```

### Real Example: Features List

From ToolCard component:

```jsx
<ul className="space-y-2 mb-4">
  {features.map((feature, index) => (
    <li key={index} className="text-sm text-gray-700">
      • {feature}
    </li>
  ))}
</ul>
```

**Input:**
```jsx
features = ['Upload CSV', 'Interactive charts', 'Export results']
```

**Output:**
```html
<ul>
  <li>• Upload CSV</li>
  <li>• Interactive charts</li>
  <li>• Export results</li>
</ul>
```

### Filtering Lists

Show only items that match a condition:

```jsx
// Show only active tools
{tools.filter(tool => tool.status === 'active').map((tool, index) => (
  <ToolCard key={index} {...tool} />
))}
```

---

## Forms and Inputs

### Controlled Components

In React, form inputs are "controlled" by state.

### Real Example: Registration Form

From our Register page:

```jsx
const Register = () => {
  // State for each input
  const [username, setUsername] = useState('')
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [confirmPassword, setConfirmPassword] = useState('')
  const [error, setError] = useState('')

  const handleSubmit = async (e) => {
    e.preventDefault()
    setError('')

    // Validation
    if (password !== confirmPassword) {
      setError('Passwords do not match')
      return
    }

    if (password.length < 6) {
      setError('Password must be at least 6 characters')
      return
    }

    // Submit form
    const result = await register(username, email, password)
    
    if (result.success) {
      navigate('/')
    } else {
      setError(result.error)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* Username Input */}
      <div>
        <label>Username</label>
        <input
          type="text"
          value={username}
          onChange={(e) => setUsername(e.target.value)}
          placeholder="Choose a username"
          required
        />
      </div>

      {/* Email Input */}
      <div>
        <label>Email</label>
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          placeholder="your.email@company.com"
          required
        />
      </div>

      {/* Password Input */}
      <div>
        <label>Password</label>
        <input
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          placeholder="Create a password"
          required
        />
      </div>

      {/* Confirm Password */}
      <div>
        <label>Confirm Password</label>
        <input
          type="password"
          value={confirmPassword}
          onChange={(e) => setConfirmPassword(e.target.value)}
          placeholder="Confirm your password"
          required
        />
        {/* Visual feedback */}
        {password && confirmPassword && password === confirmPassword && (
          <CheckCircle className="text-green-500" />
        )}
      </div>

      {/* Submit Button */}
      <button type="submit">
        Create Account
      </button>
    </form>
  )
}
```

**Key Concepts:**

1. **Controlled Input Pattern:**
   ```jsx
   const [value, setValue] = useState('')
   
   <input
     value={value}
     onChange={(e) => setValue(e.target.value)}
   />
   ```
   - State controls the input value
   - Input updates state on change
   - This is called "two-way binding"

2. **Form Validation:**
   ```jsx
   if (password !== confirmPassword) {
     setError('Passwords do not match')
     return  // Stop here, don't submit
   }
   ```
   - Check conditions before submitting
   - Show error messages
   - Prevent invalid submissions

3. **Visual Feedback:**
   ```jsx
   {password && confirmPassword && password === confirmPassword && (
     <CheckCircle className="text-green-500" />
   )}
   ```
   - Show checkmark when passwords match
   - Helps user know they're doing it right

---

## useEffect Hook

### What is useEffect?

`useEffect` runs code after your component renders. Use it for:
- Fetching data from APIs
- Setting up subscriptions
- Updating the document title
- Running code when something changes

### Basic Syntax

```jsx
import { useEffect } from 'react'

useEffect(() => {
  // Code to run
  console.log('Component rendered!')
}, [])  // Dependencies array
```

### Real Example: Loading User Data

From our Home page:

```jsx
import { useState, useEffect } from 'react'

const Home = () => {
  const [stats, setStats] = useState({
    total_analyses: 0,
    unique_users: 0,
    monthly_count: 0
  })
  const [loading, setLoading] = useState(true)

  // Run when component mounts
  useEffect(() => {
    loadStats()
  }, [])  // Empty array = run once on mount

  const loadStats = async () => {
    try {
      const result = await getHistoryStats()
      if (result.success) {
        setStats(result.stats)
      }
    } catch (error) {
      console.error('Failed to load stats:', error)
    } finally {
      setLoading(false)
    }
  }

  return (
    <div>
      {loading ? (
        <p>Loading...</p>
      ) : (
        <div>
          <p>Total Analyses: {stats.total_analyses}</p>
          <p>Unique Users: {stats.unique_users}</p>
          <p>This Month: {stats.monthly_count}</p>
        </div>
      )}
    </div>
  )
}
```

**Understanding useEffect:**

1. **When Does It Run?**
   ```jsx
   useEffect(() => {
     console.log('I run after render!')
   }, [])
   ```
   - After component renders
   - After DOM updates
   - Not during render (that would cause errors!)

2. **Dependencies Array:**
   ```jsx
   // Run once on mount
   useEffect(() => {}, [])
   
   // Run when 'count' changes
   useEffect(() => {}, [count])
   
   // Run on every render (usually avoid this!)
   useEffect(() => {})
   ```

3. **Cleanup Function:**
   ```jsx
   useEffect(() => {
     // Setup
     const timer = setInterval(() => {
       console.log('Tick')
     }, 1000)
     
     // Cleanup (runs when component unmounts)
     return () => {
       clearInterval(timer)
     }
   }, [])
   ```

### Real Example: Theme Persistence

From our ThemeContext:

```jsx
const [theme, setTheme] = useState(() => {
  // Check localStorage first
  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) {
    return savedTheme
  }
  return 'light'
})

// Save to localStorage when theme changes
useEffect(() => {
  document.documentElement.setAttribute('data-theme', theme)
  localStorage.setItem('theme', theme)
}, [theme])  // Run when 'theme' changes
```

**What's Happening:**

1. **Initial State:**
   - Check if theme was saved before
   - Use saved theme or default to 'light'

2. **useEffect with Dependency:**
   - Runs when `theme` changes
   - Updates the HTML attribute
   - Saves to localStorage
   - Next time user visits, theme is remembered!

### Common useEffect Patterns

**Pattern 1: Fetch Data on Mount**
```jsx
useEffect(() => {
  fetchData()
}, [])
```

**Pattern 2: Update When Prop Changes**
```jsx
useEffect(() => {
  console.log('User changed:', userId)
  fetchUserData(userId)
}, [userId])
```

**Pattern 3: Cleanup Subscriptions**
```jsx
useEffect(() => {
  const subscription = subscribeToData()
  
  return () => {
    subscription.unsubscribe()
  }
}, [])
```

---

## Practice Exercises

### Exercise 1: Counter Component

Create a simple counter:

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
      <button onClick={() => setCount(count - 1)}>
        Decrement
      </button>
      <button onClick={() => setCount(0)}>
        Reset
      </button>
    </div>
  )
}
```

### Exercise 2: Todo List

Create a simple todo list:

```jsx
function TodoList() {
  const [todos, setTodos] = useState([])
  const [input, setInput] = useState('')
  
  const addTodo = () => {
    if (input.trim()) {
      setTodos([...todos, input])
      setInput('')
    }
  }
  
  return (
    <div>
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        placeholder="Enter todo"
      />
      <button onClick={addTodo}>Add</button>
      
      <ul>
        {todos.map((todo, index) => (
          <li key={index}>{todo}</li>
        ))}
      </ul>
    </div>
  )
}
```

### Exercise 3: Show/Hide Component

Toggle visibility:

```jsx
function ToggleContent() {
  const [isVisible, setIsVisible] = useState(false)
  
  return (
    <div>
      <button onClick={() => setIsVisible(!isVisible)}>
        {isVisible ? 'Hide' : 'Show'} Content
      </button>
      
      {isVisible && (
        <div>
          <p>This content can be toggled!</p>
        </div>
      )}
    </div>
  )
}
```

---

## Summary - Key Takeaways

### What You Learned:

1. ✅ **Components** - Functions that return JSX
2. ✅ **JSX** - HTML-like syntax in JavaScript
3. ✅ **Props** - Pass data to components
4. ✅ **State** - Data that can change (useState)
5. ✅ **Events** - Handle user interactions
6. ✅ **Conditional Rendering** - Show/hide based on conditions
7. ✅ **Lists** - Render arrays with .map()
8. ✅ **Forms** - Controlled inputs with state
9. ✅ **useEffect** - Run code after render

### Next Steps:

1. Practice the exercises above
2. Read through our project code
3. Try modifying existing components
4. Move on to Advanced React Training

---

**Congratulations!** 🎉

You now understand React basics! You're ready to build simple applications and understand most React code you'll encounter.

Continue to **TRAINING_REACT_ADVANCED.md** for more advanced concepts!
