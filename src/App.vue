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

      <!-- Product Price Table -->
      <div v-if="isAuthenticated" class="form-container">
        <div class="auth-status">
          <span class="auth-indicator">✅ Authenticated via {{ currentAuthMethod }}</span>
          <button @click="signOut" class="sign-out-btn">Sign Out</button>
          <button @click="refreshFromSheet" :disabled="loading" class="refresh-btn">
            {{ loading ? 'Loading...' : 'Refresh from Sheet' }}
          </button>
        </div>
        
        <h2>Product Price Comparison Table</h2>
        
        <!-- Add New Product Row -->
        <div class="add-product-section">
          <div class="form-group">
            <label for="productName">New Product Name:</label>
            <input 
              type="text" 
              id="productName" 
              v-model="newProduct.name" 
              placeholder="Enter product name"
              @keyup.enter="addNewProductRow"
            />
            <button @click="addNewProductRow" :disabled="loading || !newProduct.name" class="add-row-btn">
              Add New Product Row
            </button>
          </div>
        </div>
        
        <!-- Price Table -->
        <div class="price-table-container">
          <table class="price-table">
            <thead>
              <tr>
                <th>Product</th>
                <th v-for="shop in shops" :key="shop.id" :class="shop.id">{{ shop.name }}</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(product, index) in productRows" :key="product.id" :class="{ 'unsaved': product.unsavedChanges }">
                <td class="product-name-cell">
                  <input 
                    v-model="product.name" 
                    @change="markAsUnsaved(product)"
                    class="product-name-input"
                  />
                </td>
                <td v-for="shop in shops" :key="`${product.id}-${shop.id}`">
                  <div class="price-input-container">
                    <input 
                      type="number" 
                      v-model="product.prices[shop.id]" 
                      step="0.01" 
                      placeholder="0.00"
                      @change="markAsUnsaved(product)"
                      class="price-input"
                    />
                    <span class="currency-indicator">{{ shop.currency }}</span>
                  </div>
                </td>
                <td>
                  <button @click="removeProductRow(index)" class="remove-btn" title="Remove product">
                    <span>×</span>
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        
        <!-- Bulk Actions -->
        <div class="bulk-actions">
          <button @click="saveAllPrices" :disabled="loading || !hasUnsavedChanges" class="submit-btn">
            {{ loading ? 'Saving...' : `Save Changes (${unsavedCount})` }}
          </button>
          <button @click="discardChanges" :disabled="loading || !hasUnsavedChanges" class="discard-btn">
            Discard Changes
          </button>
        </div>
      </div>
      
      <!-- Status Messages -->
      <div v-if="message" :class="['message', messageClass]">
        {{ message }}
      </div>
      
      <!-- Debug Info (only in development) -->
      <div v-if="showDebugInfo" class="debug-info">
        <h3>Debug Information</h3>
        <pre>{{ debugInfo }}</pre>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed, nextTick } from 'vue'

// Environment variables
const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID
const SHEET_ID = import.meta.env.VITE_SPREADSHEET_ID
const API_KEY = import.meta.env.VITE_GOOGLE_API_KEY
const SHEET_NAME = import.meta.env.VITE_SHEET_NAME || 'Sheet1'
const DISCOVERY_DOC = 'https://sheets.googleapis.com/$discovery/rest?version=v4'
const SCOPES = 'https://www.googleapis.com/auth/spreadsheets'

// Shop definitions
const shops = [
  { id: 'tesco_sk', name: 'TESCO SK', currency: '€' },
  { id: 'lidl_sk', name: 'LIDL SK', currency: '€' },
  { id: 'billa_sk', name: 'BILLA SK', currency: '€' },
  { id: 'kaufland', name: 'KAUFLAND', currency: '€' },
  { id: 'terno', name: 'TERNO', currency: '€' },
  { id: 'jednota', name: 'JEDNOTA', currency: '€' },
  { id: 'tesco_hu', name: 'TESCO HU', currency: 'Ft' },
  { id: 'tesco_hu_eur', name: 'TESCO HU €', currency: '€' }
]

// Initialize with empty array - will be populated from sheet
const productRows = ref([])

console.log('Environment variables loaded:', {
  CLIENT_ID: CLIENT_ID ? 'Set' : 'Missing',
  SHEET_ID: SHEET_ID ? 'Set' : 'Missing', 
  API_KEY: API_KEY ? 'Set' : 'Missing',
  SHEET_NAME
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
const gapi = ref(null)
const googleAuth = ref(null)
const showDebugInfo = ref(import.meta.env.DEV)

const newProduct = reactive({
  name: ''
})

// Computed properties
const hasUnsavedChanges = computed(() => {
  return productRows.value.some(product => product.unsavedChanges)
})

const unsavedCount = computed(() => {
  return productRows.value.filter(product => product.unsavedChanges).length
})

const debugInfo = computed(() => {
  return {
    isAuthenticated: isAuthenticated.value,
    authMethod: authMethod.value,
    productCount: productRows.value.length,
    unsavedChanges: unsavedCount.value,
    hasToken: !!accessToken.value
  }
})

// Utility functions
const showMessage = (text, type = 'info') => {
  message.value = text
  messageClass.value = type
  console.log(`[${type.toUpperCase()}] ${text}`)
  setTimeout(() => {
    message.value = ''
    messageClass.value = ''
  }, 5000)
}

const resetForm = () => {
  Object.assign(newProduct, {
    name: ''
  })
}

// Configuration check
const checkConfiguration = () => {
  isConfigured.value = !!(CLIENT_ID && SHEET_ID)
  
  if (!isConfigured.value) {
    const missing = []
    if (!CLIENT_ID) missing.push('VITE_GOOGLE_CLIENT_ID')
    if (!SHEET_ID) missing.push('VITE_SPREADSHEET_ID')
    
    showMessage(`Missing environment variables: ${missing.join(', ')}`, 'error')
  }
  
  return isConfigured.value
}

// Initialize Google API
const initializeGoogleAPI = async () => {
  try {
    if (typeof gapi === 'undefined') {
      throw new Error('Google API (gapi) not loaded. Please ensure the Google API script is included.')
    }

    await new Promise((resolve, reject) => {
      gapi.load('client:auth2', {
        callback: resolve,
        onerror: reject
      })
    })

    await gapi.client.init({
      apiKey: API_KEY,
      clientId: CLIENT_ID,
      discoveryDocs: [DISCOVERY_DOC],
      scope: SCOPES
    })

    googleAuth.value = gapi.auth2.getAuthInstance()
    console.log('Google API initialized successfully')
    return true
  } catch (error) {
    console.error('Failed to initialize Google API:', error)
    throw error
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
      const error = await response.json()
      throw new Error(error.error?.message || 'Invalid API key or sheet not accessible')
    }
    
    accessToken.value = API_KEY
    isAuthenticated.value = true
    currentAuthMethod.value = 'API Key'
    showMessage('Successfully connected with API Key!', 'success')
    
    // Load existing products from the sheet
    await loadProductsFromSheet()
    
  } catch (error) {
    console.error('API Key authentication failed:', error)
    showMessage(`API Key authentication failed: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

const signOut = async () => {
  try {
    if (authMethod.value === 'oauth' && googleAuth.value) {
      await googleAuth.value.signOut()
    }
    
    accessToken.value = ''
    isAuthenticated.value = false
    currentAuthMethod.value = ''
    authMethod.value = ''
    productRows.value = []
    
    showMessage('Signed out successfully', 'success')
  } catch (error) {
    console.error('Sign out error:', error)
    showMessage('Signed out with errors', 'warning')
  }
}

// Product management functions
const markAsUnsaved = (product) => {
  product.unsavedChanges = true
}

const addNewProductRow = () => {
  if (!newProduct.name.trim()) {
    showMessage('Please enter a product name', 'error')
    return
  }
  
  const productName = newProduct.name.trim()
  
  // Check if product already exists
  const existingProduct = productRows.value.find(p => 
    p.name.toLowerCase() === productName.toLowerCase()
  )
  
  if (existingProduct) {
    showMessage('Product already exists in the table', 'warning')
    return
  }
  
  const newId = productRows.value.length > 0 
    ? Math.max(...productRows.value.map(p => p.id)) + 1 
    : 1
    
  productRows.value.push({
    id: newId,
    name: productName,
    prices: {},
    unsavedChanges: true,
    isNew: true
  })
  
  resetForm()
  showMessage(`Added new product: ${productName}`, 'success')
}

const removeProductRow = (index) => {
  const product = productRows.value[index]
  if (confirm(`Are you sure you want to remove "${product.name}"?`)) {
    productRows.value.splice(index, 1)
    showMessage('Product removed', 'success')
  }
}

const discardChanges = () => {
  if (confirm('Are you sure you want to discard all unsaved changes?')) {
    loadProductsFromSheet()
    showMessage('Changes discarded', 'info')
  }
}

const refreshFromSheet = () => {
  loadProductsFromSheet()
}

// Google Sheets integration
const buildSheetUrl = (range = '', method = 'GET') => {
  const baseUrl = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${SHEET_NAME}`
  
  if (authMethod.value === 'oauth') {
    return range ? `${baseUrl}!${range}` : `${baseUrl}!A:Z`
  } else {
    const url = range ? `${baseUrl}!${range}` : `${baseUrl}!A:Z`
    return `${url}?key=${API_KEY}`
  }
}

const buildRequestHeaders = () => {
  if (authMethod.value === 'oauth') {
    return {
      'Authorization': `Bearer ${accessToken.value}`,
      'Content-Type': 'application/json',
    }
  } else {
    return {
      'Content-Type': 'application/json',
    }
  }
}

const parsePrice = (cellValue, currency) => {
  if (!cellValue || cellValue.trim() === '') return ''
  
  // Remove currency symbols and extra spaces
  let cleanValue = cellValue.toString()
    .replace(/€/g, '')
    .replace(/Ft/g, '')
    .replace(/\s+/g, '')
    .trim()
  
  // Handle comma as decimal separator
  if (cleanValue.includes(',') && !cleanValue.includes('.')) {
    cleanValue = cleanValue.replace(',', '.')
  }
  
  // Extract numeric value
  const match = cleanValue.match(/[\d.]+/)
  if (match) {
    const numericValue = parseFloat(match[0])
    return isNaN(numericValue) ? '' : numericValue.toString()
  }
  
  return ''
}

const formatPrice = (price, currency) => {
  if (!price || price === '') return ''
  
  const numericPrice = parseFloat(price)
  if (isNaN(numericPrice)) return ''
  
  if (currency === 'Ft') {
    return `${numericPrice.toFixed(2).replace('.', ',')} Ft`
  } else {
    return `${numericPrice.toFixed(2).replace('.', ',')}€`
  }
}

const loadProductsFromSheet = async () => {
  if (!isAuthenticated.value) {
    showMessage('Please authenticate first', 'error')
    return
  }
  
  loading.value = true
  
  try {
    let url, headers
    
    // Build URL with properly encoded sheet name
    const encodedSheetName = encodeURIComponent(SHEET_NAME)
    if (authMethod.value === 'oauth') {
      url = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${encodedSheetName}`
      headers = {
        'Authorization': `Bearer ${accessToken.value}`,
        'Content-Type': 'application/json',
      }
    } else {
      url = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${encodedSheetName}?key=${API_KEY}`
      headers = {
        'Content-Type': 'application/json',
      }
    }
    
    console.log('Loading products from:', url)
    
    const response = await fetch(url, {
      method: 'GET',
      headers
    })
    
    if (!response.ok) {
      const error = await response.json()
      console.error('API Error:', error)
      throw new Error(error.error?.message || `HTTP ${response.status}: Failed to load products from sheet`)
    }
    
    const data = await response.json()
    const values = data.values || []
    
    console.log('Raw sheet data:', values)
    
    if (values.length === 0) {
      productRows.value = []
      showMessage('Sheet is empty', 'info')
      return
    }
    
    // Find header row (look for "Product" or similar)
    let headerRowIndex = -1
    let headerRow = null
    
    for (let i = 0; i < Math.min(5, values.length); i++) {
      const row = values[i]
      if (row && row[0] && (
        row[0].toLowerCase().includes('product') ||
        row[0].toLowerCase().includes('name') ||
        row[0].toLowerCase().includes('item')
      )) {
        headerRowIndex = i
        headerRow = row
        break
      }
    }
    
    // If no header found, assume first row with data is products
    if (headerRowIndex === -1) {
      headerRowIndex = 0
      headerRow = values[0]
    }
    
    console.log('Header row found at index:', headerRowIndex, headerRow)
    
    // Parse products starting from the row after header
    const loadedProducts = []
    const startRow = headerRowIndex + 1
    
    for (let i = startRow; i < values.length; i++) {
      const row = values[i]
      if (!row || !row[0] || row[0].toString().trim() === '') continue
      
      const product = {
        id: Date.now() + i,
        name: row[0].toString().trim(),
        prices: {},
        unsavedChanges: false,
        isNew: false
      }
      
      // Map prices to shops (assuming columns match shop order)
      shops.forEach((shop, shopIndex) => {
        const cellValue = row[shopIndex + 1] // +1 because column 0 is product name
        const price = parsePrice(cellValue, shop.currency)
        if (price) {
          product.prices[shop.id] = price
        }
      })
      
      loadedProducts.push(product)
    }
    
    productRows.value = loadedProducts
    
    if (loadedProducts.length > 0) {
      showMessage(`Loaded ${loadedProducts.length} products from sheet`, 'success')
    } else {
      showMessage('No products found in sheet', 'info')
    }
    
    console.log('Loaded products:', loadedProducts)
    
  } catch (error) {
    console.error('Error loading products from sheet:', error)
    showMessage(`Error loading products: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

const saveAllPrices = async () => {
  if (!isAuthenticated.value) {
    showMessage('Please authenticate first', 'error')
    return
  }
  
  loading.value = true
  
  try {
    const productsToUpdate = productRows.value.filter(p => p.unsavedChanges)
    
    if (productsToUpdate.length === 0) {
      showMessage('No changes to save', 'info')
      return
    }
    
    console.log(`Saving ${productsToUpdate.length} products...`)
    
    // Prepare batch update data
    const sheetData = []
    
    // Add header row
    const headerRow = ['Product', ...shops.map(shop => shop.name)]
    sheetData.push(headerRow)
    
    // Add all products (both changed and unchanged)
    productRows.value.forEach(product => {
      const row = [product.name]
      shops.forEach(shop => {
        const price = product.prices[shop.id]
        row.push(price ? formatPrice(price, shop.currency) : '')
      })
      sheetData.push(row)
    })
    
    console.log('Sheet data to save:', sheetData)
    
    // Clear the sheet and write all data
    if (authMethod.value === 'oauth') {
      // Clear existing data
      const clearUrl = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${SHEET_NAME}!A:Z:clear`
      await fetch(clearUrl, {
        method: 'POST',
        headers: buildRequestHeaders()
      })
      
      // Write new data
      const updateUrl = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${SHEET_NAME}!A1?valueInputOption=USER_ENTERED`
      const response = await fetch(updateUrl, {
        method: 'PUT',
        headers: buildRequestHeaders(),
        body: JSON.stringify({ values: sheetData })
      })
      
      if (!response.ok) {
        const error = await response.json()
        throw new Error(error.error?.message || 'Failed to save to sheet')
      }
    } else {
      // For API key method, we need to handle this differently
      // This is a limitation - API key doesn't allow clearing/updating ranges
      showMessage('Batch update not supported with API key. Please use OAuth method.', 'warning')
      return
    }
    
    // Mark all products as saved
    productRows.value.forEach(product => {
      product.unsavedChanges = false
      product.isNew = false
    })
    
    showMessage(`Successfully saved ${productsToUpdate.length} products!`, 'success')
    
  } catch (error) {
    console.error('Error saving prices:', error)
    showMessage(`Error saving: ${error.message}`, 'error')
  } finally {
    loading.value = false
  }
}

// Initialize on mount
onMounted(async () => {
  console.log('App mounted, checking configuration...')
  
  if (checkConfiguration()) {
    console.log('Configuration valid')
    
    // Load Google API script if not already loaded
    if (typeof gapi === 'undefined') {
      console.log('Loading Google API script...')
      const script = document.createElement('script')
      script.src = 'https://apis.google.com/js/api.js'
      script.onload = () => {
        console.log('Google API script loaded')
      }
      document.head.appendChild(script)
    }
  }
})
</script>


<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

h1 {
  color: #333;
  text-align: center;
  margin-bottom: 30px;
}

.setup-info {
  background: #fff3cd;
  border: 1px solid #ffeaa7;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.setup-info ul {
  margin: 10px 0;
  padding-left: 20px;
}

.error-text {
  color: #721c24;
  font-weight: bold;
}

.auth-section {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  text-align: center;
}

.auth-buttons {
  display: flex;
  gap: 15px;
  justify-content: center;
  margin-top: 15px;
}

.auth-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 180px;
}

.oauth-btn {
  background: #4285f4;
  color: white;
}

.oauth-btn:hover:not(:disabled) {
  background: #3367d6;
}

.api-btn {
  background: #34a853;
  color: white;
}

.api-btn:hover:not(:disabled) {
  background: #2d8f47;
}

.auth-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.form-container {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.auth-status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding: 10px;
  background: #e8f5e8;
  border-radius: 6px;
}

.auth-indicator {
  color: #155724;
  font-weight: bold;
}

.sign-out-btn, .refresh-btn {
  padding: 8px 16px;
  background: #6c757d;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-left: 10px;
}

.sign-out-btn:hover, .refresh-btn:hover {
  background: #545b62;
}

.add-product-section {
  margin-bottom: 20px;
  padding: 15px;
  background: #f8f9fa;
  border-radius: 6px;
}

.form-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.form-group label {
  font-weight: bold;
  min-width: 150px;
}

.form-group input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.add-row-btn {
  padding: 8px 16px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.add-row-btn:hover:not(:disabled) {
  background: #0056b3;
}

.add-row-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.price-table-container {
  overflow-x: auto;
  margin-bottom: 20px;
}

.price-table {
  width: 100%;
  border-collapse: collapse;
  background: white;
}

.price-table th,
.price-table td {
  padding: 12px 8px;
  text-align: left;
  border: 1px solid #ddd;
}

.price-table th {
  background: #f8f9fa;
  font-weight: bold;
  position: sticky;
  top: 0;
  z-index: 10;
}

.price-table tbody tr:hover {
  background: #f5f5f5;
}

.price-table tbody tr.unsaved {
  background: #fff3cd;
}

.product-name-cell {
  min-width: 200px;
  font-weight: bold;
}

.product-name-input {
  width: 100%;
  padding: 4px 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-weight: bold;
}

.price-input-container {
  display: flex;
  align-items: center;
  gap: 5px;
}

.price-input {
  width: 80px;
  padding: 4px 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  text-align: right;
}

.currency-indicator {
  font-size: 12px;
  color: #666;
  min-width: 20px;
}

.remove-btn {
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 50%;
  width: 30px;
  height: 30px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.remove-btn:hover {
  background: #c82333;
}

.bulk-actions {
  display: flex;
  gap: 15px;
  justify-content: center;
  margin-top: 20px;
}

.submit-btn {
  padding: 12px 24px;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #218838;
}

.submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.discard-btn {
  padding: 12px 24px;
  background: #6c757d;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.discard-btn:hover:not(:disabled) {
  background: #545b62;
}

.discard-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.message {
  padding: 15px;
  border-radius: 6px;
  margin: 20px 0;
  font-weight: bold;
}

.message.success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.message.error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.message.warning {
  background: #fff3cd;
  color: #856404;
  border: 1px solid #ffeaa7;
}

.message.info {
  background: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
}

.debug-info {
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  padding: 15px;
  border-radius: 6px;
  margin-top: 20px;
}

.debug-info h3 {
  margin-top: 0;
  color: #6c757d;
}

.debug-info pre {
  background: #e9ecef;
  padding: 10px;
  border-radius: 4px;
  overflow: auto;
  font-size: 12px;
}

/* Responsive design */
@media (max-width: 768px) {
  .container {
    padding: 10px;
  }
  
  .auth-buttons {
    flex-direction: column;
    align-items: center;
  }
  
  .form-group {
    flex-direction: column;
    align-items: stretch;
  }
  
  .form-group label {
    min-width: auto;
    margin-bottom: 5px;
  }
  
  .auth-status {
    flex-direction: column;
    gap: 10px;
    text-align: center;
  }
  
  .bulk-actions {
    flex-direction: column;
  }
  
  .price-table {
    font-size: 14px;
  }
  
  .price-table th,
  .price-table td {
    padding: 8px 4px;
  }
}

/* Shop-specific column styling */
.price-table th.tesco_sk,
.price-table td:nth-child(2) { background-color: #e3f2fd; }

.price-table th.lidl_sk,
.price-table td:nth-child(3) { background-color: #f3e5f5; }

.price-table th.billa_sk,
.price-table td:nth-child(4) { background-color: #e8f5e8; }

.price-table th.kaufland,
.price-table td:nth-child(5) { background-color: #fff3e0; }

.price-table th.terno,
.price-table td:nth-child(6) { background-color: #fce4ec; }

.price-table th.jednota,
.price-table td:nth-child(7) { background-color: #e0f2f1; }

.price-table th.tesco_hu,
.price-table td:nth-child(8) { background-color: #e1f5fe; }

.price-table th.tesco_hu_eur,
.price-table td:nth-child(9) { background-color: #f1f8e9; }
</style>