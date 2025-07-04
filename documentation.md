## Prérequis

- Node.js 18.x ou supérieur
- React Native CLI
- Android Studio (pour Android)
- Xcode (pour iOS)
- Java 11 ou supérieur

## Installation

1. **Cloner le repository**
   ```bash
   git clone https://github.com/brandnet/zacstore.git
   cd zacstore
   ```

2. **Installer les dépendances**
   ```bash
   npm install
   cd ios && pod install && cd ..
   ```

3. **Configuration des variables d'environnement**
   ```bash
   cp .env.example .env
   # Modifier les valeurs dans .env
   ```

4. **Lancement de l'application**
   ```bash
   # Android
   npx react-native run-android
   
   # iOS
   npx react-native run-ios
   ```

## Configuration

### Variables d'environnement

```env
# API Configuration
API_BASE_URL=https://api.zacstore.com
API_VERSION=v3
API_TIMEOUT=30000

# WooCommerce Configuration
WC_CONSUMER_KEY=your_consumer_key
WC_CONSUMER_SECRET=your_consumer_secret

# Authentication
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=24h

# Performance
ENABLE_PERFORMANCE_MONITORING=true
ENABLE_CRASH_REPORTING=true
```

### Configuration des notifications

```javascript
// config/notifications.js
export const notificationConfig = {
  enablePushNotifications: true,
  soundEnabled: true,
  vibrationEnabled: true,
  categories: {
    orders: 'Commandes',
    promotions: 'Promotions',
    wishlist: 'Liste de souhaits'
  }
};
```

## Structure du projet

```
zacstore/
├── src/
│   ├── components/          # Composants réutilisables
│   ├── screens/            # Écrans de l'application
│   ├── navigation/         # Configuration de navigation
│   ├── store/             # Configuration Redux
│   ├── services/          # Services API
│   ├── utils/             # Utilitaires
│   ├── hooks/             # Hooks personnalisés
│   └── assets/            # Images, fonts, etc.
├── android/               # Code Android natif
├── ios/                   # Code iOS natif
└── __tests__/            # Tests
```
```

Cette continuation du rapport de stage comprend :

1. **Complétion du composant RobustImage** avec toutes les fonctionnalités de gestion d'erreurs et de retry
2. **Service API WooCommerce complet** avec authentification JWT, gestion des tokens et queue de requêtes
3. **Hooks personnalisés** pour la gestion des performances et la surveillance mémoire
4. **Architecture Redux détaillée** avec stores, slices et actions asynchrones
5. **Configuration d'optimisation** avec Metro bundler et paramètres de performance
6. **Tests unitaires et d'intégration** pour valider les composants et la logique métier
7. **Métriques de performance** avant/après optimisation avec analyses détaillées
8. **Documentation technique** complète pour l'installation et la configuration

Le document maintient le niveau de détail technique approprié pour un rapport de stage professionnel tout en démontrant une expertise approfondie en développement React Native et en optimisation d'applications mobiles.
