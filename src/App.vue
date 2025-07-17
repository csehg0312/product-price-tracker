<template>
  <div id="app">
    <div class="container">
      <h1>Product Price Tracker</h1>
      
      <!-- Configuration Status -->
      <div v-if="!isConfigured" class="setup-info">
        <h3>⚠️ Configuration Required</h3>
        <p>Please check your environment variables:</p>
        <ul>
          <li>VITE_GOOGLE_CLIENT_ID: {{ CLIENT_ID ? '✅ Set' : '❌ Missing' }}</li>
          <li>VITE_SPREADSHEET_ID: {{ SHEET_ID ? '✅ Set' : '❌ Missing' }}</li>
          <li>VITE_GOOGLE_API_KEY: {{ API_KEY ? '✅ Set' : '❌ Missing (Optional)' }}</li>
        </ul>
        <p v-if="!CLIENT_ID || !SHEET_ID" class="error-text">
          Missing required environment variables. Please add them to your .env file.
        </p>
      </div>

      <!-- Authentication Method Selection -->
      <div v-if="isConfigured && !isAuthenticated" class="auth-section">
        <h3>Choose Authentication Method</h3>
        <div class="auth-buttons">
          <button @click="authenticateWithOAuth" :disabled="loading" class="auth-btn oauth-btn">
            {{ loading && authMethod === 'oauth' ? 'Authenticating...' : 'Sign in with Google' }}
          </button>
          <button v-if="API_KEY" @click="authenticateWithApiKey" :disabled="loading" class="auth-btn api-btn">
            {{ loading && authMethod === 'apikey' ? 'Connecting...' : 'Use API Key' }}
          </button>
        </div>
      </div>

      <!-- Add Product Form -->
      <div v-if="isAuthenticated" class="form-container">
        <div class="auth-status">
          <span class="auth-indicator">✅ Authenticated via {{ currentAuthMethod }}</span>
          <button @click="signOut" class="sign-out-btn">Sign Out</button>
        </div>
        
        <h2>Add New Product</h2>
        <form @submit.prevent="addProduct" class="product-form">
          <div class="form-group">
            <label for="productName">Product Name:</label>
            <input 
              type="text" 
              id="productName" 
              v-model="newProduct.name" 
              required 
              placeholder="Enter product name"
            />
          </div>
          
          <div class="form-group">
            <label for="price">Price:</label>
            <input 
              type="number" 
              id="price" 
              v-model="newProduct.price" 
              step="0.01" 
              required 
              placeholder="0.00"
            />
          </div>
          
          <div class="form-group">
            <label for="currency">Currency:</label>
            <select id="currency" v-model="newProduct.currency" required>
              <option value="€">EUR (€)</option>
              <option value="Ft">HUF (Ft)</option>
              <option value="$">USD ($)</option>
            </select>
          </div>
          
          <div class="form-group">
            <label for="shop">Shop:</label>
            <select id="shop" v-model="newProduct.shop" required>
              <option value="">Select a shop</option>
              <option value="TESCO SK">TESCO SK</option>
              <option value="LIDL SK">LIDL SK</option>
              <option value="BILLA SK">BILLA SK</option>
              <option value="KAUFLAND">KAUFLAND</option>
              <option value="TERNO">TERNO</option>
              <option value="JEDNOTA">JEDNOTA</option>
              <option value="TESCO HU">TESCO HU</option>
              <option value="TESCO HU €">TESCO HU €</option>
            </select>
          </div>
          
          <div class="form-group">
            <label for="date">Date:</label>
            <input 
              type="date" 
              id="date" 
              v-model="newProduct.date" 
              required 
            />
          </div>
          
          <button type="submit" :disabled="loading" class="submit-btn">
            {{ loading ? 'Adding...' : 'Add Product' }}
          </button>
        </form>
      </div>
      
      <!-- Status Messages -->
      <div v-if="message" :class="['message', messageClass]">
        {{ message }}
      </div>
      
      <!-- Recent Products -->
      <div v-if="recentProducts.length > 0" class="recent-products">
        <h3>Recently Added Products</h3>
        <div class="products-list">
          <div 
            v-for="product in recentProducts" 
            :key="product.id" 
            class="product-item"
          >
            <span class="product-name">{{ product.name }}</span>
            <span class="product-price">{{ product.price }}{{ product.currency }}</span>
            <span class="product-shop">{{ product.shop }}</span>
            <span class="product-date">{{ product.date }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'

// Environment variables
const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID
const SHEET_ID = import.meta.env.VITE_SPREADSHEET_ID
const API_KEY = import.meta.env.GOOGLE_API_KEY
const SHEET_NAME = 'Sheet1'
const CURRENT_ROW = 17

console.log('Environment variables loaded:', {
  CLIENT_ID: CLIENT_ID ? 'Set' : 'Missing',
  SHEET_ID: SHEET_ID ? 'Set' : 'Missing', 
  API_KEY: API_KEY ? 'Set' : 'Missing'
})

// Reactive state
const loading = ref(false)
const message = ref('')
const messageClass = ref('')
const isConfigured = ref(false)
const isAuthenticated = ref(false)
const authMethod = ref('')
const currentAuthMethod = ref('')
const accessToken = ref('')

const newProduct = reactive({
  name: '',
  price: '',
  currency: '€',
  shop: '',
  date: new Date().toISOString().split('T')[0]
})

const recentProducts = ref([])

// Utility functions
const showMessage = (text, type) => {
  message.value = text
  messageClass.value = type
  setTimeout(() => {
    message.value = ''
    messageClass.value = ''
  }, 5000)
}

const resetForm = () => {
  Object.assign(newProduct, {
    name: '',
    price: '',
    currency: '€',
    shop: '',
    date: new Date().toISOString().split('T')[0]
  })
}

// Configuration check
const checkConfiguration = () => {
  isConfigured.value = !!(CLIENT_ID && SHEET_ID)
  
  if (!isConfigured.value) {
    if (!CLIENT_ID) {
      showMessage('GOOGLE_CLIENT_ID missing in .env file', 'error')
    }
    if (!SHEET_ID) {
      showMessage('SPREADSHEET_ID missing in .env file', 'error')
    }
  }
}

// Authentication methods
const authenticateWithOAuth = async () => {
  if (!CLIENT_ID) {
    showMessage('Google Client ID not configured', 'error')
    return
  }

  loading.value = true
  authMethod.value = 'oauth'
  
  try {
    if (typeof google === 'undefined') {
      throw new Error('Google API not loaded. Please refresh the page.')
    }

    const token = await new Promise((resolve, reject) => {
      const client = google.accounts.oauth2.initTokenClient({
        client_id: CLIENT_ID,
        scope: 'https://www.googleapis.com/auth/spreadsheets',
        callback: (tokenResponse) => {
          if (tokenResponse.error) {
            reject(new Error(tokenResponse.error))
          } else if (tokenResponse.access_token) {
            resolve(tokenResponse.access_token)
          } else {
            reject(new Error('No access token received'))
          }
        },
      })
      client.requestAccessToken()
    })

    accessToken.value = token
    isAuthenticated.value = true
    currentAuthMethod.value = 'OAuth'
    showMessage('Successfully authenticated with Google!', 'success')
    
  } catch (error) {
    console.error('OAuth authentication failed:', error)
    showMessage(`Authentication failed: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

const authenticateWithApiKey = async () => {
  if (!API_KEY) {
    showMessage('Google API Key not configured', 'error')
    return
  }

  loading.value = true
  authMethod.value = 'apikey'
  
  try {
    // Test the API key by making a simple request
    const testUrl = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}?key=${API_KEY}`
    const response = await fetch(testUrl)
    
    if (!response.ok) {
      throw new Error('Invalid API key or sheet not accessible')
    }
    
    accessToken.value = API_KEY
    isAuthenticated.value = true
    currentAuthMethod.value = 'API Key'
    showMessage('Successfully connected with API Key!', 'success')
    
  } catch (error) {
    console.error('API Key authentication failed:', error)
    showMessage(`API Key authentication failed: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

const signOut = () => {
  accessToken.value = ''
  isAuthenticated.value = false
  currentAuthMethod.value = ''
  authMethod.value = ''
  showMessage('Signed out successfully', 'success')
}

// Form validation
const validateForm = () => {
  if (!newProduct.name.trim()) {
    showMessage('Product name is required', 'error')
    return false
  }
  
  if (!newProduct.price || newProduct.price <= 0) {
    showMessage('Please enter a valid price', 'error')
    return false
  }
  
  if (!newProduct.shop) {
    showMessage('Please select a shop', 'error')
    return false
  }
  
  return true
}

// Google Sheets integration
const appendToSheet = async () => {
  const values = [
    [
      newProduct.name,
      `${newProduct.price}${newProduct.currency}`,
      newProduct.shop,
      newProduct.date,
      new Date().toISOString()
    ]
  ]

  let url, headers

  if (currentAuthMethod.value === 'OAuth') {
    url = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${SHEET_NAME}!A${CURRENT_ROW}:E${CURRENT_ROW}:append?valueInputOption=USER_ENTERED`
    headers = {
      'Authorization': `Bearer ${accessToken.value}`,
      'Content-Type': 'application/json',
    }
    CURRENT_ROW=CURRENT_ROW + 1
  } else {
    url = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${SHEET_NAME}!A${CURRENT_ROW}:E${CURRENT_ROW}:append?valueInputOption=USER_ENTERED`
    headers = {
      'Content-Type': 'application/json',
    }
    CURRENT_ROW = CURRENT_ROW + 1
  }

  const response = await fetch(url, {
    method: 'POST',
    headers,
    body: JSON.stringify({ values }),
  })

  if (!response.ok) {
    const error = await response.json()
    throw new Error(error.error?.message || 'Failed to add product to sheet')
  }

  return true
}

// Main product addition logic
const addProduct = async () => {
  if (!isAuthenticated.value) {
    showMessage('Please authenticate first', 'error')
    return
  }
  
  if (!validateForm()) return
  
  loading.value = true
  message.value = ''
  
  try {
    await appendToSheet()
    
    // Add to recent products
    const product = {
      id: Date.now(),
      name: newProduct.name,
      price: parseFloat(newProduct.price),
      currency: newProduct.currency,
      shop: newProduct.shop,
      date: newProduct.date
    }
    
    recentProducts.value.unshift(product)
    
    // Keep only last 5 products
    if (recentProducts.value.length > 5) {
      recentProducts.value = recentProducts.value.slice(0, 5)
    }
    
    // Save to localStorage
    localStorage.setItem('recentProducts', JSON.stringify(recentProducts.value))
    
    // Reset form
    resetForm()
    
    showMessage('Product added successfully!', 'success')
    
  } catch (error) {
    console.error('Error adding product:', error)
    showMessage(`Error adding product: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

// Load recent products from localStorage
const loadRecentProducts = () => {
  const savedProducts = localStorage.getItem('recentProducts')
  if (savedProducts) {
    recentProducts.value = JSON.parse(savedProducts)
  }
}

// Initialize on mount
onMounted(() => {
  checkConfiguration()
  loadRecentProducts()
})
</script>
<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

h1 {
  text-align: center;
  color: #2c3e50;
  margin-bottom: 30px;
  font-size: 2.5rem;
}

h2, h3 {
  color: #34495e;
  margin-bottom: 20px;
}

/* Configuration Status */
.setup-info {
  background: #fff3cd;
  border: 2px solid #ffeaa7;
  border-radius: 12px;
  padding: 25px;
  margin-bottom: 30px;
}

.setup-info ul {
  list-style: none;
  padding: 0;
}

.setup-info li {
  padding: 8px 0;
  font-family: monospace;
  font-size: 14px;
}

.error-text {
  color: #dc3545;
  font-weight: 600;
  margin-top: 15px;
}

/* Authentication Section */
.auth-section {
  background: #ffffff;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
  text-align: center;
}

.auth-buttons {
  display: flex;
  gap: 15px;
  justify-content: center;
  flex-wrap: wrap;
}

.auth-btn {
  padding: 15px 30px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 200px;
}

.oauth-btn {
  background: linear-gradient(135deg, #4285f4, #34a853);
  color: white;
}

.oauth-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #3367d6, #2d8f47);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(66, 133, 244, 0.3);
}

.api-btn {
  background: linear-gradient(135deg, #ff6b6b, #ee5a24);
  color: white;
}

.api-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #ee5a24, #d63031);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(255, 107, 107, 0.3);
}

.auth-btn:disabled {
  background: #6c757d;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* Authentication Status */
.auth-status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding: 15px;
  background: #d1edff;
  border-radius: 8px;
  border: 1px solid #bee5eb;
}

.auth-indicator {
  color: #0c5460;
  font-weight: 600;
}

.sign-out-btn {
  padding: 8px 16px;
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.3s ease;
}

.sign-out-btn:hover {
  background: #c82333;
}

/* Form Containers */
.form-container {
  background: #ffffff;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
  border: 1px solid #e9ecef;
}

/* Form Styles */
.product-form {
  display: grid;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
}

label {
  margin-bottom: 8px;
  font-weight: 600;
  color: #495057;
  font-size: 0.95rem;
}

input, select {
  padding: 14px 16px;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  font-size: 16px;
  transition: all 0.3s ease;
  background-color: #ffffff;
}

input:focus, select:focus {
  outline: none;
  border-color: #007bff;
  box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.1);
}

input:hover, select:hover {
  border-color: #ced4da;
}

/* Submit Button */
.submit-btn {
  padding: 14px 28px;
  background: linear-gradient(135deg, #007bff, #0056b3);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.submit-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #0056b3, #004085);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 123, 255, 0.3);
}

.submit-btn:disabled {
  background: #6c757d;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* Messages */
.message {
  padding: 16px 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  text-align: center;
  font-weight: 600;
  font-size: 15px;
}

.success {
  background: #d1edff;
  color: #0c5460;
  border: 1px solid #bee5eb;
}

.error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

/* Recent Products */
.recent-products {
  background: #ffffff;
  padding: 25px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  margin-bottom: 20px;
  border: 1px solid #e9ecef;
}

.products-list {
  display: grid;
  gap: 12px;
}

.product-item {
  display: grid;
  grid-template-columns: 2fr 1fr 1.2fr 1fr;
  gap: 15px;
  padding: 16px 20px;
  background: #f8f9fa;
  border-radius: 8px;
  align-items: center;
  transition: all 0.2s ease;
  border: 1px solid #e9ecef;
}

.product-item:hover {
  background: #e9ecef;
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.product-name {
  font-weight: 600;
  color: #2c3e50;
}

.product-price {
  color: #007bff;
  font-weight: 700;
  font-size: 1.1rem;
}

.product-shop {
  color: #6c757d;
  font-size: 14px;
  font-weight: 500;
}

.product-date {
  color: #adb5bd;
  font-size: 13px;
}

/* Responsive Design */
@media (max-width: 768px) {
  .container {
    padding: 15px;
  }
  
  .form-container, .setup-info, .auth-section {
    padding: 20px;
  }
  
  .auth-buttons {
    flex-direction: column;
    align-items: center;
  }
  
  .auth-btn {
    min-width: 250px;
  }
  
  .auth-status {
    flex-direction: column;
    gap: 10px;
    text-align: center;
  }
  
  .product-item {
    grid-template-columns: 1fr;
    gap: 8px;
    text-align: center;
  }
  
  .product-item span {
    padding: 4px 0;
  }
  
  h1 {
    font-size: 2rem;
  }
}

@media (max-width: 480px) {
  .container {
    padding: 10px;
  }
  
  .form-container, .setup-info, .auth-section {
    padding: 15px;
  }
  
  input, select, .submit-btn, .auth-btn {
    font-size: 14px;
  }
  
  h1 {
    font-size: 1.8rem;
  }
}
</style>