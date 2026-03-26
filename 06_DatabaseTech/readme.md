# Day 6: Database Technology & Data Storage for Game Programmers

## 🎯 Learning Objectives

- Database fundamentals (SQL vs NoSQL)
- Relational database design, normalization (3NF), and schemas
- SQL queries integrated with C++
- Indexing benefits and performance impact
- Local storage for game saves (JSON, binary, SQLite) in Unreal Engine
- Cloud databases with C++ integration
- Schema migration and data validation
- Multiplayer synchronization (floating-point vs fixed-point, epoch timestamps)
- Password security (salting, hashing, encryption methods)

---

## 📚 Database Fundamentals

### **What is a Database?**

**Database:** Organized collection of structured data, stored and accessed electronically.

**Why use databases instead of plain files?**

```
Plain Files:
   No efficient searching (must read entire file)
   No concurrent access (file locking issues)
   No data integrity (corrupt easily)
   No relationships between data
   No query language

Databases:
   Fast indexed searches
   Multiple users simultaneously
   ACID guarantees (Atomicity, Consistency, Isolation, Durability)
   Relationships between entities
   Powerful query languages (SQL)
```

---

### **Database Types Overview**

#### **1. Relational Databases (SQL)**

```
Structure: Tables with rows and columns

Players Table:
┌────┬──────────┬───────┬─────────┐
│ ID │   Name   │ Level │  Coins  │
├────┼──────────┼───────┼─────────┤
│ 1  │ Alice    │  25   │  5000   │
│ 2  │ Bob      │  18   │  3200   │
│ 3  │ Charlie  │  30   │  8500   │
└────┴──────────┴───────┴─────────┘

Inventory Table:
┌────┬───────────┬──────────┬──────────┐
│ ID │ PlayerID  │   Item   │ Quantity │
├────┼───────────┼──────────┼──────────┤
│ 1  │    1      │  Sword   │    1     │
│ 2  │    1      │  Potion  │   10     │
│ 3  │    2      │  Shield  │    1     │
└────┴───────────┴──────────┴──────────┘

Relationships enforced (PlayerID → Players.ID)
```

**Examples:** MySQL, PostgreSQL, SQLite, SQL Server

---

#### **2. Document Databases (NoSQL)**

```json5
// Each document stores all player data
{
  "player_id": "1",
  "name": "Alice",
  "level": 25,
  "coins": 5000,
  "inventory": [
    {"item": "Sword", "quantity": 1},
    {"item": "Potion", "quantity": 10}
  ],
  "achievements": ["first_kill", "level_10", "treasure_hunter"]
}
```

**Flexible structure** - no predefined schema required  
**Examples:** MongoDB, Firebase Firestore, Couchbase

---

### **SQL vs NoSQL Comparison**

| Aspect | SQL (Relational) | NoSQL (Document/KV) |
|--------|------------------|---------------------|
| **Schema** | Fixed, predefined | Flexible, dynamic |
| **Structure** | Tables, rows, columns | Documents, collections |
| **Relationships** | Foreign keys, joins | Embedded or referenced |
| **Scaling** | Vertical (bigger server) | Horizontal (more servers) |
| **Transactions** | Strong ACID guarantees | Eventual consistency |
| **Best For** | Complex queries, relationships | Rapid development, flexible data |

---

## 📚 Relational Databases & SQL

### **Database Schema**

**Schema:** The blueprint of a database - defines tables, columns, data types, relationships, and constraints.

```sql
-- Schema defines the structure
CREATE TABLE Players (
    PlayerID INT PRIMARY KEY,
    Username VARCHAR(50) NOT NULL,
    Level INT DEFAULT 1,
    Coins INT DEFAULT 0,
    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE Inventory (
    ItemID INT PRIMARY KEY,
    PlayerID INT,
    ItemName VARCHAR(50),
    Quantity INT,
    FOREIGN KEY (PlayerID) REFERENCES Players(PlayerID)
);
```

**Schema benefits:**
- **Data integrity** - enforces data types and constraints
- **Relationships** - foreign keys maintain referential integrity
- **Documentation** - schema IS the database structure documentation
- **Optimization** - database can optimize based on schema

**Schema changes require migration** - we'll cover this later.

---

### **Normalization and 3NF**

**Normalization:** Process of organizing data to reduce redundancy and improve integrity.

**3NF (Third Normal Form):** A database design standard that ensures:

1. **1NF (First Normal Form):** Each cell contains atomic (single) values, no repeating groups
2. **2NF (Second Normal Form):** No partial dependencies (all non-key attributes depend on entire primary key)
3. **3NF (Third Normal Form):** No transitive dependencies (non-key attributes don't depend on other non-key attributes)

#### **Example: Violating 3NF**

```sql
-- Redundant city/country data
CREATE TABLE Players (
    PlayerID INT,
    Username VARCHAR(50),
    City VARCHAR(50),
    Country VARCHAR(50)  -- Country depends on City, not PlayerID!
);

┌──────────┬──────────┬───────────┬─────────┐
│ PlayerID │ Username │   City    │ Country │
├──────────┼──────────┼───────────┼─────────┤
│    1     │  Alice   │  London   │   UK    │
│    2     │  Bob     │  Paris    │ France  │
│    3     │  Charlie │  London   │   UK    │  ← Redundant!
└──────────┴──────────┴───────────┴─────────┘

Problems:
   Waste space (UK repeated)
   Update anomaly (if London moved to France?)
   Inconsistency risk
```

#### **Solution: Proper 3NF**

```sql
-- Normalized to 3NF
CREATE TABLE Players (
    PlayerID INT PRIMARY KEY,
    Username VARCHAR(50),
    CityID INT,
    FOREIGN KEY (CityID) REFERENCES Cities(CityID)
);

CREATE TABLE Cities (
    CityID INT PRIMARY KEY,
    CityName VARCHAR(50),
    Country VARCHAR(50)
);

Players:
┌──────────┬──────────┬────────┐
│ PlayerID │ Username │ CityID │
├──────────┼──────────┼────────┤
│    1     │  Alice   │   1    │
│    2     │  Bob     │   2    │
│    3     │  Charlie │   1    │
└──────────┴──────────┴────────┘

Cities:
┌────────┬──────────┬─────────┐
│ CityID │ CityName │ Country │
├────────┼──────────┼─────────┤
│   1    │  London  │   UK    │
│   2    │  Paris   │ France  │
└────────┴──────────┴─────────┘

Benefits:
   No redundancy
   Update once (London → UK)
   Data integrity
```

---

### **Indexing: Performance Impact**

**Index:** A data structure that improves query speed by creating a fast lookup table.

#### **How Indexes Work**

```
Without Index (Full Table Scan):
┌────┬──────────┬───────┐
│ ID │   Name   │ Level │
├────┼──────────┼───────┤
│ 1  │ Alice    │  25   │  ← Check row 1
│ 2  │ Bob      │  18   │  ← Check row 2
│ 3  │ Charlie  │  30   │  ← Check row 3
│ 4  │ Diana    │  22   │  ← Check row 4
│... │   ...    │  ...  │  ← Check ALL rows!
└────┴──────────┴───────┘
Time: O(n) - must scan entire table
```

```
With Index on Name (B-Tree):
          ┌─────────┐
          │ Charlie │ → Row 3
          └────┬────┘
      ┌────────┴────────┐
┌─────▼──┐         ┌────▼──┐
│  Alice │ → Row 1 │ Diana │ → Row 4
└────────┘         └───┬───┘
                   ┌───▼──┐
                   │  Bob │ → Row 2
                   └──────┘

Time: O(log n) - binary search through tree
```

#### **Performance Benefits**

```sql
-- Create index on frequently searched column
CREATE INDEX idx_player_username ON Players(Username);

-- Query performance
SELECT * FROM Players WHERE Username = 'Alice';

Without Index: Must scan 1,000,000 rows → ~500ms
With Index:    B-tree lookup (log₂ 1,000,000 ≈ 20 comparisons) → ~2ms

Speedup: 250× faster!
```

#### **Index Trade-offs**

**Benefits:**
- **Dramatic query speedup** (250-1000× for large tables)
- **Faster joins** (indexed foreign keys)
- **Sorted results** (index maintains order)

**Costs:**
- **Storage overhead** (index = extra data structure, ~10-30% of table size)
- **Slower writes** (INSERT/UPDATE must also update index)
- **Memory usage** (indexes cached in RAM)

**Best Practice:**
- Index **frequently queried** columns (WHERE, JOIN, ORDER BY)
- Index **primary keys** (automatic in most databases)
- Index **foreign keys** (JOIN performance)
- **Don't over-index** (every index slows writes)

**Example impact:**
```
1,000,000 player table:
- Without index on Username: SELECT takes 500ms
- With index: SELECT takes 2ms (250× faster)
- INSERT cost: +0.1ms to update index (negligible)
```

---

### **SQL Queries in C++**

#### **Using SQLite with C++**

```cpp
#include <sqlite3.h>
#include <iostream>
#include <string>

class GameDatabase {
private:
    sqlite3* db;
    
public:
    GameDatabase(const char* dbPath) {
        if (sqlite3_open(dbPath, &db) != SQLITE_OK) {
            std::cerr << "Failed to open database: " << sqlite3_errmsg(db) << std::endl;
            db = nullptr;
        }
    }
    
    ~GameDatabase() {
        if (db) {
            sqlite3_close(db);
        }
    }
    
    void CreateTables() {
        const char* sql = R"(
            CREATE TABLE IF NOT EXISTS Players (
                PlayerID INTEGER PRIMARY KEY AUTOINCREMENT,
                Username TEXT NOT NULL UNIQUE,
                Level INTEGER DEFAULT 1,
                Coins INTEGER DEFAULT 0,
                CreatedAt DATETIME DEFAULT CURRENT_TIMESTAMP
            );
            
            CREATE TABLE IF NOT EXISTS Inventory (
                ItemID INTEGER PRIMARY KEY AUTOINCREMENT,
                PlayerID INTEGER,
                ItemName TEXT,
                Quantity INTEGER,
                FOREIGN KEY (PlayerID) REFERENCES Players(PlayerID)
            );
            
            CREATE INDEX IF NOT EXISTS idx_username ON Players(Username);
            CREATE INDEX IF NOT EXISTS idx_player_inventory ON Inventory(PlayerID);
        )";
        
        char* errMsg = nullptr;
        if (sqlite3_exec(db, sql, nullptr, nullptr, &errMsg) != SQLITE_OK) {
            std::cerr << "SQL error: " << errMsg << std::endl;
            sqlite3_free(errMsg);
        }
    }
    
    // INSERT player
    bool CreatePlayer(const std::string& username) {
        const char* sql = "INSERT INTO Players (Username) VALUES (?);";
        sqlite3_stmt* stmt;
        
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) != SQLITE_OK) {
            return false;
        }
        
        sqlite3_bind_text(stmt, 1, username.c_str(), -1, SQLITE_TRANSIENT);
        
        bool success = (sqlite3_step(stmt) == SQLITE_DONE);
        sqlite3_finalize(stmt);
        
        return success;
    }
    
    // SELECT player by username
    struct Player {
        int id;
        std::string username;
        int level;
        int coins;
    };
    
    Player GetPlayer(const std::string& username) {
        const char* sql = "SELECT PlayerID, Username, Level, Coins FROM Players WHERE Username = ?;";
        sqlite3_stmt* stmt;
        Player player = {-1, "", 0, 0};
        
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) != SQLITE_OK) {
            return player;
        }
        
        sqlite3_bind_text(stmt, 1, username.c_str(), -1, SQLITE_TRANSIENT);
        
        if (sqlite3_step(stmt) == SQLITE_ROW) {
            player.id = sqlite3_column_int(stmt, 0);
            player.username = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1));
            player.level = sqlite3_column_int(stmt, 2);
            player.coins = sqlite3_column_int(stmt, 3);
        }
        
        sqlite3_finalize(stmt);
        return player;
    }
    
    // UPDATE player level and coins
    bool UpdatePlayer(int playerID, int newLevel, int newCoins) {
        const char* sql = "UPDATE Players SET Level = ?, Coins = ? WHERE PlayerID = ?;";
        sqlite3_stmt* stmt;
        
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) != SQLITE_OK) {
            return false;
        }
        
        sqlite3_bind_int(stmt, 1, newLevel);
        sqlite3_bind_int(stmt, 2, newCoins);
        sqlite3_bind_int(stmt, 3, playerID);
        
        bool success = (sqlite3_step(stmt) == SQLITE_DONE);
        sqlite3_finalize(stmt);
        
        return success;
    }
    
    // ADD item to inventory
    bool AddItem(int playerID, const std::string& itemName, int quantity) {
        const char* sql = "INSERT INTO Inventory (PlayerID, ItemName, Quantity) VALUES (?, ?, ?);";
        sqlite3_stmt* stmt;
        
        if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) != SQLITE_OK) {
            return false;
        }
        
        sqlite3_bind_int(stmt, 1, playerID);
        sqlite3_bind_text(stmt, 2, itemName.c_str(), -1, SQLITE_TRANSIENT);
        sqlite3_bind_int(stmt, 3, quantity);
        
        bool success = (sqlite3_step(stmt) == SQLITE_DONE);
        sqlite3_finalize(stmt);
        
        return success;
    }
};

// Usage example
int main() {
    GameDatabase db("game.db");
    db.CreateTables();
    
    // Create player
    db.CreatePlayer("Alice");
    
    // Get player
    auto player = db.GetPlayer("Alice");
    std::cout << "Player: " << player.username 
              << ", Level: " << player.level 
              << ", Coins: " << player.coins << std::endl;
    
    // Update player
    db.UpdatePlayer(player.id, 25, 5000);
    
    // Add inventory item
    db.AddItem(player.id, "Sword", 1);
    
    return 0;
}
```

**Key C++ SQL patterns:**
- `sqlite3_prepare_v2()` - compile SQL statement
- `sqlite3_bind_*()` - bind parameters (prevents SQL injection)
- `sqlite3_step()` - execute statement
- `sqlite3_column_*()` - read results
- `sqlite3_finalize()` - cleanup

---

### **Schema Migration and Data Validation**

**Problem:** Your game is live with version 1.0 schema. Now you need to add features in version 2.0.

#### **Example: Adding Email Column**

**Version 1.0 Schema:**
```sql
CREATE TABLE Players (
    PlayerID INT PRIMARY KEY,
    Username VARCHAR(50),
    Level INT
);
```

**Version 2.0 Needs:**
```sql
-- Add Email column for password recovery
ALTER TABLE Players ADD COLUMN Email VARCHAR(100);
```

**But wait!** Existing players don't have emails. What happens?

---

#### **Migration Strategy**

**Step 1: Add column with default value**
```sql
ALTER TABLE Players ADD COLUMN Email VARCHAR(100) DEFAULT 'no-email@placeholder.com';
```

**Step 2: Validate existing data**
```cpp
void ValidateAndFixPlayerData(GameDatabase& db) {
    const char* sql = "SELECT PlayerID, Email FROM Players WHERE Email = 'no-email@placeholder.com';";
    sqlite3_stmt* stmt;
    
    if (sqlite3_prepare_v2(db.GetDB(), sql, -1, &stmt, nullptr) != SQLITE_OK) {
        return;
    }
    
    std::vector<int> playersNeedingEmail;
    while (sqlite3_step(stmt) == SQLITE_ROW) {
        int playerID = sqlite3_column_int(stmt, 0);
        playersNeedingEmail.push_back(playerID);
    }
    sqlite3_finalize(stmt);
    
    std::cout << "Found " << playersNeedingEmail.size() 
              << " players without email. They'll be prompted on next login." << std::endl;
    
    // Mark these players for email prompt
    for (int id : playersNeedingEmail) {
        const char* updateSQL = "UPDATE Players SET NeedsEmailPrompt = 1 WHERE PlayerID = ?;";
        sqlite3_stmt* updateStmt;
        sqlite3_prepare_v2(db.GetDB(), updateSQL, -1, &updateStmt, nullptr);
        sqlite3_bind_int(updateStmt, 1, id);
        sqlite3_step(updateStmt);
        sqlite3_finalize(updateStmt);
    }
}
```

**Step 3: Version tracking**
```cpp
class DatabaseMigration {
private:
    int currentVersion = 0;
    
public:
    void CheckAndMigrate(GameDatabase& db) {
        currentVersion = GetDatabaseVersion(db);
        
        if (currentVersion < 1) {
            MigrateTo1(db);
        }
        if (currentVersion < 2) {
            MigrateTo2(db);
        }
        if (currentVersion < 3) {
            MigrateTo3(db);
        }
        
        SetDatabaseVersion(db, 3);
    }
    
    void MigrateTo2(GameDatabase& db) {
        std::cout << "Migrating to version 2: Adding Email column..." << std::endl;
        
        const char* sql = R"(
            ALTER TABLE Players ADD COLUMN Email VARCHAR(100) DEFAULT 'no-email@placeholder.com';
            ALTER TABLE Players ADD COLUMN NeedsEmailPrompt INT DEFAULT 0;
        )";
        
        char* errMsg = nullptr;
        if (sqlite3_exec(db.GetDB(), sql, nullptr, nullptr, &errMsg) != SQLITE_OK) {
            std::cerr << "Migration failed: " << errMsg << std::endl;
            sqlite3_free(errMsg);
            return;
        }
        
        ValidateAndFixPlayerData(db);
        std::cout << "Migration to version 2 complete!" << std::endl;
    }
    
    int GetDatabaseVersion(GameDatabase& db) {
        // Check metadata table for version
        const char* sql = "SELECT Version FROM DatabaseMetadata LIMIT 1;";
        sqlite3_stmt* stmt;
        int version = 0;
        
        if (sqlite3_prepare_v2(db.GetDB(), sql, -1, &stmt, nullptr) == SQLITE_OK) {
            if (sqlite3_step(stmt) == SQLITE_ROW) {
                version = sqlite3_column_int(stmt, 0);
            }
        }
        sqlite3_finalize(stmt);
        
        return version;
    }
    
    void SetDatabaseVersion(GameDatabase& db, int version) {
        const char* sql = "UPDATE DatabaseMetadata SET Version = ?;";
        sqlite3_stmt* stmt;
        sqlite3_prepare_v2(db.GetDB(), sql, -1, &stmt, nullptr);
        sqlite3_bind_int(stmt, 1, version);
        sqlite3_step(stmt);
        sqlite3_finalize(stmt);
    }
};
```

---

#### **NoSQL: No Schema Migration Needed**

**NoSQL advantage:** Documents are flexible - just start writing new fields!

```cpp
// Firebase C++ SDK example
void UpdatePlayerWithEmail(const std::string& playerID, const std::string& email) {
    firebase::firestore::DocumentReference docRef = 
        db->Collection("players").Document(playerID);
    
    // Just add the field - no schema change needed!
    docRef.Update({
        {"email", firebase::firestore::FieldValue::String(email)}
    });
    
    // Old documents without email still work fine
    // New documents have email
    // No migration script needed!
}
```

**Trade-off:**
- **Flexible** - add fields anytime
- **No validation** - typos and inconsistencies possible
- **No schema documentation** - must read code to understand structure

---

## 📚 Local Storage in Unreal Engine

### **Save Game System in Unreal Engine C++**

Unreal provides the base class `USaveGame` for serializing player data to disk.

#### **Create Save Game Class**

```cpp
// PlayerSaveGame.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "PlayerSaveGame.generated.h"

UCLASS()
class YOURGAME_API UPlayerSaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    UPlayerSaveGame();

    UPROPERTY(VisibleAnywhere, Category = "Player")
    FString PlayerName;

    UPROPERTY(VisibleAnywhere, Category = "Player")
    int32 Level;

    UPROPERTY(VisibleAnywhere, Category = "Player")
    int32 Coins;

    UPROPERTY(VisibleAnywhere, Category = "Player")
    FVector PlayerLocation;

    UPROPERTY(VisibleAnywhere, Category = "Inventory")
    TArray<FString> Inventory;
};
```

#### **Save and Load System**

```cpp
// PlayerSaveManager.h
#pragma once

#include "CoreMinimal.h"
#include "Kismet/GameplayStatics.h"
#include "PlayerSaveGame.h"

class YOURGAME_API FPlayerSaveManager
{
public:
    static const FString SaveSlotName;
    static const uint32 UserIndex;

    static bool SavePlayerData(const FString& PlayerName, int32 Level, int32 Coins, 
                               const FVector& Location, const TArray<FString>& Inventory)
    {
        UPlayerSaveGame* SaveGameInstance = Cast<UPlayerSaveGame>(
            UGameplayStatics::CreateSaveGameObject(UPlayerSaveGame::StaticClass())
        );

        if (!SaveGameInstance) return false;

        // Populate save data
        SaveGameInstance->PlayerName = PlayerName;
        SaveGameInstance->Level = Level;
        SaveGameInstance->Coins = Coins;
        SaveGameInstance->PlayerLocation = Location;
        SaveGameInstance->Inventory = Inventory;

        // Save to disk
        bool bSuccess = UGameplayStatics::SaveGameToSlot(
            SaveGameInstance, 
            SaveSlotName, 
            UserIndex
        );

        if (bSuccess) {
            UE_LOG(LogTemp, Log, TEXT("Game saved successfully for %s"), *PlayerName);
        } else {
            UE_LOG(LogTemp, Error, TEXT("Failed to save game"));
        }

        return bSuccess;
    }

    static UPlayerSaveGame* LoadPlayerData()
    {
        if (UGameplayStatics::DoesSaveGameExist(SaveSlotName, UserIndex)) {
            UPlayerSaveGame* LoadedGame = Cast<UPlayerSaveGame>(
                UGameplayStatics::LoadGameFromSlot(SaveSlotName, UserIndex)
            );

            if (LoadedGame) {
                UE_LOG(LogTemp, Log, TEXT("Game loaded: %s, Level %d"), 
                       *LoadedGame->PlayerName, LoadedGame->Level);
                return LoadedGame;
            }
        }

        UE_LOG(LogTemp, Warning, TEXT("No save file found"));
        return nullptr;
    }

    static bool DeleteSaveData()
    {
        if (UGameplayStatics::DoesSaveGameExist(SaveSlotName, UserIndex)) {
            return UGameplayStatics::DeleteGameInSlot(SaveSlotName, UserIndex);
        }
        return false;
    }
};

const FString FPlayerSaveManager::SaveSlotName = TEXT("PlayerSaveSlot");
const uint32 FPlayerSaveManager::UserIndex = 0;
```

#### **Usage in Game**

```cpp
// Save player progress
void AMyPlayerController::SaveProgress()
{
    TArray<FString> CurrentInventory;
    CurrentInventory.Add(TEXT("Sword"));
    CurrentInventory.Add(TEXT("Shield"));
    CurrentInventory.Add(TEXT("Potion x10"));

    FVector CurrentLocation = GetPawn()->GetActorLocation();

    bool bSaved = FPlayerSaveManager::SavePlayerData(
        PlayerName,
        PlayerLevel,
        PlayerCoins,
        CurrentLocation,
        CurrentInventory
    );

    if (bSaved) {
        // Show "Game Saved" UI
    }
}

// Load player progress
void AMyPlayerController::LoadProgress()
{
    UPlayerSaveGame* LoadedData = FPlayerSaveManager::LoadPlayerData();

    if (LoadedData) {
        PlayerName = LoadedData->PlayerName;
        PlayerLevel = LoadedData->Level;
        PlayerCoins = LoadedData->Coins;
        
        // Teleport player to saved location
        GetPawn()->SetActorLocation(LoadedData->PlayerLocation);
        
        // Restore inventory
        for (const FString& Item : LoadedData->Inventory) {
            AddItemToInventory(Item);
        }
    } else {
        // New game - use defaults
        StartNewGame();
    }
}
```

**Save file location:**
- Windows: `%USERPROFILE%\AppData\Local\[ProjectName]\Saved\SaveGames\`
- Mac: `~/Library/Application Support/[ProjectName]/Saved/SaveGames/`
- Linux: `~/.config/[ProjectName]/Saved/SaveGames/`

---

### **JSON Serialization in C++**

For more flexible save formats, use JSON:

```cpp
// Using nlohmann/json library
#include "json.hpp"
#include <fstream>

using json = nlohmann::json;

class PlayerDataSerializer {
public:
    struct PlayerData {
        std::string name;
        int level;
        int coins;
        std::vector<std::string> inventory;
        struct { float x, y, z; } position;
    };

    static bool SaveToJSON(const PlayerData& data, const std::string& filepath) {
        json j;
        j["player"]["name"] = data.name;
        j["player"]["level"] = data.level;
        j["player"]["coins"] = data.coins;
        j["player"]["position"]["x"] = data.position.x;
        j["player"]["position"]["y"] = data.position.y;
        j["player"]["position"]["z"] = data.position.z;
        j["player"]["inventory"] = data.inventory;

        std::ofstream file(filepath);
        if (file.is_open()) {
            file << j.dump(4);  // Pretty print with 4 space indent
            file.close();
            return true;
        }
        return false;
    }

    static PlayerData LoadFromJSON(const std::string& filepath) {
        PlayerData data;
        std::ifstream file(filepath);
        
        if (file.is_open()) {
            json j;
            file >> j;
            file.close();

            data.name = j["player"]["name"];
            data.level = j["player"]["level"];
            data.coins = j["player"]["coins"];
            data.position.x = j["player"]["position"]["x"];
            data.position.y = j["player"]["position"]["y"];
            data.position.z = j["player"]["position"]["z"];
            data.inventory = j["player"]["inventory"].get<std::vector<std::string>>();
        }

        return data;
    }
};

// Usage
PlayerDataSerializer::PlayerData player;
player.name = "Alice";
player.level = 25;
player.coins = 5000;
player.inventory = {"Sword", "Shield", "Potion"};
player.position = {100.0f, 200.0f, 50.0f};

PlayerDataSerializer::SaveToJSON(player, "savegame.json");

// Result: savegame.json
/*
{
    "player": {
        "name": "Alice",
        "level": 25,
        "coins": 5000,
        "position": {
            "x": 100.0,
            "y": 200.0,
            "z": 50.0
        },
        "inventory": ["Sword", "Shield", "Potion"]
    }
}
*/
```

---

## 📚 Cloud Databases with C++ Integration

### **Firebase Firestore with C++ SDK**

Firebase provides C++ SDK for Unreal Engine integration.

#### **Setup**

1. Download Firebase C++ SDK
2. Add to Unreal project's `ThirdParty` folder
3. Update `.Build.cs`:

```csharp
// YourGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", 
    "CoreUObject", 
    "Engine", 
    "InputCore",
    "FirebaseApp",
    "FirebaseAuth",
    "FirebaseFirestore"
});

PublicIncludePaths.Add(Path.Combine(ModuleDirectory, "../ThirdParty/firebase_cpp_sdk/include"));
```

#### **Initialize Firebase**

```cpp
// FirebaseManager.h
#pragma once

#include "firebase/app.h"
#include "firebase/firestore.h"
#include "firebase/auth.h"

class FFirebaseManager {
private:
    firebase::App* app;
    firebase::firestore::Firestore* db;
    firebase::auth::Auth* auth;

public:
    FFirebaseManager() {
        // Initialize Firebase
        #if PLATFORM_ANDROID
            app = firebase::App::Create(firebase::AppOptions(), GetJavaEnv());
        #else
            app = firebase::App::Create();
        #endif

        db = firebase::firestore::Firestore::GetInstance(app);
        auth = firebase::auth::Auth::GetAuth(app);
    }

    ~FFirebaseManager() {
        delete db;
        delete auth;
        delete app;
    }

    firebase::firestore::Firestore* GetFirestore() { return db; }
    firebase::auth::Auth* GetAuth() { return auth; }
};
```

#### **Save Player Data to Cloud**

```cpp
void SavePlayerToCloud(FFirebaseManager& firebase, const std::string& playerID,
                       const std::string& username, int level, int coins)
{
    firebase::firestore::Firestore* db = firebase.GetFirestore();

    firebase::firestore::DocumentReference docRef = 
        db->Collection("players").Document(playerID);

    firebase::firestore::MapFieldValue playerData {
        {"username", firebase::firestore::FieldValue::String(username)},
        {"level", firebase::firestore::FieldValue::Integer(level)},
        {"coins", firebase::firestore::FieldValue::Integer(coins)},
        {"lastUpdated", firebase::firestore::FieldValue::ServerTimestamp()}
    };

    docRef.Set(playerData).OnCompletion([](const firebase::Future<void>& result) {
        if (result.error() == firebase::firestore::Error::kErrorOk) {
            UE_LOG(LogTemp, Log, TEXT("Player data saved to cloud"));
        } else {
            UE_LOG(LogTemp, Error, TEXT("Failed to save: %s"), 
                   UTF8_TO_TCHAR(result.error_message()));
        }
    });
}
```

#### **Load Player Data from Cloud**

```cpp
void LoadPlayerFromCloud(FFirebaseManager& firebase, const std::string& playerID,
                         std::function<void(std::string, int, int)> callback)
{
    firebase::firestore::Firestore* db = firebase.GetFirestore();

    firebase::firestore::DocumentReference docRef = 
        db->Collection("players").Document(playerID);

    docRef.Get().OnCompletion([callback](
        const firebase::Future<firebase::firestore::DocumentSnapshot>& result)
    {
        if (result.error() == firebase::firestore::Error::kErrorOk) {
            const firebase::firestore::DocumentSnapshot* snapshot = result.result();
            
            if (snapshot->exists()) {
                std::string username = snapshot->Get("username").string_value();
                int level = snapshot->Get("level").integer_value();
                int coins = snapshot->Get("coins").integer_value();

                callback(username, level, coins);
                
                UE_LOG(LogTemp, Log, TEXT("Loaded player: %s, Level %d"), 
                       UTF8_TO_TCHAR(username.c_str()), level);
            } else {
                UE_LOG(LogTemp, Warning, TEXT("Player document doesn't exist"));
            }
        } else {
            UE_LOG(LogTemp, Error, TEXT("Failed to load: %s"), 
                   UTF8_TO_TCHAR(result.error_message()));
        }
    });
}
```

#### **Realtime Updates**

```cpp
void ListenToPlayerUpdates(FFirebaseManager& firebase, const std::string& playerID) {
    firebase::firestore::Firestore* db = firebase.GetFirestore();

    firebase::firestore::DocumentReference docRef = 
        db->Collection("players").Document(playerID);

    // Listen for realtime changes
    docRef.AddSnapshotListener([](
        const firebase::firestore::DocumentSnapshot& snapshot,
        firebase::firestore::Error error,
        const std::string& errorMsg)
    {
        if (error == firebase::firestore::Error::kErrorOk) {
            if (snapshot.exists()) {
                int newLevel = snapshot.Get("level").integer_value();
                int newCoins = snapshot.Get("coins").integer_value();

                UE_LOG(LogTemp, Log, TEXT("Player updated: Level %d, Coins %d"),
                       newLevel, newCoins);
                
                // Update local game state
                UpdateLocalPlayerData(newLevel, newCoins);
            }
        }
    });
}
```

---

## 📚 Multiplayer Game Data

### **Floating-Point vs Fixed-Point Arithmetic**

**Problem:** Multiplayer games must synchronize state across clients. Floating-point has determinism issues.

#### **Floating-Point Issues**

```cpp
// Different CPUs/compilers may produce slightly different results
float playerPosX_Client1 = 100.0f + 0.1f * deltaTime;  // = 100.016
float playerPosX_Client2 = 100.0f + 0.1f * deltaTime;  // = 100.016001 (!)

// After 1000 frames:
// Client 1: 116.0
// Client 2: 116.001  ← DESYNC!
```

**Why this happens:**
- Rounding errors accumulate
- Different FPU implementations
- Compiler optimizations vary
- Platform differences (x86 vs ARM)

---

#### **Fixed-Point Solution**

```cpp
// Fixed-point: Store as integer, interpret with decimal place
// 16.16 fixed-point: 16 bits integer, 16 bits fractional

class Fixed {
private:
    int32_t value;  // Internal representation
    static constexpr int32_t SHIFT = 16;
    static constexpr int32_t ONE = 1 << SHIFT;  // 65536 represents 1.0

public:
    Fixed() : value(0) {}
    Fixed(int32_t i) : value(i << SHIFT) {}
    Fixed(float f) : value(static_cast<int32_t>(f * ONE)) {}

    float ToFloat() const { return static_cast<float>(value) / ONE; }
    int32_t ToInt() const { return value >> SHIFT; }

    // Arithmetic operations
    Fixed operator+(const Fixed& other) const {
        Fixed result;
        result.value = value + other.value;
        return result;
    }

    Fixed operator*(const Fixed& other) const {
        Fixed result;
        result.value = (static_cast<int64_t>(value) * other.value) >> SHIFT;
        return result;
    }

    Fixed operator/(const Fixed& other) const {
        Fixed result;
        result.value = (static_cast<int64_t>(value) << SHIFT) / other.value;
        return result;
    }

    bool operator==(const Fixed& other) const {
        return value == other.value;  // EXACT equality!
    }
};

// Deterministic multiplayer position
Fixed playerPosX(100);          // 100.0
Fixed deltaTime(0.016f);         // 0.016
Fixed velocity(10);              // 10.0

for (int frame = 0; frame < 1000; frame++) {
    playerPosX = playerPosX + velocity * deltaTime;
}

// GUARANTEED same result on ALL clients!
// No floating-point drift
```

**Benefits:**
- **Deterministic** - same input = same output on all platforms
- **Exact equality** - no epsilon comparisons
- **Reproducible** - replays work perfectly

**Trade-offs:**
- **Limited range** - 16.16 = -32768 to +32767
- **Limited precision** - 1/65536 ≈ 0.000015
- **Slower** - integer ops instead of FPU (but deterministic!)

---

### **Epoch Timestamps for Client-Server Validation**

**Epoch Time:** Seconds since January 1, 1970 00:00:00 UTC (Unix timestamp).

**Why use epoch timestamps?**
- **Universal** - same value worldwide
- **Monotonic** - always increases
- **Compact** - single int64
- **Easy comparison** - simple arithmetic

#### **Server Timestamp Validation**

```cpp
#include <chrono>
#include <ctime>

class TimestampValidator {
public:
    // Get current epoch timestamp (milliseconds)
    static int64_t GetCurrentEpochMs() {
        auto now = std::chrono::system_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            now.time_since_epoch()
        );
        return ms.count();
    }

    // Validate client timestamp is within acceptable range
    static bool ValidateClientTimestamp(int64_t clientTimestamp, 
                                        int64_t maxDriftMs = 5000)
    {
        int64_t serverTime = GetCurrentEpochMs();
        int64_t diff = std::abs(serverTime - clientTimestamp);

        if (diff > maxDriftMs) {
            UE_LOG(LogTemp, Warning, 
                   TEXT("Client clock drift: %lld ms (max allowed: %lld ms)"),
                   diff, maxDriftMs);
            return false;
        }

        return true;
    }

    // Prevent replay attacks
    struct PlayerAction {
        int64_t timestamp;
        std::string action;
        std::string data;
    };

    static bool ValidateAction(const PlayerAction& action, 
                               int64_t lastProcessedTimestamp)
    {
        // Reject old timestamps (replay attack)
        if (action.timestamp <= lastProcessedTimestamp) {
            UE_LOG(LogTemp, Warning, 
                   TEXT("Rejected old timestamp: %lld <= %lld (replay attack?)"),
                   action.timestamp, lastProcessedTimestamp);
            return false;
        }

        // Reject future timestamps (clock manipulation)
        int64_t serverTime = GetCurrentEpochMs();
        if (action.timestamp > serverTime + 1000) {  // 1 second tolerance
            UE_LOG(LogTemp, Warning, 
                   TEXT("Rejected future timestamp: %lld > %lld (clock cheating?)"),
                   action.timestamp, serverTime);
            return false;
        }

        return true;
    }
};

// Usage in server
class GameServer {
private:
    std::map<std::string, int64_t> playerLastActionTime;

public:
    void ProcessPlayerAction(const std::string& playerID, 
                             const TimestampValidator::PlayerAction& action)
    {
        // Validate client timestamp isn't too far off
        if (!TimestampValidator::ValidateClientTimestamp(action.timestamp)) {
            SendErrorToClient(playerID, "Clock sync error");
            return;
        }

        // Prevent replay attacks
        int64_t lastTimestamp = playerLastActionTime[playerID];
        if (!TimestampValidator::ValidateAction(action, lastTimestamp)) {
            SendErrorToClient(playerID, "Invalid action timestamp");
            BanPlayer(playerID);  // Possible cheating
            return;
        }

        // Update last action time
        playerLastActionTime[playerID] = action.timestamp;

        // Process action
        ExecuteAction(playerID, action);
    }
};
```

#### **Server Authoritative Time**

```cpp
// Client sends: "I moved to (100, 200) at time 1234567890"
// Server validates: "Your clock says 1234567890, mine says 1234567892 (2ms diff - OK)"
// Server responds: "Confirmed. Server time is 1234567895"

struct ServerResponse {
    bool success;
    int64_t serverTimestamp;  // Client synchronizes to this
    std::string message;
};

ServerResponse ProcessMove(int64_t clientTimestamp, float x, float y) {
    int64_t serverTime = TimestampValidator::GetCurrentEpochMs();
    int64_t latency = serverTime - clientTimestamp;

    ServerResponse response;
    response.serverTimestamp = serverTime;

    if (latency > 500) {  // 500ms max latency
        response.success = false;
        response.message = "High latency detected";
        return response;
    }

    // Accept move
    UpdatePlayerPosition(x, y);
    response.success = true;
    response.message = "Move confirmed";
    return response;
}
```

---

### **Conflict Resolution in Multiplayer**

**Problem:** Two clients modify same data simultaneously.

#### **Last-Write-Wins (Simple)**

```cpp
struct PlayerState {
    int health;
    int64_t lastUpdateTime;
};

std::map<std::string, PlayerState> players;

void UpdatePlayerHealth(const std::string& playerID, int newHealth, int64_t timestamp) {
    auto& player = players[playerID];

    if (timestamp > player.lastUpdateTime) {
        player.health = newHealth;
        player.lastUpdateTime = timestamp;
        UE_LOG(LogTemp, Log, TEXT("Updated health: %d at %lld"), newHealth, timestamp);
    } else {
        UE_LOG(LogTemp, Warning, TEXT("Ignored stale update: %lld <= %lld"),
               timestamp, player.lastUpdateTime);
    }
}
```

#### **Server Authoritative (Best for Games)**

```cpp
// Client sends INPUT, not state
struct PlayerInput {
    int64_t timestamp;
    float moveX;
    float moveY;
    bool jump;
    bool shoot;
};

class ServerAuthority {
private:
    std::map<std::string, PlayerState> authoritative_state;

public:
    void ProcessInput(const std::string& playerID, const PlayerInput& input) {
        // Server is the ONLY source of truth
        auto& player = authoritative_state[playerID];

        // Apply input to server state
        player.position.x += input.moveX * deltaTime;
        player.position.y += input.moveY * deltaTime;

        if (input.jump && player.onGround) {
            player.velocity.z = 10.0f;
        }

        if (input.shoot && player.canShoot) {
            SpawnProjectile(player.position, player.aim);
            player.canShoot = false;
            player.shootCooldown = 0.5f;
        }

        // Send authoritative state back to ALL clients
        BroadcastPlayerState(playerID, player);
    }
};

// Clients display server state, predict locally for smoothness
```

---

## 📚  Data Privacy & Security

### **Password Security: Hashing and Salting**

**NEVER store plaintext passwords!**

#### **What is Salting?**

**Salt:** Random data added to password before hashing to prevent rainbow table attacks.

**Without salt:**
```
Password: "password123"
Hash:     "482c811da5d5b4bc6d497ffa98491e38"  ← Same for everyone!

Attacker's rainbow table:
"password123" → "482c811da5d5b4bc6d497ffa98491e38"
"admin"       → "21232f297a57a5a743894a0e4a801fc3"
...

Instant crack!
```

**With salt:**
```
Password: "password123"
Salt:     "a3f8k9m2p5"  ← Random, unique per user
Combined: "password123a3f8k9m2p5"
Hash:     "7d9c4f2a1b8e5f6c3a9d8e7b4f2a1c5d"  ← Unique!

Attacker's rainbow table useless - every user has different hash!
```

---

#### **Secure Password Hashing in C++**

```cpp
#include <openssl/sha.h>
#include <openssl/rand.h>
#include <sstream>
#include <iomanip>
#include <cstring>

class PasswordSecurity {
public:
    // Generate random salt
    static std::string GenerateSalt(size_t length = 16) {
        unsigned char salt[16];
        RAND_bytes(salt, length);

        std::stringstream ss;
        for (size_t i = 0; i < length; i++) {
            ss << std::hex << std::setw(2) << std::setfill('0') 
               << static_cast<int>(salt[i]);
        }

        return ss.str();
    }

    // Hash password with salt using SHA-256
    static std::string HashPassword(const std::string& password, 
                                    const std::string& salt)
    {
        std::string combined = password + salt;
        unsigned char hash[SHA256_DIGEST_LENGTH];

        SHA256(reinterpret_cast<const unsigned char*>(combined.c_str()),
               combined.length(), hash);

        std::stringstream ss;
        for (int i = 0; i < SHA256_DIGEST_LENGTH; i++) {
            ss << std::hex << std::setw(2) << std::setfill('0') 
               << static_cast<int>(hash[i]);
        }

        return ss.str();
    }

    // Store user (for database)
    struct UserCredentials {
        std::string username;
        std::string passwordHash;
        std::string salt;
    };

    static UserCredentials CreateUser(const std::string& username, 
                                      const std::string& password)
    {
        UserCredentials creds;
        creds.username = username;
        creds.salt = GenerateSalt();
        creds.passwordHash = HashPassword(password, creds.salt);

        UE_LOG(LogTemp, Log, TEXT("Created user: %s"), 
               UTF8_TO_TCHAR(username.c_str()));
        UE_LOG(LogTemp, Log, TEXT("Salt: %s"), 
               UTF8_TO_TCHAR(creds.salt.c_str()));
        UE_LOG(LogTemp, Log, TEXT("Hash: %s"), 
               UTF8_TO_TCHAR(creds.passwordHash.c_str()));

        return creds;
    }

    // Verify password
    static bool VerifyPassword(const std::string& inputPassword,
                              const UserCredentials& stored)
    {
        std::string inputHash = HashPassword(inputPassword, stored.salt);
        return inputHash == stored.passwordHash;
    }
};

// Usage
void RegisterUser(const std::string& username, const std::string& password) {
    auto creds = PasswordSecurity::CreateUser(username, password);

    // Store in database
    const char* sql = "INSERT INTO Users (Username, PasswordHash, Salt) VALUES (?, ?, ?);";
    // ... bind and execute SQL
}

void LoginUser(const std::string& username, const std::string& password) {
    // Load from database
    UserCredentials stored = LoadUserCredentials(username);

    if (PasswordSecurity::VerifyPassword(password, stored)) {
        UE_LOG(LogTemp, Log, TEXT("Login successful!"));
        StartGameSession(username);
    } else {
        UE_LOG(LogTemp, Warning, TEXT("Invalid password"));
        ShowLoginError();
    }
}
```

---

### **Encryption Methods Overview**

**Encryption:** Converting data into unreadable format that can only be decrypted with a key.

#### **Common Encryption Algorithms**

**Symmetric Encryption** (same key for encrypt/decrypt):
- **AES-256** (Advanced Encryption Standard)
   - Industry standard, very secure
   - Fast (hardware accelerated)
   - Best for: File encryption, database encryption
   - Example: Encrypt save files so players can't cheat

- **ChaCha20**
   - Fast in software (mobile-friendly)
   - Used by Google, Cloudflare
   - Best for: Streaming data, mobile games

**Asymmetric Encryption** (public key encrypts, private key decrypts):
- **RSA**
   - Secure key exchange
   - Slow for large data
   - Best for: Sending encryption keys, digital signatures

- **Elliptic Curve Cryptography (ECC)**
   - Smaller keys than RSA (256-bit ECC = 3072-bit RSA security)
   - Faster than RSA
   - Best for: Mobile, IoT, modern systems

**Hashing** (one-way, can't decrypt):
- **SHA-256**
   - Secure, widely used
   - Best for: Passwords (with salt), file integrity

- **bcrypt / Argon2**
   - Specifically designed for passwords
   - Configurable difficulty (slow = harder to crack)
   - Best for: Password storage (better than SHA-256!)

---

#### **When to Use Each**

```cpp
// Password storage: bcrypt/Argon2 (or SHA-256 with salt)
std::string passwordHash = bcrypt::hashpw(password, bcrypt::gensalt());

// File encryption: AES-256
// (prevents players from editing save files)
AES256::Encrypt(saveFileData, secretKey);

// Network communication: RSA + AES
// 1. Exchange AES key using RSA
// 2. Encrypt all data with AES (faster)
RSAKeyPair keys = RSA::GenerateKeys();
SendPublicKey(keys.publicKey);
std::string aesKey = GenerateAESKey();
std::string encryptedKey = RSA::Encrypt(aesKey, recipientPublicKey);
SendEncrypted(AES::Encrypt(gameData, aesKey));

// File integrity check: SHA-256
std::string fileHash = SHA256::Hash(fileContents);
// Detect if file was tampered with
```

---

# 📚 Exercises

## Assignment 1: RPG Inventory System with SQLite

**Objective:** Build a Diablo-style loot system with persistent storage and visual inventory grid.

**Theme:** You're creating the backend for an action RPG where players collect randomized loot, manage inventory, and track legendary items.

---

### Game Features

**Player Management:**
- Create heroes (Warrior, Mage, Rogue classes)
- Track level, experience, gold
- Persistent save/load across sessions

**Inventory System:**
- Items: Weapons, Armor, Potions
- Rarity: Common (gray), Rare (blue), Legendary (orange)
- Stats: Attack, Defense, Health bonuses
- Legendary items are unique (only one per player)

**Loot Generation:**
- Random drops with rarity chances based on player level
- Early game: 85% common, 14% rare, 1% legendary
- Late game: 40% common, 50% rare, 10% legendary

**Raylib Visualization:**
- Player stats display
- 8×4 inventory grid
- Color-coded items by rarity
- Hover to see item stats

---

### Requirements

**1. Database Schema - 3NF**

```sql
-- Players table
CREATE TABLE Players (
    PlayerID INTEGER PRIMARY KEY AUTOINCREMENT,
    HeroName TEXT NOT NULL UNIQUE,
    Class TEXT NOT NULL CHECK(Class IN ('Warrior', 'Mage', 'Rogue')),
    Level INTEGER DEFAULT 1 CHECK(Level >= 1 AND Level <= 99),
    Experience INTEGER DEFAULT 0,
    Gold INTEGER DEFAULT 100,
    CreatedAt DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Item catalog (all possible items in game)
CREATE TABLE ItemCatalog (
    ItemID INTEGER PRIMARY KEY AUTOINCREMENT,
    ItemName TEXT NOT NULL UNIQUE,
    ItemType TEXT NOT NULL CHECK(ItemType IN ('Weapon', 'Armor', 'Potion')),
    Rarity TEXT NOT NULL CHECK(Rarity IN ('Common', 'Rare', 'Legendary')),
    AttackBonus INTEGER DEFAULT 0,
    DefenseBonus INTEGER DEFAULT 0,
    HealthBonus INTEGER DEFAULT 0
);

-- Player inventory (which items each player owns)
CREATE TABLE Inventory (
    InventoryID INTEGER PRIMARY KEY AUTOINCREMENT,
    PlayerID INTEGER NOT NULL,
    ItemID INTEGER NOT NULL,
    Quantity INTEGER DEFAULT 1 CHECK(Quantity > 0),
    AcquiredAt DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (PlayerID) REFERENCES Players(PlayerID) ON DELETE CASCADE,
    FOREIGN KEY (ItemID) REFERENCES ItemCatalog(ItemID),
    UNIQUE(PlayerID, ItemID)  -- Can't have duplicate items
);

-- Performance indexes
CREATE INDEX idx_player_name ON Players(HeroName);
CREATE INDEX idx_player_inventory ON Inventory(PlayerID);
CREATE INDEX idx_item_rarity ON ItemCatalog(Rarity);
CREATE INDEX idx_item_type ON ItemCatalog(ItemType);
```

**Why 3NF?**
- No redundancy: Item stats stored once in ItemCatalog, referenced by Inventory
- No transitive dependencies: Inventory references both Player and Item via foreign keys
- Data integrity: If item stats change, all players see update automatically

---

**2. Core Database Operations**

Implement `RPGDatabase.h/.cpp`:

```cpp
class RPGDatabase {
private:
    sqlite3* db;

public:
    RPGDatabase(const char* dbPath);
    ~RPGDatabase();
    
    // Schema management
    void CreateTables();
    void PopulateItemCatalog();  // Pre-fill with game items
    
    // Hero management
    struct Hero {
        int id;
        std::string name;
        std::string heroClass;
        int level;
        int experience;
        int gold;
    };
    
    bool CreateHero(const std::string& name, const std::string& heroClass);
    Hero GetHero(const std::string& name);
    bool UpdateHeroStats(int playerID, int level, int exp, int gold);
    bool DeleteHero(int playerID);
    bool HeroExists(const std::string& name);
    
    // Item management
    struct Item {
        int itemID;
        std::string name;
        std::string type;
        std::string rarity;
        int attack;
        int defense;
        int health;
    };
    
    std::vector<Item> GetAllItems();
    Item GetItemByID(int itemID);
    
    // Inventory management
    bool AddItemToInventory(int playerID, int itemID);
    bool RemoveItemFromInventory(int playerID, int itemID);
    std::vector<Item> GetPlayerInventory(int playerID);
    std::vector<Item> GetItemsByRarity(int playerID, const std::string& rarity);
    std::vector<Item> GetItemsByType(int playerID, const std::string& type);
    int CountItems(int playerID);
    int CountLegendaryItems(int playerID);
    bool HasItem(int playerID, int itemID);
    
    // Loot generation
    Item GenerateRandomLoot(int playerLevel);
};
```

---

**3. Loot Generation System**

```cpp
Item RPGDatabase::GenerateRandomLoot(int playerLevel) {
    // Calculate rarity based on level
    int roll = rand() % 100;
    std::string rarity;
    
    if (playerLevel < 5) {
        // Levels 1-4: Mostly common
        if (roll < 85) rarity = "Common";
        else if (roll < 99) rarity = "Rare";
        else rarity = "Legendary";
    } else if (playerLevel < 10) {
        // Levels 5-9: More rare items
        if (roll < 60) rarity = "Common";
        else if (roll < 95) rarity = "Rare";
        else rarity = "Legendary";
    } else {
        // Level 10+: Best loot
        if (roll < 40) rarity = "Common";
        else if (roll < 90) rarity = "Rare";
        else rarity = "Legendary";
    }
    
    // Query random item of chosen rarity
    const char* sql = R"(
        SELECT ItemID, ItemName, ItemType, Rarity, AttackBonus, DefenseBonus, HealthBonus
        FROM ItemCatalog
        WHERE Rarity = ?
        ORDER BY RANDOM()
        LIMIT 1;
    )";
    
    // TODO: Execute query and return Item
}
```

---

**4. Raylib Visualization**

```cpp
void DrawInventoryGrid(const std::vector<RPGDatabase::Item>& items, int selectedIndex) {
    const int gridSize = 64;
    const int gridCols = 8;
    const int gridRows = 4;
    const int startX = 50;
    const int startY = 150;
    
    // Draw grid background
    for (int row = 0; row < gridRows; row++) {
        for (int col = 0; col < gridCols; col++) {
            int x = startX + col * (gridSize + 10);
            int y = startY + row * (gridSize + 10);
            DrawRectangle(x, y, gridSize, gridSize, DARKGRAY);
        }
    }
    
    // Draw items
    for (size_t i = 0; i < items.size() && i < 32; i++) {
        int col = i % gridCols;
        int row = i / gridCols;
        int x = startX + col * (gridSize + 10);
        int y = startY + row * (gridSize + 10);
        
        // Border color by rarity
        Color borderColor;
        if (items[i].rarity == "Common") borderColor = GRAY;
        else if (items[i].rarity == "Rare") borderColor = BLUE;
        else borderColor = ORANGE;  // Legendary
        
        // Highlight if selected
        if ((int)i == selectedIndex) {
            DrawRectangle(x - 2, y - 2, gridSize + 4, gridSize + 4, YELLOW);
        }
        
        DrawRectangle(x, y, gridSize, gridSize, DARKGRAY);
        DrawRectangleLinesEx({(float)x, (float)y, (float)gridSize, (float)gridSize}, 
                            3, borderColor);
        
        // Item name (truncated)
        std::string shortName = items[i].name.substr(0, 8);
        DrawText(shortName.c_str(), x + 4, y + 4, 10, WHITE);
        
        // Stats
        int statY = y + 20;
        if (items[i].attack > 0) {
            DrawText(TextFormat("+%d", items[i].attack), x + 4, statY, 10, RED);
            statY += 12;
        }
        if (items[i].defense > 0) {
            DrawText(TextFormat("+%d", items[i].defense), x + 4, statY, 10, SKYBLUE);
            statY += 12;
        }
        if (items[i].health > 0) {
            DrawText(TextFormat("+%d", items[i].health), x + 4, statY, 10, GREEN);
        }
    }
}

void DrawItemTooltip(const RPGDatabase::Item& item, int mouseX, int mouseY) {
    // Tooltip background
    DrawRectangle(mouseX + 10, mouseY, 200, 80, Fade(BLACK, 0.9f));
    DrawRectangleLines(mouseX + 10, mouseY, 200, 80, 
                      item.rarity == "Legendary" ? ORANGE : 
                      item.rarity == "Rare" ? BLUE : GRAY);
    
    // Item details
    Color nameColor = item.rarity == "Legendary" ? ORANGE :
                     item.rarity == "Rare" ? BLUE : LIGHTGRAY;
    DrawText(item.name.c_str(), mouseX + 15, mouseY + 5, 16, nameColor);
    DrawText(item.type.c_str(), mouseX + 15, mouseY + 25, 12, DARKGRAY);
    
    if (item.attack > 0)
        DrawText(TextFormat("Attack: +%d", item.attack), 
                mouseX + 15, mouseY + 40, 12, RED);
    if (item.defense > 0)
        DrawText(TextFormat("Defense: +%d", item.defense), 
                mouseX + 15, mouseY + 55, 12, SKYBLUE);
    if (item.health > 0)
        DrawText(TextFormat("Health: +%d", item.health), 
                mouseX + 15, mouseY + 70, 12, GREEN);
}
```

---

### Sample Item Catalog

Pre-populate database with these items (in `PopulateItemCatalog()`):

**Weapons (Common):**
- Rusty Sword (+5 ATK)
- Wooden Staff (+4 ATK)
- Iron Dagger (+6 ATK)
- Old Bow (+5 ATK)

**Weapons (Rare):**
- Steel Longsword (+25 ATK)
- Crystal Staff (+20 ATK)
- Elven Blade (+22 ATK)
- Enchanted Bow (+24 ATK)

**Weapons (Legendary):**
- Dragonbane (+60 ATK)
- Staff of Eternity (+55 ATK)
- Shadowfang (+58 ATK)
- Sunfury Bow (+56 ATK)

**Armor (Common):**
- Leather Vest (+8 DEF)
- Cloth Robe (+6 DEF)
- Wooden Shield (+7 DEF)

**Armor (Rare):**
- Chainmail (+20 DEF)
- Mage's Vestments (+18 DEF)
- Steel Shield (+22 DEF)

**Armor (Legendary):**
- Plate of the Titans (+50 DEF)
- Archmage Robes (+45 DEF)
- Aegis Shield (+48 DEF)

**Potions:**
- Minor Health Potion (Common, +20 HP)
- Greater Health Potion (Rare, +50 HP)
- Elixir of Life (Legendary, +150 HP)

---

### Complete Example Code

**main.cpp:**
```cpp
#include <raylib.h>
#include "RPGDatabase.h"
#include <iostream>
#include <cstdlib>
#include <ctime>

int main() {
    srand((unsigned int)time(nullptr));
    
    // Initialize database
    RPGDatabase db("rpg_inventory.db");
    db.CreateTables();
    db.PopulateItemCatalog();
    
    // Create or load hero
    std::string heroName = "Aragorn";
    if (!db.HeroExists(heroName)) {
        db.CreateHero(heroName, "Warrior");
        std::cout << "Created new hero: " << heroName << std::endl;
        
        // Start with some basic items
        db.AddItemToInventory(1, 1);  // Rusty Sword
        db.AddItemToInventory(1, 9);  // Leather Vest
    }
    
    auto hero = db.GetHero(heroName);
    auto inventory = db.GetPlayerInventory(hero.id);
    
    // Initialize Raylib
    InitWindow(800, 600, "RPG Inventory System");
    SetTargetFPS(60);
    
    int selectedItem = 0;
    bool showTooltip = false;
    RPGDatabase::Item hoveredItem;
    
    while (!WindowShouldClose()) {
        // Input: Generate loot
        if (IsKeyPressed(KEY_L)) {
            auto loot = db.GenerateRandomLoot(hero.level);
            
            if (loot.rarity == "Legendary" && db.HasItem(hero.id, loot.itemID)) {
                std::cout << "Already have legendary item: " << loot.name << std::endl;
            } else {
                db.AddItemToInventory(hero.id, loot.itemID);
                inventory = db.GetPlayerInventory(hero.id);
                std::cout << "Looted: " << loot.name 
                         << " (" << loot.rarity << ")" << std::endl;
            }
        }
        
        // Input: Filter by rarity
        if (IsKeyPressed(KEY_R)) {
            inventory = db.GetItemsByRarity(hero.id, "Rare");
            std::cout << "Showing Rare items only" << std::endl;
        }
        if (IsKeyPressed(KEY_E)) {
            inventory = db.GetItemsByRarity(hero.id, "Legendary");
            std::cout << "Showing Legendary items only" << std::endl;
        }
        if (IsKeyPressed(KEY_A)) {
            inventory = db.GetPlayerInventory(hero.id);
            std::cout << "Showing all items" << std::endl;
        }
        
        // Input: Navigation
        if (IsKeyPressed(KEY_RIGHT)) selectedItem = (selectedItem + 1) % inventory.size();
        if (IsKeyPressed(KEY_LEFT)) selectedItem = (selectedItem - 1 + inventory.size()) % inventory.size();
        
        // Input: Delete selected item
        if (IsKeyPressed(KEY_DELETE) && !inventory.empty()) {
            db.RemoveItemFromInventory(hero.id, inventory[selectedItem].itemID);
            inventory = db.GetPlayerInventory(hero.id);
            if (selectedItem >= (int)inventory.size()) selectedItem = inventory.size() - 1;
            if (selectedItem < 0) selectedItem = 0;
            std::cout << "Item removed" << std::endl;
        }
        
        // Mouse hover detection
        Vector2 mousePos = GetMousePosition();
        showTooltip = false;
        
        const int gridSize = 64;
        const int gridCols = 8;
        const int startX = 50;
        const int startY = 150;
        
        for (size_t i = 0; i < inventory.size(); i++) {
            int col = i % gridCols;
            int row = i / gridCols;
            int x = startX + col * (gridSize + 10);
            int y = startY + row * (gridSize + 10);
            
            if (mousePos.x >= x && mousePos.x <= x + gridSize &&
                mousePos.y >= y && mousePos.y <= y + gridSize) {
                showTooltip = true;
                hoveredItem = inventory[i];
                break;
            }
        }
        
        // Draw
        BeginDrawing();
            ClearBackground(BLACK);
            
            // Hero stats
            DrawText(TextFormat("%s the %s", hero.name.c_str(), hero.heroClass.c_str()),
                    20, 20, 24, WHITE);
            DrawText(TextFormat("Level: %d", hero.level), 20, 50, 18, LIGHTGRAY);
            DrawText(TextFormat("Gold: %dg", hero.gold), 20, 75, 18, GOLD);
            DrawText(TextFormat("Items: %d/32", (int)inventory.size()), 20, 100, 18, SKYBLUE);
            
            // Controls
            DrawText("[L] Loot  [R] Rare  [E] Legendary  [A] All  [DEL] Remove", 
                    20, 125, 14, DARKGRAY);
            
            // Inventory grid
            DrawInventoryGrid(inventory, selectedItem);
            
            // Tooltip
            if (showTooltip) {
                DrawItemTooltip(hoveredItem, (int)mousePos.x, (int)mousePos.y);
            }
            
        EndDrawing();
    }
    
    CloseWindow();
    return 0;
}
```

---

## Assignment 2: Turn-Based Combat Simulator

**Objective:** Create a multiplayer-ready combat system using fixed-point math and timestamp validation.

**Theme:** Two fighters duel in turn-based combat. The server validates all actions with timestamps to prevent cheating, and uses fixed-point math for deterministic damage.

---

### Game Features

**Combat Mechanics:**
- Two fighters (Player vs Enemy)
- Health, Attack, Defense stats
- Actions: Attack, Defend (+50% DEF), Use Potion
- Damage formula with random variance (deterministic!)

**Anti-Cheat System:**
- All actions include timestamp
- Server validates timestamps (not too old, not too new)
- Replay attack detection (can't reuse old actions)
- Clock manipulation detection (future timestamps rejected)

**Raylib Visualization:**
- Both fighters with animated health bars
- Action log showing combat history
- Turn indicator
- "CHEATER DETECTED" warning on invalid timestamp

---

### Requirements

**1. Fixed-Point Math**

Implement 16.16 fixed-point arithmetic:

```cpp
// Fixed.h
class Fixed {
private:
    int32_t value;  // Internal: 16 bits integer, 16 bits fractional
    static constexpr int32_t SHIFT = 16;
    static constexpr int32_t ONE = 1 << SHIFT;  // 65536 = 1.0

public:
    Fixed() : value(0) {}
    Fixed(int32_t i) : value(i << SHIFT) {}
    Fixed(float f) : value(static_cast<int32_t>(f * ONE)) {}
    
    float ToFloat() const { return (float)value / ONE; }
    int32_t ToInt() const { return value >> SHIFT; }
    
    // Arithmetic (must be deterministic!)
    Fixed operator+(const Fixed& other) const;
    Fixed operator-(const Fixed& other) const;
    Fixed operator*(const Fixed& other) const;
    Fixed operator/(const Fixed& other) const;
    
    // Comparison
    bool operator<(const Fixed& other) const;
    bool operator>(const Fixed& other) const;
    bool operator==(const Fixed& other) const;
    bool operator<=(const Fixed& other) const;
    bool operator>=(const Fixed& other) const;
};

// Damage formula (deterministic)
Fixed CalculateDamage(Fixed attackPower, Fixed defense, Fixed variance) {
    // Base damage = attack - (defense / 2)
    Fixed baseDamage = attackPower - (defense / Fixed(2));
    
    // Clamp to minimum 1
    if (baseDamage < Fixed(1)) {
        baseDamage = Fixed(1);
    }
    
    // Apply variance: 0.9× to 1.1× (±10%)
    Fixed minMult(0.9f);
    Fixed maxMult(1.1f);
    Fixed range = maxMult - minMult;
    Fixed multiplier = minMult + range * variance;
    
    Fixed finalDamage = baseDamage * multiplier;
    
    return finalDamage;
}
```

**Why fixed-point?**
- Floating-point operations are non-deterministic across platforms
- `100.0f + 0.1f` might give slightly different results on different CPUs
- Fixed-point: `100 + 0.1` always gives EXACT same result
- Essential for multiplayer - both clients calculate same damage

---

**2. Timestamp Validation**

```cpp
// TimestampValidator.h
class TimestampValidator {
public:
    // Get current time in milliseconds since epoch
    static int64_t GetCurrentEpochMs() {
        auto now = std::chrono::system_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            now.time_since_epoch()
        );
        return ms.count();
    }
    
    // Validate timestamp is within acceptable range (±5 seconds)
    static bool ValidateTimestamp(int64_t clientTimestamp, 
                                  int64_t maxDriftMs = 5000) {
        int64_t serverTime = GetCurrentEpochMs();
        int64_t diff = std::abs(serverTime - clientTimestamp);
        
        if (diff > maxDriftMs) {
            std::cout << "⚠️ Clock drift too high: " << diff << "ms" << std::endl;
            return false;
        }
        
        return true;
    }
    
    // Check for replay attack (action timestamp ≤ last action)
    static bool IsReplayAttack(int64_t actionTimestamp, 
                              int64_t lastActionTimestamp) {
        if (actionTimestamp <= lastActionTimestamp) {
            std::cout << "🚨 REPLAY ATTACK: " << actionTimestamp 
                     << " <= " << lastActionTimestamp << std::endl;
            return true;
        }
        return false;
    }
    
    // Check for future timestamp (clock manipulation)
    static bool IsFutureTimestamp(int64_t actionTimestamp, 
                                  int64_t toleranceMs = 1000) {
        int64_t serverTime = GetCurrentEpochMs();
        
        if (actionTimestamp > serverTime + toleranceMs) {
            std::cout << "🚨 FUTURE TIMESTAMP: " << actionTimestamp 
                     << " > " << serverTime + toleranceMs << std::endl;
            return true;
        }
        return false;
    }
};
```

---

**3. Combat Server**

```cpp
// CombatServer.h
class CombatServer {
public:
    enum class ActionType {
        Attack,
        Defend,
        UsePotion
    };
    
    struct Fighter {
        std::string name;
        Fixed maxHealth;
        Fixed currentHealth;
        Fixed attack;
        Fixed defense;
        int potions;
        bool defending;
    };
    
    struct CombatAction {
        ActionType action;
        int64_t timestamp;
        uint32_t randomSeed;  // For deterministic variance
    };
    
    struct CombatResult {
        bool valid;
        std::string message;
        Fixed damageDealt;
        bool combatOver;
        std::string winner;
    };

private:
    Fighter player;
    Fighter enemy;
    int64_t lastPlayerActionTime;
    std::vector<std::string> actionLog;

public:
    CombatServer(const Fighter& p, const Fighter& e);
    
    CombatResult ProcessPlayerAction(const CombatAction& action);
    CombatResult ProcessEnemyTurn(uint32_t seed);
    
    const Fighter& GetPlayer() const { return player; }
    const Fighter& GetEnemy() const { return enemy; }
    const std::vector<std::string>& GetLog() const { return actionLog; }

private:
    bool ValidateAction(const CombatAction& action);
    Fixed ExecuteAttack(Fighter& attacker, Fighter& defender, uint32_t seed);
    Fixed GetDeterministicVariance(uint32_t seed);
};

// Deterministic random from seed
Fixed CombatServer::GetDeterministicVariance(uint32_t seed) {
    // Use seed to get value 0.0 to 1.0 (deterministic!)
    uint32_t value = seed % 1000;  // 0-999
    return Fixed(value / 1000.0f);
}

CombatResult CombatServer::ProcessPlayerAction(const CombatAction& action) {
    CombatResult result;
    result.valid = false;
    result.damageDealt = Fixed(0);
    result.combatOver = false;
    
    // Validate timestamp
    if (!ValidateAction(action)) {
        result.message = "Invalid timestamp - action rejected!";
        return result;
    }
    
    result.valid = true;
    lastPlayerActionTime = action.timestamp;
    
    // Execute action
    switch (action.action) {
        case ActionType::Attack: {
            Fixed damage = ExecuteAttack(player, enemy, action.randomSeed);
            result.damageDealt = damage;
            result.message = player.name + " attacks for " + 
                           std::to_string((int)damage.ToFloat()) + " damage!";
            break;
        }
        
        case ActionType::Defend:
            player.defending = true;
            result.message = player.name + " defends! (Defense +50%)";
            break;
        
        case ActionType::UsePotion:
            if (player.potions > 0) {
                player.potions--;
                Fixed heal(50);
                player.currentHealth = player.currentHealth + heal;
                if (player.currentHealth > player.maxHealth) {
                    player.currentHealth = player.maxHealth;
                }
                result.message = player.name + " uses potion! (+50 HP)";
            } else {
                result.message = player.name + " has no potions!";
            }
            break;
    }
    
    actionLog.push_back(result.message);
    
    // Check for victory
    if (enemy.currentHealth <= Fixed(0)) {
        result.combatOver = true;
        result.winner = player.name;
    }
    
    return result;
}
```

---

**4. Raylib Visualization**

```cpp
void DrawCombatUI(const CombatServer& server, bool cheatDetected, 
                 float cheatTimer, bool playerTurn) {
    auto player = server.GetPlayer();
    auto enemy = server.GetEnemy();
    auto log = server.GetLog();
    
    // Fighter names
    DrawText(player.name.c_str(), 100, 80, 24, BLUE);
    DrawText(enemy.name.c_str(), 500, 80, 24, RED);
    
    // Health bars with animation
    float playerHealthPercent = player.currentHealth.ToFloat() / 
                               player.maxHealth.ToFloat();
    float enemyHealthPercent = enemy.currentHealth.ToFloat() / 
                              enemy.maxHealth.ToFloat();
    
    // Player health
    DrawRectangle(100, 110, 200, 30, DARKGRAY);
    DrawRectangle(100, 110, (int)(200 * playerHealthPercent), 30, GREEN);
    DrawRectangleLines(100, 110, 200, 30, WHITE);
    DrawText(TextFormat("%.0f / %.0f HP", 
            player.currentHealth.ToFloat(), player.maxHealth.ToFloat()),
            105, 115, 20, WHITE);
    
    // Enemy health
    DrawRectangle(500, 110, 200, 30, DARKGRAY);
    DrawRectangle(500, 110, (int)(200 * enemyHealthPercent), 30, GREEN);
    DrawRectangleLines(500, 110, 200, 30, WHITE);
    DrawText(TextFormat("%.0f / %.0f HP",
            enemy.currentHealth.ToFloat(), enemy.maxHealth.ToFloat()),
            505, 115, 20, WHITE);
    
    // Stats
    DrawText(TextFormat("ATK: %.0f  DEF: %.0f  Potions: %d",
            player.attack.ToFloat(), player.defense.ToFloat(), player.potions),
            100, 150, 14, LIGHTGRAY);
    DrawText(TextFormat("ATK: %.0f  DEF: %.0f",
            enemy.attack.ToFloat(), enemy.defense.ToFloat()),
            500, 150, 14, LIGHTGRAY);
    
    // Action log (last 10 lines)
    DrawRectangle(50, 200, 700, 250, Fade(BLACK, 0.7f));
    int startIdx = std::max(0, (int)log.size() - 10);
    for (size_t i = startIdx; i < log.size(); i++) {
        DrawText(log[i].c_str(), 60, 210 + (i - startIdx) * 22, 16, GRAY);
    }
    
    // Turn indicator
    if (playerTurn) {
        DrawText("YOUR TURN", 320, 470, 24, YELLOW);
        DrawText("[1] Attack  [2] Defend  [3] Potion", 200, 500, 18, WHITE);
    } else {
        DrawText("ENEMY TURN...", 300, 470, 24, RED);
    }
    
    // Cheat detection warning
    if (cheatDetected && cheatTimer > 0) {
        DrawRectangle(250, 230, 300, 100, Fade(RED, 0.9f));
        DrawRectangleLines(250, 230, 300, 100, YELLOW);
        DrawText("CHEATER DETECTED!", 270, 250, 20, BLACK);
        DrawText("Invalid Timestamp", 290, 280, 16, BLACK);
    }
    
    // Debug info
    DrawText("[C] Test Replay Attack", 10, 580, 12, DARKGRAY);
    DrawText("Timestamp Validation: ON", 600, 580, 12, GREEN);
}
```

---

### Complete Main Code

**main.cpp:**
```cpp
#include <raylib.h>
#include "Fixed.h"
#include "TimestampValidator.h"
#include "CombatServer.h"
#include <iostream>
#include <cstdlib>
#include <ctime>

// Test determinism
void TestDeterminism() {
    std::cout << "\n=== Testing Fixed-Point Determinism ===" << std::endl;
    
    Fixed start(100);
    Fixed velocity(10);
    Fixed dt(0.016f);
    
    std::cout << "Starting position: " << start.ToFloat() << std::endl;
    std::cout << "Running 1000 frames of movement..." << std::endl;
    
    for (int run = 0; run < 10; run++) {
        Fixed pos = start;
        for (int frame = 0; frame < 1000; frame++) {
            pos = pos + velocity * dt;
        }
        std::cout << "Run " << run << ": " << pos.ToFloat() << std::endl;
    }
    
    std::cout << "All runs should show IDENTICAL results!\n" << std::endl;
}

// Test timestamp validation
void TestTimestampValidation() {
    std::cout << "=== Testing Timestamp Validation ===" << std::endl;
    
    int64_t serverTime = TimestampValidator::GetCurrentEpochMs();
    
    // Test 1: Valid timestamp
    std::cout << "\nTest 1: Valid timestamp (2ms drift)" << std::endl;
    int64_t validTime = serverTime - 2;
    bool valid = TimestampValidator::ValidateTimestamp(validTime);
    std::cout << (valid ? "PASSED" : "FAILED") << std::endl;
    
    // Test 2: Too old
    std::cout << "\nTest 2: Clock too far behind (7000ms)" << std::endl;
    int64_t oldTime = serverTime - 7000;
    bool tooOld = !TimestampValidator::ValidateTimestamp(oldTime);
    std::cout << (tooOld ? "PASSED (correctly rejected)" : "FAILED") << std::endl;
    
    // Test 3: Replay attack
    std::cout << "\nTest 3: Replay attack detection" << std::endl;
    int64_t action1 = serverTime;
    int64_t action2 = serverTime - 100;  // Older than action1
    bool isReplay = TimestampValidator::IsReplayAttack(action2, action1);
    std::cout << (isReplay ? "PASSED (replay detected)" : "FAILED") << std::endl;
    
    // Test 4: Future timestamp
    std::cout << "\nTest 4: Future timestamp detection" << std::endl;
    int64_t futureTime = serverTime + 5000;
    bool isFuture = TimestampValidator::IsFutureTimestamp(futureTime);
    std::cout << (isFuture ? "PASSED (future timestamp detected)" : "FAILED") << std::endl;
    
    std::cout << std::endl;
}

int main() {
    srand((unsigned int)time(nullptr));
    
    // Run tests
    TestDeterminism();
    TestTimestampValidation();
    
    std::cout << "Press ENTER to start combat simulation..." << std::endl;
    std::cin.get();
    
    // Initialize combat
    CombatServer::Fighter player {
        "Hero",
        Fixed(100),  // maxHealth
        Fixed(100),  // currentHealth
        Fixed(20),   // attack
        Fixed(10),   // defense
        3,           // potions
        false        // defending
    };
    
    CombatServer::Fighter enemy {
        "Goblin",
        Fixed(80),
        Fixed(80),
        Fixed(15),
        Fixed(8),
        0,
        false
    };
    
    CombatServer server(player, enemy);
    
    InitWindow(800, 600, "Turn-Based Combat Simulator");
    SetTargetFPS(60);
    
    bool playerTurn = true;
    bool cheatDetected = false;
    float cheatTimer = 0.0f;
    float enemyTurnDelay = 0.0f;
    
    while (!WindowShouldClose()) {
        float dt = GetFrameTime();
        
        // Update cheat timer
        if (cheatTimer > 0) {
            cheatTimer -= dt;
            if (cheatTimer <= 0) {
                cheatDetected = false;
            }
        }
        
        // Player turn
        if (playerTurn) {
            CombatServer::CombatAction action;
            action.timestamp = TimestampValidator::GetCurrentEpochMs();
            action.randomSeed = rand();
            
            bool actionTaken = false;
            
            if (IsKeyPressed(KEY_ONE)) {
                action.action = CombatServer::ActionType::Attack;
                actionTaken = true;
            } else if (IsKeyPressed(KEY_TWO)) {
                action.action = CombatServer::ActionType::Defend;
                actionTaken = true;
            } else if (IsKeyPressed(KEY_THREE)) {
                action.action = CombatServer::ActionType::UsePotion;
                actionTaken = true;
            }
            
            // Cheat test: old timestamp
            if (IsKeyPressed(KEY_C)) {
                action.timestamp -= 10000;  // 10 seconds old
                action.action = CombatServer::ActionType::Attack;
                actionTaken = true;
                std::cout << "CHEAT TEST: Sending old timestamp!\n" << std::endl;
            }
            
            if (actionTaken) {
                auto result = server.ProcessPlayerAction(action);
                
                if (!result.valid) {
                    cheatDetected = true;
                    cheatTimer = 3.0f;
                    std::cout << result.message << std::endl;
                } else {
                    std::cout << result.message << std::endl;
                    
                    if (result.combatOver) {
                        std::cout << "\n🎉 VICTORY! " << result.winner << " wins!\n" << std::endl;
                    } else {
                        playerTurn = false;
                        enemyTurnDelay = 1.5f;
                    }
                }
            }
        } else {
            // Enemy turn (delayed)
            enemyTurnDelay -= dt;
            if (enemyTurnDelay <= 0) {
                auto result = server.ProcessEnemyTurn(rand());
                std::cout << result.message << std::endl;
                
                if (result.combatOver) {
                    std::cout << "\nDEFEAT! " << result.winner << " wins!\n" << std::endl;
                } else {
                    playerTurn = true;
                }
            }
        }
        
        // Draw
        BeginDrawing();
            ClearBackground(BLACK);
            DrawCombatUI(server, cheatDetected, cheatTimer, playerTurn);
        EndDrawing();
    }
    
    CloseWindow();
    return 0;
}
```

---

### Expected Test Output

```
=== Testing Fixed-Point Determinism ===
Starting position: 100
Running 1000 frames of movement...
Run 0: 260.000000
Run 1: 260.000000
Run 2: 260.000000
Run 3: 260.000000
Run 4: 260.000000
Run 5: 260.000000
Run 6: 260.000000
Run 7: 260.000000
Run 8: 260.000000
Run 9: 260.000000
All runs show IDENTICAL results!

=== Testing Timestamp Validation ===

Test 1: Valid timestamp (2ms drift)
PASSED

Test 2: Clock too far behind (7000ms)
⚠Clock drift too high: 7000ms
PASSED (correctly rejected)

Test 3: Replay attack detection
REPLAY ATTACK: 1234567790 <= 1234567890
PASSED (replay detected)

Test 4: Future timestamp detection
FUTURE TIMESTAMP: 1234572890 > 1234568890
PASSED (future timestamp detected)

=== Combat Simulation ===
Hero attacks Goblin for 13 damage!
Goblin attacks Hero for 7 damage!
Hero defends! (Defense +50%)
Goblin attacks Hero for 4 damage!
Hero uses potion! (+50 HP)
...
Hero attacks Goblin for 15 damage!

VICTORY! Hero wins!
```