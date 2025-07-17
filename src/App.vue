<template>
  <div id="app">
    <div class="container">
      <h1>Product Price Tracker</h1>
      
      <!-- Configuration Section -->
      <div v-if="!isConfigured" class="setup-info">
        <h3>⚠️ Setup Required</h3>
        <p>Configure your Google Sheets API access to start tracking products:</p>
        
        <div class="config-form">
          <div class="form-group">
            <label for="apiKey">Google Sheets API Key:</label>
            <input 
              type="password" 
              id="apiKey" 
              v-model="config.apiKey" 
              placeholder="Enter your API key"
            />
          </div>
          
          <div class="form-group">
            <label for="sheetId">Google Sheet ID:</label>
            <input 
              type="text" 
              id="sheetId" 
              v-model="config.sheetId" 
              placeholder="Sheet ID from URL"
            />
          </div>
          
          <div class="form-group">
            <label for="sheetName">Sheet Name:</label>
            <input 
              type="text" 
              id="sheetName" 
              v-model="config.sheetName" 
              placeholder="Sheet1"
            />
          </div>
          
          <button @click="saveConfig" class="config-btn">
            Save Configuration
          </button>
        </div>
      </div>

      <!-- Add Product Form -->
      <div v-if="isConfigured" class="form-container">
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

      <!-- Reset Configuration -->
      <div v-if="isConfigured" class="reset-section">
        <button @click="resetConfig" class="reset-btn">
          Reset Configuration
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'

// Environment variables
const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID

// Reactive state
const loading = ref(false)
const message = ref('')
const messageClass = ref('')
const isConfigured = ref(false)

const newProduct = reactive({
  name: '',
  price: '',
  currency: '€',
  shop: '',
  date: new Date().toISOString().split('T')[0]
})

const config = reactive({
  apiKey: '',
  sheetId: '',
  sheetName: 'Sheet1'
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

// Configuration management
const loadConfig = () => {
  const savedConfig = localStorage.getItem('googleSheetsConfig')
  if (savedConfig) {
    Object.assign(config, JSON.parse(savedConfig))
    isConfigured.value = !!(config.apiKey && config.sheetId)
  }
  
  const savedProducts = localStorage.getItem('recentProducts')
  if (savedProducts) {
    recentProducts.value = JSON.parse(savedProducts)
  }
}

const saveConfig = () => {
  if (!config.apiKey || !config.sheetId) {
    showMessage('Please provide both API key and Sheet ID', 'error')
    return
  }
  
  localStorage.setItem('googleSheetsConfig', JSON.stringify(config))
  isConfigured.value = true
  showMessage('Configuration saved successfully!', 'success')
}

const resetConfig = () => {
  if (confirm('Are you sure you want to reset the configuration?')) {
    localStorage.removeItem('googleSheetsConfig')
    Object.assign(config, { apiKey: '', sheetId: '', sheetName: 'Sheet1' })
    isConfigured.value = false
    showMessage('Configuration reset successfully', 'success')
  }
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
const getAccessToken = async () => {
  if (!CLIENT_ID) {
    throw new Error('Google Client ID not configured in environment variables')
  }
  
  return new Promise((resolve, reject) => {
    if (typeof google === 'undefined') {
      reject(new Error('Google API not loaded'))
      return
    }
    
    const client = google.accounts.oauth2.initTokenClient({
      client_id: CLIENT_ID,
      scope: 'https://www.googleapis.com/auth/spreadsheets',
      callback: (tokenResponse) => {
        if (tokenResponse.error) {
          reject(new Error(tokenResponse.error))
        } else {
          resolve(tokenResponse.access_token)
        }
      },
    })

    client.requestAccessToken()
  })
}

const appendToSheet = async () => {
  const url = `https://sheets.googleapis.com/v4/spreadsheets/${config.sheetId}/values/${config.sheetName}!A:E:append?valueInputOption=RAW`

  const values = [
    [
      newProduct.name,
      `${newProduct.price}${newProduct.currency}`,
      newProduct.shop,
      newProduct.date,
      new Date().toISOString()
    ]
  ]

  try {
    const accessToken = await getAccessToken()

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ values }),
    })

    if (!response.ok) {
      const error = await response.json()
      throw new Error(error.error?.message || 'Failed to add product to sheet')
    }

    return true
  } catch (err) {
    console.error('Error appending to sheet:', err)
    throw err
  }
}

// Main product addition logic
const addProduct = async () => {
  if (!isConfigured.value) {
    showMessage('Please configure Google Sheets API first', 'error')
    return
  }
  
  if (!validateForm()) return
  
  loading.value = true
  message.value = ''
  
  try {
    const success = await appendToSheet()
    
    if (success) {
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
    }
    
  } catch (error) {
    console.error('Error adding product:', error)
    showMessage(`Error adding product: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

// Initialize on mount
onMounted(() => {
  loadConfig()
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

h2 {
  color: #34495e;
  margin-bottom: 20px;
}

h3 {
  color: #e67e22;
  margin-top: 0;
}

/* Form Containers */
.form-container, .setup-info {
  background: #ffffff;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
  border: 1px solid #e9ecef;
}

.setup-info {
  background: #fff8e1;
  border-color: #ffcc02;
}

.config-form {
  background: #ffffff;
  padding: 25px;
  border-radius: 8px;
  margin-top: 20px;
  border: 1px solid #dee2e6;
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

/* Buttons */
.submit-btn, .config-btn, .reset-btn {
  padding: 14px 28px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.submit-btn {
  background: linear-gradient(135deg, #007bff, #0056b3);
  color: white;
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

.config-btn {
  background: linear-gradient(135deg, #28a745, #1e7e34);
  color: white;
}

.config-btn:hover {
  background: linear-gradient(135deg, #1e7e34, #155724);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(40, 167, 69, 0.3);
}

.reset-btn {
  background: linear-gradient(135deg, #dc3545, #c82333);
  color: white;
  font-size: 14px;
  padding: 10px 20px;
}

.reset-btn:hover {
  background: linear-gradient(135deg, #c82333, #a71e2a);
  transform: translateY(-1px);
  box-shadow: 0 3px 8px rgba(220, 53, 69, 0.3);
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

/* Reset Section */
.reset-section {
  text-align: center;
  margin-top: 30px;
  padding-top: 20px;
  border-top: 1px solid #e9ecef;
}

/* Responsive Design */
@media (max-width: 768px) {
  .container {
    padding: 15px;
  }
  
  .form-container, .setup-info {
    padding: 20px;
  }
  
  .config-form {
    padding: 20px;
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
  
  .form-container, .setup-info {
    padding: 15px;
  }
  
  input, select, .submit-btn, .config-btn {
    font-size: 14px;
  }
  
  h1 {
    font-size: 1.8rem;
  }
}
</style>