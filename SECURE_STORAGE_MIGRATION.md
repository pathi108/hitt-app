# Migrating from AsyncStorage to SecureStore

This guide shows how to migrate your existing storage utilities to use the new secure storage.

## Why Migrate?

**AsyncStorage** stores data in **plain text** files that can be read by:
- Rooted devices
- ADB backups
- Other apps with elevated permissions
- Malware

**SecureStore** uses:
- **Android Keystore** (hardware-backed encryption on Android)
- **iOS Keychain** (secure enclave on iOS)
- Encryption keys that never leave the device

---

## Quick Start: Update Import Statements

The new secure storage API is a drop-in replacement for AsyncStorage in most cases:

```typescript
// OLD - AsyncStorage (INSECURE)
import AsyncStorage from '@react-native-async-storage/async-storage';

const value = await AsyncStorage.getItem('key');
await AsyncStorage.setItem('key', 'value');
await AsyncStorage.removeItem('key');

// NEW - SecureStore (SECURE)
import { securelyGetItem, securelySetItem, securelyDeleteItem } from '@/src/utils/secureStorage';

const value = await securelyGetItem('key');
await securelySetItem('key', 'value');
await securelyDeleteItem('key');
```

---

## Migration Examples

### Example 1: supplementStorage.ts

**Before:**
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';

export const getAllSupplements = async (): Promise<Supplement[]> => {
  try {
    const jsonValue = await AsyncStorage.getItem(STORAGE_KEY);
    return jsonValue != null ? JSON.parse(jsonValue) : [];
  } catch (error) {
    console.error('Error loading supplements:', error);
    return [];
  }
};

export const saveSupplements = async (supplements: Supplement[]): Promise<void> => {
  try {
    const jsonValue = JSON.stringify(supplements);
    await AsyncStorage.setItem(STORAGE_KEY, jsonValue);
  } catch (error) {
    console.error('Error saving supplements:', error);
    throw error;
  }
};
```

**After:**
```typescript
import { securelyGetObject, securelySetObject } from '@/src/utils/secureStorage';

export const getAllSupplements = async (): Promise<Supplement[]> => {
  try {
    const supplements = await securelyGetObject<Supplement[]>(STORAGE_KEY);
    return supplements ?? [];
  } catch (error) {
    console.error('Error loading supplements:', error);
    return [];
  }
};

export const saveSupplements = async (supplements: Supplement[]): Promise<void> => {
  try {
    await securelySetObject(STORAGE_KEY, supplements);
  } catch (error) {
    console.error('Error saving supplements:', error);
    throw error;
  }
};
```

---

### Example 2: With Migration Logic

If you want to migrate existing AsyncStorage data to SecureStore automatically:

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { securelyGetObject, securelySetObject } from '@/src/utils/secureStorage';

const STORAGE_KEY = '@hiit_supplements';
const MIGRATION_KEY = '@hiit_supplements_migrated';

export const getAllSupplements = async (): Promise<Supplement[]> => {
  try {
    // Check if we've already migrated
    const migrated = await securelyGetItem(MIGRATION_KEY);

    if (!migrated) {
      // Try to load from old AsyncStorage
      const oldData = await AsyncStorage.getItem(STORAGE_KEY);
      if (oldData) {
        const supplements = JSON.parse(oldData);
        // Save to SecureStore
        await securelySetObject(STORAGE_KEY, supplements);
        // Mark as migrated
        await securelySetItem(MIGRATION_KEY, 'true');
        // Clean up old data
        await AsyncStorage.removeItem(STORAGE_KEY);
        console.log('✅ Migrated supplements to SecureStore');
        return supplements;
      }
    }

    // Load from SecureStore
    const supplements = await securelyGetObject<Supplement[]>(STORAGE_KEY);
    return supplements ?? [];
  } catch (error) {
    console.error('Error loading supplements:', error);
    return [];
  }
};
```

---

## API Reference

### securelyGetItem(key: string)
Get a string value from secure storage.

```typescript
const value = await securelyGetItem('myKey');
// Returns: string | null
```

### securelySetItem(key: string, value: string)
Save a string value to secure storage.

```typescript
await securelySetItem('myKey', 'myValue');
```

### securelyDeleteItem(key: string)
Delete a value from secure storage.

```typescript
await securelyDeleteItem('myKey');
```

### securelyGetObject<T>(key: string)
Get an object from secure storage (automatically parsed from JSON).

```typescript
const user = await securelyGetObject<User>('user');
// Returns: T | null
```

### securelySetObject<T>(key: string, value: T)
Save an object to secure storage (automatically stringified to JSON).

```typescript
await securelySetObject('user', { name: 'John', age: 30 });
```

---

## Files to Migrate

### High Priority (Contains Sensitive Data)
1. ✅ **src/utils/supplementStorage.ts** - Supplement tracking data
2. ✅ **src/utils/timerStorage.ts** - Timer configurations
3. ✅ **src/utils/workoutStorage.ts** - Workout data

### How to Apply Changes

For each file:

1. **Update imports:**
   ```typescript
   // Remove this:
   import AsyncStorage from '@react-native-async-storage/async-storage';

   // Add this:
   import { securelyGetObject, securelySetObject, securelyDeleteItem } from '@/src/utils/secureStorage';
   ```

2. **Replace AsyncStorage.getItem + JSON.parse:**
   ```typescript
   // Before:
   const jsonValue = await AsyncStorage.getItem(KEY);
   return jsonValue != null ? JSON.parse(jsonValue) : [];

   // After:
   return await securelyGetObject<Type[]>(KEY) ?? [];
   ```

3. **Replace JSON.stringify + AsyncStorage.setItem:**
   ```typescript
   // Before:
   const jsonValue = JSON.stringify(data);
   await AsyncStorage.setItem(KEY, jsonValue);

   // After:
   await securelySetObject(KEY, data);
   ```

4. **Replace AsyncStorage.removeItem:**
   ```typescript
   // Before:
   await AsyncStorage.removeItem(KEY);

   // After:
   await securelyDeleteItem(KEY);
   ```

---

## Testing

After migration, test that:

1. **New data saves correctly:**
   - Add new supplements/timers/workouts
   - Verify they persist after app restart

2. **Existing data still loads:**
   - If you implemented migration logic, old data should be accessible
   - If not, users will need to re-enter their data (acceptable for v1.0)

3. **Data is encrypted:**
   - Check device file system (requires root)
   - Old AsyncStorage files were plain JSON
   - New SecureStore data is encrypted

---

## Performance Considerations

SecureStore is slightly slower than AsyncStorage due to encryption/decryption:
- **Read:** ~5-10ms overhead per operation
- **Write:** ~10-20ms overhead per operation

For your use case (supplements, timers, workouts), this overhead is negligible.

---

## Rollback Plan

If you encounter issues, you can temporarily revert:

1. Keep the AsyncStorage imports alongside SecureStore
2. Try to read from SecureStore first, fall back to AsyncStorage
3. This gives you time to debug without data loss

```typescript
export const getData = async () => {
  // Try SecureStore first
  let data = await securelyGetObject(KEY);

  // Fall back to AsyncStorage if not found
  if (!data) {
    const oldData = await AsyncStorage.getItem(KEY);
    if (oldData) {
      data = JSON.parse(oldData);
    }
  }

  return data;
};
```

---

## FAQ

**Q: Will existing users lose their data?**
A: Only if you don't implement migration logic. Use the migration example above to automatically move data.

**Q: Does SecureStore work on web?**
A: The utility falls back to localStorage on web (not secure, but functional). Consider web a development-only platform.

**Q: What if the device doesn't support hardware encryption?**
A: SecureStore falls back to software encryption. Still much better than plain text AsyncStorage.

**Q: Can I use both AsyncStorage and SecureStore?**
A: Yes, you can use AsyncStorage for non-sensitive data (like UI preferences) and SecureStore for sensitive data.

---

**Next Steps:**
1. Choose whether to implement automatic migration or require fresh data entry
2. Update one storage file at a time
3. Test thoroughly
4. Deploy with release notes explaining the security improvements

---

**Last Updated:** 2026-03-19
