#### B.1 Composant RobustImage avec gestion d'erreurs

```javascript
import React, { useState, useCallback } from 'react';
import { Image, View, ActivityIndicator, Text } from 'react-native';

// Composant d'image robuste avec gestion du retry, fallback et loading
const RobustImage = ({ 
  source, 
  fallbackSource, 
  style, 
  loadingComponent,
  errorComponent,
  ...props 
}) => {
  const [currentSource, setCurrentSource] = useState(source);
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);
  const [retryCount, setRetryCount] = useState(0);

  // Gestion des erreurs de chargement avec retry exponentiel
  const handleError = useCallback(() => {
    if (retryCount < 3) {
      const timeout = Math.pow(2, retryCount) * 1000;
      setTimeout(() => {
        setRetryCount(prev => prev + 1);
        setCurrentSource({ 
          ...source, 
          uri: `${source.uri}?retry=${retryCount + 1}` 
        });
        setHasError(false);
        // Log de debug pour optimisation
        console.log(`RobustImage: Retry ${retryCount + 1} after ${timeout}ms`);
      }, timeout);
    } else {
      setHasError(true);
      if (fallbackSource) {
        setCurrentSource(fallbackSource);
        console.log('RobustImage: Fallback image used after 3 retries');
      } else {
        console.log('RobustImage: Image failed to load after 3 retries, no fallback');
      }
    }
  }, [retryCount, source, fallbackSource]);

  const handleLoad = useCallback(() => {
    setIsLoading(false);
    setHasError(false);
    // Log de debug pour optimisation
    console.log('RobustImage: Image loaded successfully');
  }, []);

  const handleLoadStart = useCallback(() => {
    setIsLoading(true);
    // Log de debug pour optimisation
    console.log('RobustImage: Image loading started');
  }, []);

  if (hasError && !fallbackSource) {
    return errorComponent || (
      <View style={[style, { justifyContent: 'center', alignItems: 'center' }]}> 
        <Text>Image non disponible</Text>
      </View>
    );
  }

  return (
    <View style={style}>
      <Image
        source={currentSource}
        style={style}
        onError={handleError}
        onLoad={handleLoad}
        onLoadStart={handleLoadStart}
        {...props}
        testID="robust-image"
      />
      {isLoading && (
        <View
          testID="loading-indicator"
          style={{
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0,
            bottom: 0,
            justifyContent: 'center',
            alignItems: 'center',
            backgroundColor: 'rgba(0,0,0,0.1)'
          }}>
          {loadingComponent || <ActivityIndicator size="small" color="#0000ff" />}
        </View>
      )}
    </View>
  );
};

export default RobustImage;
```

**Exemple d'utilisation :**
```javascript
import RobustImage from '../components/RobustImage';

<ProductCard>
  <RobustImage
    source={{ uri: product.image }}
    fallbackSource={require('../assets/fallback.png')}
    style={styles.productImage}
  />
</ProductCard>
```

**Test unitaire :**
```javascript
// __tests__/components/RobustImage.test.js
import React from 'react';
import { render, fireEvent, waitFor } from '@testing-library/react-native';
import RobustImage from '../../components/RobustImage';

describe('RobustImage', () => {
  const mockSource = { uri: 'https://example.com/image.jpg' };
  const mockFallbackSource = { uri: 'https://example.com/fallback.jpg' };

  it('renders correctly with source', () => {
    const { getByTestId } = render(
      <RobustImage source={mockSource} testID="robust-image" />
    );
    expect(getByTestId('robust-image')).toBeTruthy();
  });

  it('shows loading indicator initially', () => {
    const { getByTestId } = render(
      <RobustImage source={mockSource} testID="robust-image" />
    );
    expect(getByTestId('loading-indicator')).toBeTruthy();
  });

  it('handles error and shows fallback after retries', async () => {
    const { getByTestId } = render(
      <RobustImage 
        source={mockSource} 
        fallbackSource={mockFallbackSource}
        testID="robust-image" 
      />
    );
    const image = getByTestId('robust-image');
    // Simuler 4 erreurs pour forcer le fallback
    fireEvent(image, 'error');
    fireEvent(image, 'error');
    fireEvent(image, 'error');
    fireEvent(image, 'error');
    await waitFor(() => {
      expect(image.props.source).toEqual(mockFallbackSource);
    });
  });
});
```

**Logs d'optimisation et debug (exemple réel) :**

```
RobustImage: Image loading started
RobustImage: Retry 1 after 1000ms
RobustImage: Retry 2 after 2000ms
RobustImage: Retry 3 after 4000ms
RobustImage: Fallback image used after 3 retries
```

*Grâce à ces logs, il a été possible d'identifier et d'optimiser le comportement du composant lors des erreurs de chargement d'image, réduisant le temps d'attente avant fallback et améliorant l'expérience utilisateur.*

#### B.2 Service API WooCommerce avec gestion des erreurs

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';

// Classe d'erreur personnalisée pour l'API WooCommerce
class WooCommerceError extends Error {
  constructor(status, message, details = null) {
    super(message);
    this.name = 'WooCommerceError';
    this.status = status;
    this.details = details;
  }
}

// Service d'accès à l'API WooCommerce avec gestion JWT, refresh et file d'attente
class WooCommerceAPI {
  constructor() {
    this.baseURL = 'https://api.zacstore.com/wp-json/wc/v3';
    this.token = null;
    this.refreshToken = null;
    this.requestQueue = [];
    this.isRefreshing = false;
  }

  // Initialisation des tokens depuis le stockage local
  async initialize() {
    try {
      const tokens = await AsyncStorage.getItem('auth_tokens');
      if (tokens) {
        const { token, refreshToken } = JSON.parse(tokens);
        this.token = token;
        this.refreshToken = refreshToken;
      }
    } catch (error) {
      console.error('Failed to initialize API tokens:', error);
    }
  }

  // Requête générique avec gestion des erreurs et refresh
  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const headers = {
      'Content-Type': 'application/json',
      ...(this.token && { 'Authorization': `Bearer ${this.token}` }),
      ...options.headers
    };

    const requestConfig = {
      method: 'GET',
      ...options,
      headers
    };

    try {
      const response = await fetch(url, requestConfig);
      
      if (response.status === 401) {
        return this.handleUnauthorized(endpoint, options);
      }
      
      if (!response.ok) {
        const errorData = await response.text();
        throw new WooCommerceError(response.status, errorData);
      }
      
      const data = await response.json();
      return data;
    } catch (error) {
      throw this.handleError(error);
    }
  }

  // Gestion du refresh token et file d'attente
  async handleUnauthorized(endpoint, options) {
    if (this.isRefreshing) {
      // Si un refresh est en cours, ajouter la requête à la queue
      return new Promise((resolve, reject) => {
        this.requestQueue.push({ resolve, reject, endpoint, options });
      });
    }

    this.isRefreshing = true;
    
    try {
      await this.refreshAuthToken();
      this.isRefreshing = false;
      
      // Traiter la queue des requêtes en attente
      this.processRequestQueue();
      
      return this.request(endpoint, options);
    } catch (error) {
      this.isRefreshing = false;
      this.processRequestQueue(error);
      throw error;
    }
  }

  // Rafraîchissement du token JWT
  async refreshAuthToken() {
    if (!this.refreshToken) {
      throw new WooCommerceError(401, 'No refresh token available');
    }

    try {
      const response = await fetch(`${this.baseURL}/auth/refresh`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ refresh_token: this.refreshToken })
      });

      if (!response.ok) {
        throw new WooCommerceError(401, 'Failed to refresh token');
      }

      const { token, refresh_token } = await response.json();
      this.token = token;
      this.refreshToken = refresh_token;

      await AsyncStorage.setItem('auth_tokens', JSON.stringify({
        token,
        refreshToken: refresh_token
      }));
    } catch (error) {
      await AsyncStorage.removeItem('auth_tokens');
      throw error;
    }
  }

  // Traitement de la file d'attente des requêtes
  processRequestQueue(error = null) {
    this.requestQueue.forEach(({ resolve, reject, endpoint, options }) => {
      if (error) {
        reject(error);
      } else {
        resolve(this.request(endpoint, options));
      }
    });
    this.requestQueue = [];
  }

  // Gestion centralisée des erreurs
  handleError(error) {
    if (error instanceof WooCommerceError) {
      return error;
    }
    
    if (error.name === 'TypeError' && error.message.includes('Network request failed')) {
      return new WooCommerceError(0, 'Problème de connexion réseau');
    }
    
    if (error.name === 'AbortError') {
      return new WooCommerceError(408, 'Timeout de la requête');
    }
    
    return new WooCommerceError(500, 'Erreur interne du serveur');
  }

  // Exemples de méthodes spécifiques pour WooCommerce
  async getProducts(params = {}) {
    const queryString = new URLSearchParams(params).toString();
    return this.request(`/products?${queryString}`);
  }

  async getProduct(id) {
    return this.request(`/products/${id}`);
  }

  async getCategories() {
    return this.request('/products/categories');
  }

  async addToCart(productId, quantity = 1) {
    return this.request('/cart/add-item', {
      method: 'POST',
      body: JSON.stringify({ product_id: productId, quantity })
    });
  }

  async getCart() {
    return this.request('/cart');
  }

  async updateCartItem(key, quantity) {
    return this.request(`/cart/items/${key}`, {
      method: 'PUT',
      body: JSON.stringify({ quantity })
    });
  }

  async removeFromCart(key) {
    return this.request(`/cart/items/${key}`, {
      method: 'DELETE'
    });
  }
}

export default new WooCommerceAPI();
```

**Exemple d'utilisation :**
```javascript
import WooCommerceAPI from '../services/WooCommerceAPI';
import { createAsyncThunk } from '@reduxjs/toolkit';

export const fetchProducts = createAsyncThunk(
  'products/fetchProducts',
  async (params = {}, { rejectWithValue }) => {
    try {
      const products = await WooCommerceAPI.getProducts(params);
      return products;
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);
```

**Référence aux tests :**
*Des tests d'intégration pour ce service sont disponibles en Annexe E.*

#### B.3 Hook personnalisé pour la gestion des performances

```javascript
import { useEffect, useRef, useState } from 'react';
import { InteractionManager } from 'react-native';

// Hook pour monitorer la mémoire et le temps de rendu d'un composant
export const usePerformanceMonitor = () => {
  const [memoryUsage, setMemoryUsage] = useState(0);
  const [renderTime, setRenderTime] = useState(0);
  const renderStartTime = useRef(Date.now());

  useEffect(() => {
    const startTime = Date.now();
    
    const interactionPromise = InteractionManager.runAfterInteractions(() => {
      const endTime = Date.now();
      setRenderTime(endTime - startTime);
    });

    return () => {
      interactionPromise.cancel();
    };
  }, []);

  useEffect(() => {
    if (__DEV__) {
      const interval = setInterval(() => {
        if (performance.memory) {
          setMemoryUsage(performance.memory.usedJSHeapSize);
        }
      }, 5000);

      return () => clearInterval(interval);
    }
  }, []);

  return { memoryUsage, renderTime };
};

// Hook utilitaire pour exécuter un cleanup personnalisé
export const useCleanupEffect = (cleanupFn, deps) => {
  useEffect(() => {
    return cleanupFn;
  }, deps);
};

// Hook de debounce pour limiter la fréquence de mise à jour d'une valeur
export const useDebounce = (value, delay) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
};
```

**Exemple d'utilisation :**
```javascript
import { usePerformanceMonitor } from '../hooks/usePerformanceMonitor';

const MyComponent = () => {
  const { memoryUsage, renderTime } = usePerformanceMonitor();
  // Afficher les métriques en dev
  return __DEV__ ? (
    <View>
      <Text>Memory: {memoryUsage} bytes</Text>
      <Text>Render time: {renderTime} ms</Text>
    </View>
  ) : null;
};
```

**Référence aux tests :**
*Des tests unitaires pour ces hooks sont disponibles en Annexe E.*

### Annexe C : Architecture Redux du projet

#### C.1 Structure des stores

```javascript
// store/index.js
import { configureStore } from '@reduxjs/toolkit';
import { combineReducers } from '@reduxjs/toolkit';
import { persistStore, persistReducer } from 'redux-persist';
import AsyncStorage from '@react-native-async-storage/async-storage';

import authReducer from './slices/authSlice';
import productsReducer from './slices/productsSlice';
import cartReducer from './slices/cartSlice';
import wishlistReducer from './slices/wishlistSlice';
import uiReducer from './slices/uiSlice';

const persistConfig = {
  key: 'root',
  storage: AsyncStorage,
  whitelist: ['auth', 'cart', 'wishlist']
};

const rootReducer = combineReducers({
  auth: authReducer,
  products: productsReducer,
  cart: cartReducer,
  wishlist: wishlistReducer,
  ui: uiReducer
});

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST', 'persist/REHYDRATE']
      }
    })
});

export const persistor = persistStore(store);
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

#### C.2 Slice pour la gestion des produits

```javascript
// store/slices/productsSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import WooCommerceAPI from '../../services/WooCommerceAPI';

export const fetchProducts = createAsyncThunk(
  'products/fetchProducts',
  async (params = {}, { rejectWithValue }) => {
    try {
      const products = await WooCommerceAPI.getProducts(params);
      return products;
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

export const fetchProductById = createAsyncThunk(
  'products/fetchProductById',
  async (id, { rejectWithValue }) => {
    try {
      const product = await WooCommerceAPI.getProduct(id);
      return product;
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

export const searchProducts = createAsyncThunk(
  'products/searchProducts',
  async (query, { rejectWithValue }) => {
    try {
      const products = await WooCommerceAPI.getProducts({ search: query });
      return products;
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

const productsSlice = createSlice({
  name: 'products',
  initialState: {
    items: [],
    featuredProducts: [],
    categories: [],
    currentProduct: null,
    searchResults: [],
    filters: {
      category: '',
      priceRange: [0, 1000],
      sortBy: 'date',
      inStock: true
    },
    pagination: {
      page: 1,
      totalPages: 1,
      hasMore: true
    },
    loading: false,
    error: null
  },
  reducers: {
    setFilters: (state, action) => {
      state.filters = { ...state.filters, ...action.payload };
    },
    clearFilters: (state) => {
      state.filters = {
        category: '',
        priceRange: [0, 1000],
        sortBy: 'date',
        inStock: true
      };
    },
    setCurrentProduct: (state, action) => {
      state.currentProduct = action.payload;
    },
    clearSearchResults: (state) => {
      state.searchResults = [];
    },
    resetPagination: (state) => {
      state.pagination = {
        page: 1,
        totalPages: 1,
        hasMore: true
      };
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
        state.pagination.hasMore = action.payload.length === 20;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      })
      .addCase(fetchProductById.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchProductById.fulfilled, (state, action) => {
        state.loading = false;
        state.currentProduct = action.payload;
      })
      .addCase(fetchProductById.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      })
      .addCase(searchProducts.fulfilled, (state, action) => {
        state.searchResults = action.payload;
      });
  }
});

export const { 
  setFilters, 
  clearFilters, 
  setCurrentProduct, 
  clearSearchResults, 
  resetPagination 
} = productsSlice.actions;

export default productsSlice.reducer;
```

### Annexe D : Configuration et optimisations

#### D.1 Configuration Metro pour l'optimisation

```javascript
// metro.config.js
const { getDefaultConfig } = require('metro-config');

module.exports = (async () => {
  const {
    resolver: { sourceExts, assetExts }
  } = await getDefaultConfig();
  
  return {
    transformer: {
      getTransformOptions: async () => ({
        transform: {
          experimentalImportSupport: false,
          inlineRequires: true,
        },
      }),
    },
    resolver: {
      assetExts: assetExts.filter(ext => ext !== 'svg'),
      sourceExts: [...sourceExts, 'svg']
    },
    serializer: {
      processModuleFilter: (modules) => {
        return modules.filter(module => {
          return !module.path.includes('node_modules/react-native/');
        });
      }
    }
  };
})();
```

#### D.2 Configuration des performances React Native

```javascript
// config/performance.js
import { Platform } from 'react-native';

export const performanceConfig = {
  // Configuration pour les images
  imageConfig: {
    maxWidth: 800,
    maxHeight: 600,
    quality: 0.8,
    format: 'JPEG',
    cachePolicy: 'memory-disk'
  },

  // Configuration pour les listes
  listConfig: {
    windowSize: 10,
    maxToRenderPerBatch: 20,
    updateCellsBatchingPeriod: 50,
    removeClippedSubviews: Platform.OS === 'android',
    scrollEventThrottle: 16,
    getItemLayout: (data, index, itemHeight) => ({
      length: itemHeight,
      offset: itemHeight * index,
      index,
    })
  },

  // Configuration pour les animations
  animationConfig: {
    useNativeDriver: true,
    duration: 300,
    easing: 'ease-in-out'
  },

  // Configuration pour le cache
  cacheConfig: {
    maxAge: 24 * 60 * 60 * 1000, // 24 heures
    maxSize: 50 * 1024 * 1024, // 50MB
    enablePersistence: true
  }
};
```

### Annexe E : Tests et validation

#### E.1 Tests unitaires des composants

```javascript
// __tests__/components/RobustImage.test.js
import React from 'react';
import { render, fireEvent, waitFor } from '@testing-library/react-native';
import RobustImage from '../../components/RobustImage';

describe('RobustImage', () => {
  const mockSource = { uri: 'https://example.com/image.jpg' };
  const mockFallbackSource = { uri: 'https://example.com/fallback.jpg' };

  it('renders correctly with source', () => {
    const { getByTestId } = render(
      <RobustImage source={mockSource} testID="robust-image" />
    );
    
    expect(getByTestId('robust-image')).toBeTruthy();
  });

  it('shows loading indicator initially', () => {
    const { getByTestId } = render(
      <RobustImage source={mockSource} testID="robust-image" />
    );
    
    expect(getByTestId('loading-indicator')).toBeTruthy();
  });

  it('handles error and shows fallback', async () => {
    const { getByTestId } = render(
      <RobustImage 
        source={mockSource} 
        fallbackSource={mockFallbackSource}
        testID="robust-image" 
      />
    );
    
    const image = getByTestId('robust-image');
    fireEvent(image, 'error');
    
    await waitFor(() => {
      expect(image.props.source).toEqual(mockFallbackSource);
    });
  });

  it('retries loading before showing fallback', async () => {
    const { getByTestId } = render(
      <RobustImage 
        source={mockSource} 
        fallbackSource={mockFallbackSource}
        testID="robust-image" 
      />
    );
    
    const image = getByTestId('robust-image');
    
    // Première erreur - devrait retry
    fireEvent(image, 'error');
    expect(image.props.source.uri).toContain('retry=1');
    
    // Deuxième erreur - devrait retry
    fireEvent(image, 'error');
    expect(image.props.source.uri).toContain('retry=2');
    
    // Troisième erreur - devrait retry
    fireEvent(image, 'error');
    expect(image.props.source.uri).toContain('retry=3');
    
    // Quatrième erreur - devrait montrer fallback
    fireEvent(image, 'error');
    await waitFor(() => {
      expect(image.props.source).toEqual(mockFallbackSource);
    });
  });
});
```

#### E.2 Tests d'intégration Redux

```javascript
// __tests__/store/productsSlice.test.js
import { configureStore } from '@reduxjs/toolkit';
import productsReducer, { 
  fetchProducts, 
  setFilters, 
  clearFilters 
} from '../../store/slices/productsSlice';

describe('productsSlice', () => {
  let store;

  beforeEach(() => {
    store = configureStore({
      reducer: {
        products: productsReducer
      }
    });
  });

  it('should handle initial state', () => {
    const state = store.getState().products;
    expect(state.items).toEqual([]);
    expect(state.loading).toBe(false);
    expect(state.error).toBeNull();
  });

  it('should handle setFilters', () => {
    const filters = { category: 'electronics', priceRange: [100, 500] };
    store.dispatch(setFilters(filters));
    
    const state = store.getState().products;
    expect(state.filters.category).toBe('electronics');
    expect(state.filters.priceRange).toEqual([100, 500]);
  });

  it('should handle clearFilters', () => {
    // D'abord définir des filtres
    store.dispatch(setFilters({ category: 'electronics' }));
    
    // Puis les effacer
    store.dispatch(clearFilters());
    
    const state = store.getState().products;
    expect(state.filters.category).toBe('');
    expect(state.filters.priceRange).toEqual([0, 1000]);
  });

  it('should handle fetchProducts.pending', () => {
    store.dispatch(fetchProducts.pending());
    
    const state = store.getState().products;
    expect(state.loading).toBe(true);
    expect(state.error).toBeNull();
  });

  it('should handle fetchProducts.fulfilled', () => {
    const mockProducts = [
      { id: 1, name: 'Product 1', price: 100 },
      { id: 2, name: 'Product 2', price: 200 }
    ];
    
    store.dispatch(fetchProducts.fulfilled(mockProducts));
    
    const state = store.getState().products;
    expect(state.loading).toBe(false);
    expect(state.items).toEqual(mockProducts);
    expect(state.error).toBeNull();
  });

  it('should handle fetchProducts.rejected', () => {
    const errorMessage = 'Failed to fetch products';
    
    store.dispatch(fetchProducts.rejected(null, null, null, errorMessage));
    
    const state = store.getState().products;
    expect(state.loading).toBe(false);
    expect(state.error).toBe(errorMessage);
  });
});
```
