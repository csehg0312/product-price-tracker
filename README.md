# Product Price Tracker - Vercel Static App

A Vue.js static application for tracking product prices in Google Sheets, optimized for Vercel deployment.

## 🚀 Features

- ✅ **Static App** - No backend server required
- ✅ **Client-side Google Sheets integration** - Direct API calls from browser
- ✅ **Local storage** - Configuration and recent products saved locally
- ✅ **Responsive design** - Works on all devices
- ✅ **Easy deployment** - One-click Vercel deployment

## 📋 Prerequisites

1. **Google Cloud Project** with Google Sheets API enabled
2. **Google Sheets API Key** (not OAuth - simpler for static apps)
3. **Public Google Sheet** or sheet shared with "Anyone with the link"

## 🔧 Google Sheets API Setup

### Step 1: Create Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing one
3. Enable the **Google Sheets API**:
   - Go to **APIs & Services** → **Library**
   - Search for "Google Sheets API"
   - Click **Enable**

### Step 2: Create API Key

1. Go to **APIs & Services** → **Credentials**
2. Click **Create Credentials** → **API Key**
3. **Important**: Restrict the API key:
   - Click on the created API key
   - Under **API restrictions**, select **Restrict key**
   - Choose **Google Sheets API**
   - Under **Website restrictions**, add your domain (optional but recommended)

### Step 3: Prepare Your Google Sheet

1. Create a new Google Sheet or use existing one
2. **Make it public**:
   - Click **Share** button
   - Change access to **Anyone with the link**
   - Set permission to **Viewer** (the app will append data via API)
3. **Get the Sheet ID** from the URL:
   ```
   https://docs.google.com/spreadsheets/d/SHEET_ID_HERE/edit
   ```
4. **Optional**: Add headers to your sheet:
   ```
   | Product Name | Price | Shop | Date | Added At |
   ```

## 🛠️ Local Development

### Setup

1. **Clone the repository**
2. **Install dependencies**:
   ```bash
   npm install
   ```
3. **Start development server**:
   ```bash
   npm run serve
   ```
4. **Open** `http://localhost:8080`

### First-time Configuration

1. Open the app in your browser
2. You'll see a setup section at the bottom
3. Enter your:
   - **Google Sheets API Key**
   - **Sheet ID** (from the URL)
   - **Sheet Name** (usually "Sheet1")
4. Click **Save Configuration**

## 🌐 Vercel Deployment

### Option 1: Deploy from GitHub

1. **Push your code** to GitHub repository
2. **Connect Vercel** to your GitHub account
3. **Import your repository** in Vercel
4. **Deploy** - Vercel will automatically detect Vue.js and build

### Option 2: Deploy with Vercel CLI

1. **Install Vercel CLI**:
   ```bash
   npm install -g vercel
   ```
2. **Deploy**:
   ```bash
   vercel --prod
   ```

### Option 3: One-Click Deploy

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/yourusername/product-price-tracker)

## 📁 Project Structure

```
project-root/
├── src/
│   ├── App.vue              # Main Vue component
│   └── main.js              # Vue.js entry point
├── public/
│   └── index.html           # HTML template
├── package.json             # Dependencies
├── vue.config.js            # Vue configuration
├── vercel.json              # Vercel deployment config
└── README.md                # This file
```

## 🔧 Configuration

The app stores configuration in browser's localStorage:

```javascript
{
  "apiKey": "your-google-sheets-api-key",
  "sheetId": "your-sheet-id",
  "sheetName": "Sheet1"
}
```

## 📱 Usage

1. **Configure the app** (first time only):
   - Enter your Google Sheets API key
   - Enter your Sheet ID
   - Specify sheet name (default: "Sheet1")

2. **Add products**:
   - Fill in product name, price, currency
   - Select shop from dropdown
   - Choose date
   - Click "Add Product"

3. **View recent products** in the list below the form

## 🔒 Security Notes

- **API Key**: Stored in browser localStorage (client-side only)
- **Sheet Access**: Make sure your sheet is properly configured for public access
- **CORS**: Google Sheets API supports CORS for browser requests
- **Rate Limits**: Google Sheets API has rate limits - the app handles basic errors

## 🐛 Troubleshooting

### Common Issues

1. **"Failed to add product"**
   - Check API key is valid and has Sheets API access
   - Verify sheet ID is correct
   - Ensure sheet is publicly accessible

2. **CORS errors**
   - This shouldn't happen with Google Sheets API
   - Check that you're using the correct API endpoints

3. **Rate limit errors**
   - Google Sheets API has quotas
   - Try again after a few minutes

### Debug Tips

- **Check browser console** for detailed error messages
- **Test API key** with a simple curl request:
  ```bash
  curl "https://sheets.googleapis.com/v4/spreadsheets/SHEET_ID?key=API_KEY"
  ```

## 🎯 Environment Variables (Optional)

For build-time configuration, you can use:

```bash
# .env.local
VUE_APP_DEFAULT_SHEET_ID=your-sheet-id
VUE_APP_DEFAULT_API_KEY=your-api-key
```

## 🚀 Performance

- **Static build** - Fast loading
- **Minimal dependencies** - Small bundle size
- **Client-side caching** - Recent products stored locally
- **Responsive design** - Works on all devices

## 📄 License

MIT License

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

---

**Need help?** Check the browser console for error messages or create an issue in the repository.