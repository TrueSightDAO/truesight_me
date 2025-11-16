# Exchange Rates Update Approach

## Overview
Google App Script will update `data/exchange-rates.json` directly via GitHub API, and `index.html` will load it dynamically on page load.

## Implementation Steps

### 1. Update `index.html` to fetch JSON dynamically

Replace the inlined data with a fetch call:

```javascript
// Load exchange rates from JSON file
async function loadExchangeRates() {
  try {
    const response = await fetch('data/exchange-rates.json');
    if (!response.ok) throw new Error('Failed to load exchange rates');
    const json = await response.json();
    return json.data; // Extract the data object
  } catch (error) {
    console.error('Error loading exchange rates:', error);
    // Return fallback data or show error message
    return null;
  }
}

// Populate stats when data loads
loadExchangeRates().then(exchangeRatesData => {
  if (!exchangeRatesData) {
    // Show error message or use cached/fallback data
    return;
  }
  
  // Use existing formatStatValue logic
  const statElements = document.querySelectorAll('.stat-value[data-key]');
  statElements.forEach(function(el) {
    const key = el.getAttribute('data-key');
    const data = exchangeRatesData[key];
    el.textContent = formatStatValue(key, data);
  });
});
```

### 2. Create Google App Script function to update GitHub

Example function (based on existing patterns in your codebase):

```javascript
/**
 * Update exchange-rates.json on GitHub
 * @param {Object} rates - Exchange rates data object
 */
function updateExchangeRatesOnGitHub(rates) {
  const GITHUB_TOKEN = creds.GITHUB_API_TOKEN; // From your credentials
  const REPO_OWNER = 'TrueSightDAO';
  const REPO_NAME = 'truesight_me';
  const BRANCH = 'main';
  const FILE_PATH = 'data/exchange-rates.json';
  
  // Create the JSON structure
  const jsonContent = {
    timestamp: new Date().toISOString(),
    description: "TrueSight DAO financial metrics and exchange rates",
    data: rates
  };
  
  // Convert to JSON string
  const content = JSON.stringify(jsonContent, null, 2);
  const encodedContent = Utilities.base64Encode(content);
  
  // Get current file SHA (required for updates)
  const apiUrl = `https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/contents/${FILE_PATH}`;
  const getResponse = UrlFetchApp.fetch(apiUrl, {
    headers: {
      'Authorization': `token ${GITHUB_TOKEN}`,
      'Accept': 'application/vnd.github.v3+json'
    },
    muteHttpExceptions: true
  });
  
  let sha = null;
  if (getResponse.getResponseCode() === 200) {
    const fileData = JSON.parse(getResponse.getContentText());
    sha = fileData.sha;
  }
  
  // Update file
  const payload = {
    message: `Update exchange rates - ${new Date().toISOString()}`,
    content: encodedContent,
    branch: BRANCH
  };
  
  if (sha) {
    payload.sha = sha; // Required for updates
  }
  
  const options = {
    method: 'PUT',
    headers: {
      'Authorization': `token ${GITHUB_TOKEN}`,
      'Accept': 'application/vnd.github.v3+json'
    },
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };
  
  const response = UrlFetchApp.fetch(apiUrl, options);
  const status = response.getResponseCode();
  
  if (status === 200 || status === 201) {
    Logger.log('✅ Successfully updated exchange-rates.json on GitHub');
    return true;
  } else {
    Logger.log(`❌ Failed to update: ${response.getContentText()}`);
    return false;
  }
}
```

### 3. Integrate with existing App Script workflow

Call this function when your exchange rates are updated (e.g., after reading from Google Sheets or calculating values).

### 4. Hosting considerations

- **GitHub Pages**: JSON will be served from `https://username.github.io/truesight_me/data/exchange-rates.json`
- **Custom domain**: Same structure
- **Local development**: Use a local server (e.g., `python -m http.server` or `npx serve`)

### 5. Error handling and fallbacks

- Show loading state while fetching
- Cache last known values in localStorage
- Display error message if fetch fails
- Consider retry logic for network issues

## Benefits

1. ✅ **Real-time updates** - No manual sync needed
2. ✅ **Single source of truth** - JSON file is authoritative
3. ✅ **Automated** - App Script handles updates
4. ✅ **Version controlled** - All updates tracked in Git
5. ✅ **Simple deployment** - Just push HTML changes, data updates automatically

## Migration Path

1. Update `index.html` to fetch JSON (keep inlined as fallback initially)
2. Test with GitHub Pages or local server
3. Create/update App Script function
4. Test App Script → GitHub update flow
5. Remove inlined data once confirmed working
6. Remove sync script dependency (optional)

