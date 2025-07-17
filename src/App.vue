<script setup>
const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;
const CLIENT_SECRET = import.meta.env.VITE_GOOGLE_CLIENT_SECRET;
</script>

<template>
  <div id="app">
    <div class="container">
      <h1>Product Price Tracker</h1>
      
      <!-- Add Product Form -->
      <div class="form-container">
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
      <div v-if="message" :class="messageClass" class="message">
        {{ message }}
      </div>
      
      <!-- Recent Products -->
      <div class="recent-products" v-if="recentProducts.length > 0">
        <h3>Recently Added Products</h3>
        <div class="products-list">
          <div v-for="product in recentProducts" :key="product.id" class="product-item">
            <span class="product-name">{{ product.name }}</span>
            <span class="product-price">{{ product.price }}{{ product.currency }}</span>
            <span class="product-shop">{{ product.shop }}</span>
            <span class="product-date">{{ product.date }}</span>
          </div>
        </div>
      </div>
      
      <!-- Setup Instructions -->
      <div class="setup-info" v-if="!isConfigured">
        <h3>⚠️ Setup Required</h3>
        <p>To use this app, you need to configure Google Sheets API access:</p>
        <ol>
          <li>Create a Google Cloud Project and enable the Google Sheets API</li>
          <li>Create an API Key (restrict to Google Sheets API)</li>
          <li>Make your Google Sheet public or shareable with link</li>
          <li>Add your API key and Sheet ID to the configuration below</li>
        </ol>
        
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
          
          <button @click="saveConfig" class="config-btn">Save Configuration</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ProductTracker',
  data() {
    return {
      newProduct: {
        name: '',
        price: '',
        currency: '€',
        shop: '',
        date: new Date().toISOString().split('T')[0]
      },
      recentProducts: [],
      loading: false,
      message: '',
      messageClass: '',
      config: {
        apiKey: '',
        sheetId: '',
        sheetName: 'Sheet1'
      },
      isConfigured: false
    }
  },
  
  mounted() {
    this.loadConfig();
  },
  
  methods: {
    loadConfig() {
      const savedConfig = localStorage.getItem('googleSheetsConfig');
      if (savedConfig) {
        this.config = JSON.parse(savedConfig);
        this.isConfigured = !!(this.config.apiKey && this.config.sheetId);
      }
      
      // Load recent products from localStorage
      const savedProducts = localStorage.getItem('recentProducts');
      if (savedProducts) {
        this.recentProducts = JSON.parse(savedProducts);
      }
    },
    
    saveConfig() {
      if (!this.config.apiKey || !this.config.sheetId) {
        this.showMessage('Please provide both API key and Sheet ID', 'error');
        return;
      }
      
      localStorage.setItem('googleSheetsConfig', JSON.stringify(this.config));
      this.isConfigured = true;
      this.showMessage('Configuration saved successfully!', 'success');
    },
    
    async addProduct() {
      if (!this.isConfigured) {
        this.showMessage('Please configure Google Sheets API first', 'error');
        return;
      }
      
      if (!this.validateForm()) return;
      
      this.loading = true;
      this.message = '';
      
      try {
        const success = await this.appendToSheet();
        
        if (success) {
          // Add to recent products
          const product = {
            id: Date.now(),
            name: this.newProduct.name,
            price: parseFloat(this.newProduct.price),
            currency: this.newProduct.currency,
            shop: this.newProduct.shop,
            date: this.newProduct.date
          };
          
          this.recentProducts.unshift(product);
          
          // Keep only last 5 products
          if (this.recentProducts.length > 5) {
            this.recentProducts = this.recentProducts.slice(0, 5);
          }
          
          // Save to localStorage
          localStorage.setItem('recentProducts', JSON.stringify(this.recentProducts));
          
          // Reset form
          this.newProduct = {
            name: '',
            price: '',
            currency: '€',
            shop: '',
            date: new Date().toISOString().split('T')[0]
          };
          
          this.showMessage('Product added successfully!', 'success');
        }
        
      } catch (error) {
        console.error('Error adding product:', error);
        this.showMessage('Error adding product. Please check your configuration.', 'error');
      } finally {
        this.loading = false;
      }
    },
    
    async getAccessToken() {
      const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;
      
      if (!CLIENT_ID) {
        throw new Error('Google Client ID not configured');
      }
      
      return new Promise((resolve, reject) => {
        const client = google.accounts.oauth2.initTokenClient({
          client_id: CLIENT_ID,
          scope: 'https://www.googleapis.com/auth/spreadsheets',
          callback: (tokenResponse) => {
            if (tokenResponse.error) {
              reject(tokenResponse);
            } else {
              resolve(tokenResponse.access_token);
            }
          },
        });

        client.requestAccessToken();
      });
    },

    async appendToSheet() {
      const url = `https://sheets.googleapis.com/v4/spreadsheets/${this.config.sheetId}/values/${this.config.sheetName}!A:E:append?valueInputOption=RAW`;

      const values = [
        [
          this.newProduct.name,
          `${this.newProduct.price}${this.newProduct.currency}`,
          this.newProduct.shop,
          this.newProduct.date,
          new Date().toISOString()
        ]
      ];

      try {
        const accessToken = await this.getAccessToken(); // Get OAuth token

        const response = await fetch(url, {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${accessToken}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ values }),
        });

        if (!response.ok) {
          const error = await response.json();
          throw new Error(error.error?.message || 'Failed to add product to sheet');
        }

        return true;
      } catch (err) {
        console.error('Error appending to sheet:', err);
        throw err;
      }
    },
    
    validateForm() {
      if (!this.newProduct.name.trim()) {
        this.showMessage('Product name is required', 'error');
        return false;
      }
      
      if (!this.newProduct.price || this.newProduct.price <= 0) {
        this.showMessage('Please enter a valid price', 'error');
        return false;
      }
      
      if (!this.newProduct.shop) {
        this.showMessage('Please select a shop', 'error');
        return false;
      }
      
      return true;
    },
    
    showMessage(text, type) {
      this.message = text;
      this.messageClass = type;
      setTimeout(() => {
        this.message = '';
        this.messageClass = '';
      }, 5000);
    }
  }
}
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
}

.form-container {
  background: #f9f9f9;
  padding: 30px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  margin-bottom: 30px;
}

.product-form {
  display: grid;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
}

label {
  margin-bottom: 5px;
  font-weight: bold;
  color: #555;
}

input, select {
  padding: 12px;
  border: 2px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
  transition: border-color 0.3s;
}

input:focus, select:focus {
  outline: none;
  border-color: #007bff;
}

.submit-btn, .config-btn {
  padding: 15px 30px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s;
}

.submit-btn:hover:not(:disabled), .config-btn:hover {
  background: #0056b3;
}

.submit-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.message {
  padding: 15px;
  border-radius: 4px;
  margin-bottom: 20px;
  text-align: center;
  font-weight: bold;
}

.success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.recent-products {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  margin-bottom: 20px;
}

.recent-products h3 {
  margin-top: 0;
  color: #333;
}

.products-list {
  display: grid;
  gap: 10px;
}

.product-item {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr;
  gap: 15px;
  padding: 15px;
  background: #f8f9fa;
  border-radius: 4px;
  align-items: center;
}

.product-name {
  font-weight: bold;
  color: #333;
}

.product-price {
  color: #007bff;
  font-weight: bold;
}

.product-shop {
  color: #666;
  font-size: 14px;
}

.product-date {
  color: #999;
  font-size: 12px;
}

.setup-info {
  background: #fff3cd;
  border: 1px solid #ffeaa7;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
}

.setup-info h3 {
  color: #856404;
  margin-top: 0;
}

.setup-info p {
  color: #856404;
  margin-bottom: 15px;
}

.setup-info ol {
  color: #856404;
  padding-left: 20px;
}

.config-form {
  background: white;
  padding: 20px;
  border-radius: 4px;
  margin-top: 20px;
}

.config-form .form-group {
  margin-bottom: 15px;
}

.config-btn {
  background: #28a745;
  margin-top: 10px;
}

.config-btn:hover {
  background: #218838;
}

@media (max-width: 768px) {
  .container {
    padding: 10px;
  }
  
  .form-container {
    padding: 20px;
  }
  
  .product-item {
    grid-template-columns: 1fr;
    gap: 5px;
  }
  
  .product-item span {
    text-align: left;
  }
}
</style>