# Plugin Registry

Central registry for all dashboard plugins.

## Registry URL
- **Main Registry**: `https://raw.githubusercontent.com/yourusername/plugin-registry/main/registry.json`
- **Individual Plugins**: `https://raw.githubusercontent.com/yourusername/plugin-registry/main/plugins/{plugin-id}.json`

## Adding a Plugin
Plugins are automatically added via CI/CD when they're deployed.

## Registry Format
```json
{
  "version": "1.0.0",
  "lastUpdated": "2024-01-20T12:00:00.000Z",
  "plugins": [
    {
      "id": "plugin-id",
      "name": "Plugin Name",
      "version": "1.0.0",
      "manifestUrl": "https://raw.githubusercontent.com/.../plugins/plugin-id.json"
    }
  ]
}
```

Push to GitHub:
```bash
git add .
git commit -m "Initial plugin registry"
git push origin main
```

## Step 2: Update Your Plugin CI/CD

Update your plugin's `.github/workflows/plugin-ci.yml` to update the GitHub registry:

```yaml
name: Plugin CI/CD

on:
  push:
    branches: [ main ]

env:
  FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
  CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - name: Install and test client
      run: |
        cd src/client
        npm ci
        npm run build
    
    - name: Install and test server
      run: |
        cd src/server
        npm ci
        npm run build

  deploy-backend:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - uses: superfly/flyctl-actions/setup-flyctl@master
    
    - name: Deploy backend to Fly.io
      run: |
        cd src/server
        flyctl deploy --remote-only

  deploy-frontend:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build frontend
      run: |
        cd src/client
        npm ci
        npm run build
    
    - name: Deploy to Cloudflare Pages
      uses: cloudflare/pages-action@v1
      with:
        apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        projectName: ${{ github.event.repository.name }}-ui
        directory: src/client/dist

  update-registry:
    needs: [deploy-backend, deploy-frontend]
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      
    - name: Update Plugin Registry
      env:
        REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}
        PLUGIN_NAME: ${{ github.event.repository.name }}
      run: |
        # Clone the registry repo
        git clone https://${{ secrets.REGISTRY_TOKEN }}@github.com/yourusername/plugin-registry.git registry-repo
        cd registry-repo
        
        # Configure git
        git config user.name "Plugin CI"
        git config user.email "ci@yourapp.com"
        
        # Create/update the plugin manifest
        PLUGIN_ID="${{ github.event.repository.name }}"
        MANIFEST_FILE="plugins/${PLUGIN_ID}.json"
        
        # Read the source manifest and update URLs
        cat > "$MANIFEST_FILE" << 'EOF'
        {
          "id": "${{ github.event.repository.name }}",
          "name": "${{ github.event.repository.name }}",
          "version": "1.0.0",
          "description": "Plugin for dashboard",
          "ui": {
            "remoteEntry": "https://${{ github.event.repository.name }}-ui.pages.dev/assets/remoteEntry.js",
            "expose": "./PluginApp"
          },
          "server": {
            "routeBase": "/api/integrations/${{ github.event.repository.name }}",
            "deploymentUrl": "https://${{ github.event.repository.name }}.fly.dev",
            "healthCheck": "/api/health"
          },
          "manifestUrl": "https://raw.githubusercontent.com/yourusername/plugin-registry/main/plugins/${{ github.event.repository.name }}.json",
          "lastUpdated": "$(date -u +%Y-%m-%dT%H:%M:%S.000Z)"
        }
        EOF
        
        # Update main registry.json
        python3 << 'EOF'
        import json
        import os
        from datetime import datetime
        
        plugin_id = os.environ['PLUGIN_NAME']
        
        # Read current registry
        try:
            with open('registry.json', 'r') as f:
                registry = json.load(f)
        except:
            registry = {"version": "1.0.0", "plugins": []}
        
        # Remove existing plugin with same ID
        registry['plugins'] = [p for p in registry['plugins'] if p['id'] != plugin_id]
        
        # Add updated plugin
        registry['plugins'].append({
            "id": plugin_id,
            "name": plugin_id,
            "version": "1.0.0",
            "manifestUrl": f"https://raw.githubusercontent.com/yourusername/plugin-registry/main/plugins/{plugin_id}.json"
        })
        
        registry['lastUpdated'] = datetime.utcnow().isoformat() + 'Z'
        
        # Write updated registry
        with open('registry.json', 'w') as f:
            json.dump(registry, f, indent=2)
        EOF
        
        # Commit and push changes
        git add .
        git commit -m "Update plugin: ${{ github.event.repository.name }}" || echo "No changes to commit"
        git push origin main
```

## Step 3: Update Your Existing Next.js App

Since your Next.js app is already deployed, you'll add **client-side** plugin loading. Update your existing components:

### 3.1. Create Plugin Registry Client

Add to your existing Next.js app - `lib/pluginRegistry.ts`:

```typescript
export interface PluginManifest {
  id: string;
  name: string;
  version: string;
  ui: {
    remoteEntry: string;
    expose: string;
  };
  server: {
    routeBase: string;
    deploymentUrl?: string;
    healthCheck?: string;
  };
  manifestUrl: string;
  lastUpdated: string;
}

export interface Registry {
  version: string;
  lastUpdated: string;
  plugins: Array<{
    id: string;
    name: string;
    version: string;
    manifestUrl: string;
  }>;
}

const REGISTRY_URL = 'https://raw.githubusercontent.com/yourusername/plugin-registry/main/registry.json';

// Cache for performance
let registryCache: Registry | null = null;
let cacheTimestamp = 0;
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

export async function fetchAvailablePlugins(): Promise<PluginManifest[]> {
  try {
    // Check cache first
    const now = Date.now();
    if (registryCache && (now - cacheTimestamp) < CACHE_DURATION) {
      return await loadPluginManifests(registryCache.plugins);
    }

    // Fetch fresh registry
    const response = await fetch(REGISTRY_URL, {
      cache: 'no-cache', // Always get fresh data
    });
    
    if (!response.ok) {
      throw new Error(`Failed to fetch registry: ${response.status}`);
    }

    const registry: Registry = await response.json();
    
    // Update cache
    registryCache = registry;
    cacheTimestamp = now;

    // Load individual manifests
    return await loadPluginManifests(registry.plugins);
    
  } catch (error) {
    console.error('Failed to fetch plugin registry:', error);
    return [];
  }
}

async function loadPluginManifests(pluginList: Registry['plugins']): Promise<PluginManifest[]> {
  const manifests = await Promise.allSettled(
    pluginList.map(async (plugin) => {
      const response = await fetch(plugin.manifestUrl);
      if (!response.ok) {
        throw new Error(`Failed to fetch manifest for ${plugin.id}`);
      }
      return response.json();
    })
  );

  return manifests
    .filter((result): result is PromiseFulfilledResult<PluginManifest> => 
      result.status === 'fulfilled'
    )
    .map(result => result.value);
}

// Dynamic plugin loader (same as before)
const pluginCache = new Map<string, any>();

export async function loadPlugin(manifest: PluginManifest): Promise<any> {
  if (pluginCache.has(manifest.id)) {
    return pluginCache.get(manifest.id);
  }

  try {
    // Load the remote entry script
    const script = document.createElement('script');
    script.src = manifest.ui.remoteEntry;
    script.type = 'text/javascript';
    script.async = true;
    
    await new Promise((resolve, reject) => {
      script.onload = resolve;
      script.onerror = () => reject(new Error(`Failed to load ${manifest.ui.remoteEntry}`));
      document.head.appendChild(script);
    });

    // Get the remote container
    const containerName = manifest.id.replace(/[^a-zA-Z0-9]/g, ''); // Clean container name
    const container = (window as any)[containerName];
    
    if (!container) {
      throw new Error(`Container ${containerName} not found`);
    }

    // Initialize the container
    if (container.init) {
      await container.init((window as any).__webpack_share_scopes__?.default || {});
    }
    
    // Get the exposed module
    const factory = await container.get(manifest.ui.expose.replace('./', ''));
    const Module = factory();
    
    pluginCache.set(manifest.id, Module);
    return Module;
    
  } catch (error) {
    console.error(`Failed to load plugin ${manifest.id}:`, error);
    throw error;
  }
}
```

### 3.2. Plugin Loader Component (same as before)

```typescript
// components/PluginLoader.tsx - same as the previous implementation
```

## Step 4: Test the Complete Flow

1. **Create your registry repo** on GitHub
2. **Deploy a plugin** - the CI/CD will update the registry
3. **Your Next.js app** will automatically discover new plugins from GitHub
4. **Plugin URLs** will be like:
   - Registry: `https://raw.githubusercontent.com/yourusername/plugin-registry/main/registry.json`
   - Plugin Manifest: `https://raw.githubusercontent.com/yourusername/plugin-registry/main/plugins/calander2.json`

This approach gives you:
✅ **No changes to deployed Next.js app** - just add client-side code  
✅ **GitHub-hosted registry** - reliable, fast, version-controlled  
✅ **Automatic plugin discovery** - CI/CD updates registry automatically  
✅ **Zero downtime** - plugins can be added without app redeployment  

Your existing Vercel app just needs to read from the GitHub registry URL!