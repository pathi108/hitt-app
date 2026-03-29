# Nutrition & Supplement Tracker

## Overview
A comprehensive nutrition and supplement tracking system for the HIIT LIFE app that helps users manage their supplement intake, track consumption, and receive low-stock alerts.

## 📁 File Structure

```
hiit-app/
├── src/utils/
│   └── supplementStorage.ts          # Data model & AsyncStorage management
├── app/(tabs)/
│   └── favorites.tsx                 # Main nutrition tracker screen
└── app/
    └── supplement-config.tsx         # Add/Edit supplement screen
```

## 🎯 Features

### 1. Data Model (`supplementStorage.ts`)
Complete TypeScript data model with AsyncStorage integration:

**Supplement Interface:**
- `id`: Unique identifier
- `name`: Supplement name
- `type`: Category (Protein Shake, Pre-workout, etc.)
- `servingsPerPackage`: Total servings in package
- `servingsConsumed`: Servings already consumed
- `lastLogged`: Last consumption timestamp
- `consumptionTimes`: Scheduled consumption times (HH:MM)
- `lowThreshold`: Low-stock alert threshold
- `createdAt`: Creation timestamp

**Available Functions:**
- `getAllSupplements()` - Retrieve all supplements
- `getSupplementById(id)` - Get specific supplement
- `addSupplement(data)` - Create new supplement
- `updateSupplement(id, updates)` - Update existing supplement
- `deleteSupplement(id)` - Remove supplement
- `consumeServings(id, count)` - Log consumption
- `getServingsRemaining(supplement)` - Calculate remaining servings
- `isRunningLow(supplement)` - Check if below threshold
- `getNextConsumptionTime(supplement)` - Get next scheduled time

### 2. Main List Screen (`favorites.tsx`)

**Collapsible Type Sections:**
- Supplements grouped by type (Protein Shake, Pre-workout, etc.)
- Color-coded type indicators
- Expandable/collapsible sections
- Item count per category

**Supplement Cards Display:**
- Name with type badge
- Servings remaining counter
- Low stock warning (⚠️ LOW)
- Out of stock indicator (OUT)
- Next scheduled consumption time
- Last logged timestamp
- Quick action buttons:
  - **Consume 1** - Log a serving with loading state
  - **Edit** - Navigate to edit screen
  - **Delete** - Remove with confirmation

**Features:**
- Real-time stock warnings
- Automatic reload on focus
- Loading states with spinners
- Empty state with helpful message
- Bottom "Add Supplement" button

### 3. Add/Edit Screen (`supplement-config.tsx`)

**Form Fields:**
1. **Supplement Name** (required)
   - Text input with placeholder
   - Validation on save

2. **Type** (required)
   - 7 predefined categories
   - Single-selection grid buttons
   - Visual active state

3. **Servings per Package** (required)
   - Number input
   - Validates positive integers
   - Helper text explains purpose

4. **Low Stock Threshold**
   - Default: 5 servings
   - Triggers alerts when reached
   - Customizable per supplement

5. **Consumption Schedule** (optional)
   - Multiple time entries (HH:MM format)
   - Add/remove time chips
   - Time format validation
   - Sorted display
   - Placeholder for future notifications

**Actions:**
- **Save** - Create or update supplement
- **Delete** - Remove supplement (edit mode only)
- **Back** - Cancel and return

**Validation:**
- All required fields checked
- Time format validation (HH:MM)
- Positive number validation
- Duplicate time prevention

## 🎨 Design System

### Colors (from AppTheme)
- **Background**: `#252525` (dark)
- **Foreground**: `#FBFBFB` (white text)
- **Primary Accent**: `#39FF14` (neon green)
- **Muted Text**: `#B5B5B5` (secondary text)
- **Card Background**: `#252525`
- **Border**: `#454545`
- **Destructive**: `#D4183D` (red for delete)

### Type Colors
```typescript
{
  'Protein Shake': '#FF6B6B',    // Red
  'Pre-workout': '#4ECDC4',      // Teal
  'Post-workout': '#95E1D3',     // Light teal
  'Hydration': '#4A90E2',        // Blue
  'Micronutrient': '#F7B731',    // Yellow
  'Carbs/Energy Gel': '#FFA07A', // Orange
  'Other': '#9B9B9B'             // Gray
}
```

### Typography
- **Header**: 32px, bold
- **Subtitle**: 14px, muted
- **Card Title**: 18px, semi-bold
- **Body**: 16px, normal
- **Helper Text**: 12px, muted

### Layout
- **Padding**: 20px horizontal
- **Border Radius**: 10px (lg), 8px (md), 6px (sm)
- **Gap**: 8px between elements
- **Card Padding**: 16px

## 🔔 Notifications (Placeholder)

Currently includes UI elements and data structure for future implementation:

### Planned Features:
1. **Consumption Reminders**
   - Alert at scheduled times
   - Based on `consumptionTimes` array
   - Can be configured per supplement

2. **Low Stock Alerts**
   - Triggered when `servingsRemaining <= lowThreshold`
   - Shown after consumption
   - Visual warnings in UI

3. **Out of Stock Warnings**
   - Prevent consumption when empty
   - Suggest reordering

### Implementation Notes:
```typescript
// Placeholder for Expo Notifications integration
// TODO: Set up scheduled notifications
// - Use Expo Notifications API
// - Schedule based on consumptionTimes
// - Trigger alerts on low threshold
```

## 📊 Data Flow

### Adding a Supplement
```
User → Tap "Add Supplement"
     → Fill form (name, type, servings, times)
     → Tap "Save"
     → Validation
     → addSupplement()
     → AsyncStorage
     → Navigate back
     → Refresh list
```

### Consuming a Serving
```
User → Tap "Consume 1" button
     → consumeServings(id, 1)
     → Update servingsConsumed
     → Log timestamp to lastLogged
     → Save consumption log
     → Check if running low
     → Show alert if threshold reached
     → Refresh display
```

### Editing a Supplement
```
User → Tap "Edit" button
     → Load existing data
     → Modify fields
     → Tap "Update"
     → updateSupplement()
     → AsyncStorage
     → Navigate back
     → Refresh list
```

## 🚀 Usage Examples

### Example 1: Track Pre-Workout
```typescript
const preWorkout = {
  name: "C4 Pre-Workout",
  type: "Pre-workout",
  servingsPerPackage: 30,
  lowThreshold: 5,
  consumptionTimes: ["06:00", "18:00"]
};
```

User gets:
- Reminder at 6:00 AM and 6:00 PM (future)
- Alert when 5 servings remaining
- Quick log button each workout

### Example 2: Track Protein Powder
```typescript
const protein = {
  name: "Whey Protein",
  type: "Protein Shake",
  servingsPerPackage: 60,
  lowThreshold: 10,
  consumptionTimes: ["07:30", "20:00"]
};
```

User gets:
- Organized under "Protein Shake" section
- Color-coded badge
- Consumption tracking
- Low stock warning at 10 servings

## 🔧 Future Enhancements

### Phase 1 (Current)
- ✅ Data model with TypeScript
- ✅ AsyncStorage integration
- ✅ CRUD operations
- ✅ Consumption logging
- ✅ Low stock warnings
- ✅ Type-based organization
- ✅ Collapsible sections

### Phase 2 (Planned)
- ⏳ Expo Notifications integration
- ⏳ Scheduled consumption reminders
- ⏳ Push notifications for low stock
- ⏳ Consumption history graph
- ⏳ Analytics dashboard

### Phase 3 (Future)
- 📋 Macro tracking
- 📋 Meal logging
- 📋 Barcode scanning
- 📋 Shopping list generation
- 📋 Cloud sync
- 📋 Export consumption data

## 🧪 Testing Checklist

- [ ] Add new supplement
- [ ] Edit existing supplement
- [ ] Delete supplement with confirmation
- [ ] Consume 1 serving
- [ ] Consume until low stock (see warning)
- [ ] Consume until out of stock
- [ ] Add multiple consumption times
- [ ] Remove consumption times
- [ ] Validate time format (HH:MM)
- [ ] Collapse/expand type sections
- [ ] See sample data on first launch
- [ ] Navigate between screens
- [ ] Data persists after app restart

## 📱 Screen Navigation

```
[Nutrition Tab] (favorites.tsx)
    ├─ [Tap "+ Add Supplement"]
    │   └─> [Supplement Config] (supplement-config.tsx)
    │        ├─ Fill form
    │        ├─ [Tap "Save"]
    │        └─> Back to [Nutrition Tab]
    │
    ├─ [Tap "Edit" on card]
    │   └─> [Supplement Config] (supplement-config.tsx?id=123)
    │        ├─ Load existing data
    │        ├─ [Tap "Update"]
    │        └─> Back to [Nutrition Tab]
    │
    └─ [Tap "Consume 1"]
         └─ Log consumption
         └─ Refresh display
         └─ Show alert if low
```

## 💾 Data Storage

### AsyncStorage Keys
- `@hiit_supplements` - Main supplements array
- `@hiit_supplement_logs` - Consumption history logs

### Sample Data Structure
```json
{
  "id": "1672531200000",
  "name": "Whey Protein",
  "type": "Protein Shake",
  "servingsPerPackage": 30,
  "servingsConsumed": 15,
  "lastLogged": "2025-12-22T08:30:00.000Z",
  "consumptionTimes": ["07:00", "20:00"],
  "lowThreshold": 5,
  "createdAt": "2025-12-01T00:00:00.000Z"
}
```

## 🎯 Key Components

### FlatList Optimization
```typescript
<FlatList
  data={groupedSupplements}
  renderItem={renderTypeSection}
  keyExtractor={item => item.type}
  showsVerticalScrollIndicator={false}
/>
```

### State Management
- `useState` for local component state
- `useEffect` for data loading
- `useFocusEffect` for screen refresh
- AsyncStorage for persistence

### Error Handling
- Try-catch blocks on all async operations
- User-friendly Alert dialogs
- Console logging for debugging
- Graceful fallbacks

## 📖 Code Comments

All major functions include JSDoc-style comments:
```typescript
/**
 * Load all supplements from storage
 */
const loadSupplements = async () => {
  // Implementation
};
```

## 🎓 Learning Resources

- **AsyncStorage**: React Native async persistent storage
- **TypeScript**: Type-safe data models
- **FlatList**: Performant list rendering
- **Alert API**: Native confirmation dialogs
- **useFocusEffect**: React Navigation hook for screen focus

---

**Built with ❤️ for HIIT LIFE Fitness App**
**Theme: Dark Mode with Neon Green Accents**
**Ready for Expo Notifications integration**
