# Text-Based Fantasy RPG - Technischer Implementierungsplan v1.0

**Status**: APPROVED  
**Letzte Aktualisierung**: 2024-01-15  
**Nächste Phase**: TASKS

---

## Architektur-Übersicht

```
[GameEngine] -> [WorldManager] -> [Area/Location]
     |              |                    |
     v              v                    v
[CharacterManager] [QuestEngine] -> [NPC/Dialog]
     |              |
     v              v
[CombatSystem] -> [Monster/AI]
     |
     v
[ItemSystem] -> [Inventory]
     |
     v
[SaveSystem] -> [JSON Files]

[GameUI] -> [InputHandler] -> [All Systems]
```

**Architektur-Prinzip**: Hexagonal Architecture (Ports & Adapters)
- **Core Domain**: Character, Combat, Quest, World
- **Application Layer**: GameEngine, Managers
- **Infrastructure**: SaveSystem, ConsoleUI, ContentLoader

---

## Komponenten & Verantwortungen

### Core Engine (Application Layer)
- **`GameEngine`**: Hauptschleife, Koordination aller Systeme, Session-Management
- **`GameState`**: Aktueller Spielzustand, Zustandsübergänge
- **`InputHandler`**: Eingabeverarbeitung, Menü-Navigation, Validierung

### Domain Layer (Business Logic)
- **`CharacterManager`**: Klassen, Stats, Skills, Level-System, Progression
- **`WorldManager`**: Areas, Locations, Verbindungen, NPCs, Navigation
- **`CombatSystem`**: Rundenbasierte Kämpfe, Skills, Statuseffekte, AI
- **`QuestEngine`**: Story-Progression, Entscheidungen, Konsequenzen, Branching
- **`ItemSystem`**: Items, Equipment, Inventory, Crafting

### Infrastructure Layer (Ports & Adapters)
- **`SaveRepository`**: JSON-basierte Persistierung, Schema-Validierung
- **`ContentLoader`**: Statische Daten (Areas, Quests, Monster) aus JSON
- **`ConsoleUI`**: Text-basierte Benutzeroberfläche, Formatierung

---

## API-Verträge (Interne Interfaces)

### Core Interfaces
```java
// System Lifecycle
interface GameSystem {
    void initialize();
    void update(GameState state);
    void shutdown();
}

// Persistence
interface Saveable {
    JsonObject toJson();
    void fromJson(JsonObject data);
}

// Event System
interface GameEventListener {
    void onEvent(GameEvent event);
}
```

### Character System
```java
interface CharacterClass {
    String getName();
    Stats getBaseStats();
    List<Skill> getStartingSkills();
    List<Skill> getAvailableSkills(int level);
    boolean canUseItem(Item item);
}

interface Skill {
    String getName();
    int getManaCost();
    boolean canExecute(Character actor, CombatContext context);
    SkillResult execute(Character actor, CombatContext context);
}
```

### Combat System
```java
interface CombatAction {
    String getName();
    boolean canExecute(Character actor, CombatContext context);
    CombatResult execute(Character actor, Character target, CombatContext context);
    int getPriority(); // Initiative order
}

interface StatusEffect {
    String getName();
    void apply(Character target);
    void tick(Character target); // Per-turn effect
    boolean isExpired();
}
```

### Quest System
```java
interface QuestCondition {
    boolean isMet(GameState state);
    String getDescription();
}

interface QuestReward {
    void apply(Character character, GameState state);
    String getDescription();
}

interface QuestChoice {
    String getText();
    List<QuestConsequence> getConsequences();
    boolean isAvailable(GameState state);
}
```

---

## Datenmodell

### Character Entity
```java
class Character implements Saveable {
    String name;
    CharacterClass characterClass;
    int level, experience;
    Stats stats; // HP, MP, STR, DEX, INT, WIS
    Stats currentStats; // Current HP/MP
    List<Skill> skills;
    Inventory inventory;
    Equipment equipment;
    Map<String, Integer> reputation; // Faction -> Value
    List<StatusEffect> activeEffects;
    QuestLog questLog;
}

class Stats {
    int hitPoints, manaPoints;
    int strength, dexterity, intelligence, wisdom;
    int armor, magicResistance;
}
```

### World Entity
```java
class Area implements Saveable {
    String id, name, description;
    Map<Direction, String> exits; // Direction -> AreaId
    List<NPC> npcs;
    List<Monster> monsters;
    List<Item> items;
    List<String> availableQuests;
    boolean visited;
    Map<String, Object> state; // Area-specific state
}

class NPC {
    String id, name, description;
    List<Dialog> dialogs;
    List<String> quests;
    Map<String, Integer> relationshipRequirements;
}
```

### Quest Entity
```java
class Quest implements Saveable {
    String id, title, description;
    QuestType type; // MAIN, SIDE, RANDOM
    List<QuestStep> steps;
    Map<String, QuestChoice> choices; // ChoiceId -> Consequences
    QuestReward reward;
    List<QuestCondition> prerequisites;
    QuestStatus status; // NOT_STARTED, ACTIVE, COMPLETED, FAILED
}

class QuestStep {
    String id, description;
    List<QuestCondition> conditions;
    List<QuestChoice> choices;
    boolean isCompleted;
}
```

### Combat Entity
```java
class Monster {
    String id, name, description;
    Stats stats;
    List<CombatAction> actions;
    List<Item> lootTable;
    int experienceReward;
    MonsterAI ai;
}

class CombatEncounter {
    List<Character> allies;
    List<Monster> enemies;
    int currentTurn;
    List<CombatAction> actionQueue;
    CombatResult result;
}
```

---

## NFR-Budget

### Performance
- **Spielaktionen**: < 50ms (Combat-Berechnungen, Skill-Ausführung)
- **Weltübergänge**: < 100ms (Area-Loading, NPC-Initialisierung)
- **Speichern/Laden**: < 500ms (JSON-Serialisierung)
- **Memory**: < 100MB RAM für komplette Spielwelt

### Storage
- **Spieldaten**: < 50MB für alle statischen Inhalte
- **Saves**: < 5MB pro Spielstand
- **Logs**: < 100MB pro Spielsitzung

### Reliability
- **Uptime**: 99.9% (keine Crashes während normaler Nutzung)
- **Data Integrity**: Atomic Saves, Backup-System
- **Error Recovery**: Graceful Degradation bei korrupten Daten

---

## Sicherheit

### Input-Validierung
```java
// Alle Benutzereingaben validieren
class InputValidator {
    boolean isValidMenuChoice(String input, int maxOptions);
    boolean isValidCharacterName(String name);
    boolean isValidSaveFileName(String fileName);
    String sanitizeInput(String input);
}
```

### Save-Security
- **Schema-Validierung**: JSON-Schema für alle Save-Dateien
- **Backup-System**: Automatische Backups vor kritischen Operationen
- **Checksummen**: Integritätsprüfung gegen manuelle Manipulation
- **Sanitization**: Dateinamen und Pfade gegen Directory-Traversal

### Error Handling
```java
// Strukturierte Fehlerbehandlung
class GameException extends Exception {
    ErrorType type;
    String userMessage;
    Map<String, Object> context;
}

enum ErrorType {
    SAVE_CORRUPTION, INVALID_INPUT, SYSTEM_ERROR, CONTENT_MISSING
}
```

---

## Observability-Plan

### Logs (JSON-Format)
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "event": "combat.started",
  "player_id": "player_123",
  "character_class": "warrior",
  "character_level": 5,
  "monster": "orc_warrior",
  "area": "dark_forest",
  "trace_id": "abc123",
  "session_id": "sess_456"
}
```

### Metriken
```java
// Counter
game.sessions.total
combat.encounters.total{result=victory|defeat|flee}
quest.completions.total{type=main|side|random}
save.operations.total{result=success|failure}
errors.total{type=save_corruption|invalid_input|system_error}

// Histogram
combat.duration.seconds
character.level.distribution
quest.completion.time.minutes
save.operation.duration.milliseconds

// Gauge
active.characters.count
world.areas.explored.percentage
```

### Traces
```java
// Wichtige Spans
game_session {
  character_creation,
  world_exploration {
    area_transition,
    npc_interaction,
    combat_encounter {
      combat_round,
      skill_execution,
      damage_calculation
    }
  },
  quest_progression {
    quest_start,
    choice_made,
    quest_completion
  },
  save_operation
}
```

---

## Teststrategie

### Unit Tests (80% Coverage)
```java
// Character System
@Test void shouldLevelUpWhenExperienceThresholdReached()
@Test void shouldApplyClassBonusesToStats()
@Test void shouldLearnSkillsAtCorrectLevels()

// Combat System
@Test void shouldCalculateDamageBasedOnStats()
@Test void shouldApplyStatusEffectsCorrectly()
@Test void shouldHandleCriticalHitsAndMisses()

// Quest System
@Test void shouldEvaluateConditionsCorrectly()
@Test void shouldApplyConsequencesOfChoices()
@Test void shouldTrackQuestProgressAccurately()
```

### Integration Tests
```java
// Save/Load Cycles
@Test void shouldPreserveAllDataThroughSaveLoadCycle()
@Test void shouldHandleCorruptedSaveFiles()

// Combat Flows
@Test void shouldExecuteCompleteCombaFromStartToResolution()
@Test void shouldHandleComplexSkillInteractions()

// Quest Progression
@Test void shouldProgressThroughMultiStepQuests()
@Test void shouldBranchCorrectlyBasedOnChoices()
```

### Component Tests
```java
// System Integration
@Test void shouldCoordinateAllSystemsCorrectly()
@Test void shouldHandleStateTransitionsCleanly()
@Test void shouldMaintainDataConsistencyAcrossSystems()
```

### E2E Tests
```java
// Complete Workflows
@Test void shouldSupportFullCharacterLifecycle()
@Test void shouldHandleCompleteQuestWithMultipleChoices()
@Test void shouldSupportAllCombatScenariosForAllClasses()
```

---

## Performance & Resilienz

### Caching-Strategie
```java
// Static Data Caching
class ContentCache {
    Map<String, Area> areaCache;
    Map<String, Monster> monsterTemplates;
    Map<String, Quest> questTemplates;
    Map<String, Item> itemTemplates;
}

// Lazy Loading
class WorldManager {
    Area loadArea(String areaId); // Cache after first load
    void preloadAdjacentAreas(String currentAreaId);
}
```

### Error Handling & Recovery
```java
// Graceful Degradation
class SaveSystem {
    SaveResult save(GameState state) {
        try {
            return atomicSave(state);
        } catch (IOException e) {
            return fallbackSave(state); // Simplified format
        }
    }
    
    GameState load(String saveFile) {
        try {
            return loadWithValidation(saveFile);
        } catch (ValidationException e) {
            return loadWithDefaults(saveFile); // Skip corrupted parts
        }
    }
}
```

### Resource Management
```java
// Memory Management
class ObjectPool<T> {
    Queue<T> available;
    Supplier<T> factory;
    
    T acquire();
    void release(T object);
}

// Pools for frequent objects
ObjectPool<CombatResult> combatResultPool;
ObjectPool<GameEvent> eventPool;
```

---

## Tooling/CI/CD

### Development Stack
```xml
<!-- pom.xml dependencies -->
<dependencies>
    <!-- Core -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.16.1</version>
    </dependency>
    
    <!-- Logging -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>2.0.9</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.4.14</version>
    </dependency>
    
    <!-- Testing -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.8.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Code Quality
```xml
<!-- Maven plugins -->
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.2.0</version>
</plugin>
<plugin>
    <groupId>com.puppycrawl.tools</groupId>
    <artifactId>checkstyle</artifactId>
    <version>10.12.5</version>
</plugin>
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <configuration>
        <rules>
            <rule>
                <element>BUNDLE</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.75</minimum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</plugin>
```

### CI Pipeline
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
      - run: mvn clean compile
      - run: mvn test
      - run: mvn spotbugs:check
      - run: mvn checkstyle:check
      - run: mvn jacoco:report jacoco:check
      - run: mvn package
      - run: java -jar target/textbased-fantasy-*.jar --smoke-test
```

---

## Risiken, Alternativen, Trade-offs

### Identifizierte Risiken

| Risiko | Wahrscheinlichkeit | Impact | Mitigation |
|--------|-------------------|--------|------------|
| Quest-System zu komplex | Hoch | Mittel | Einfache Quest-Types zuerst, iterative Erweiterung |
| Kampf-Balancing schwierig | Mittel | Hoch | Konfigurierbare Formeln, extensive Playtests |
| Save-Corruption | Niedrig | Hoch | Schema-Validierung, Backup-System, Atomic Saves |
| Performance bei großen Welten | Mittel | Mittel | Lazy Loading, Caching, Memory-Pools |

### Technische Alternativen

| Entscheidung | Gewählt | Alternative | Trade-off |
|--------------|---------|-------------|-----------|
| Persistierung | JSON | SQLite Database | JSON: Einfacher, Human-readable vs. DB: Performanter, ACID |
| Architektur | Hexagonal | Layered | Hexagonal: Testbarer, flexibler vs. Layered: Einfacher |
| Logging | SLF4J+Logback | java.util.logging | SLF4J: Professioneller, konfigurierbar vs. JUL: Keine Dependencies |
| Testing | JUnit 5 | TestNG | JUnit 5: Standard, gute IDE-Integration vs. TestNG: Flexiblere Konfiguration |

---

## Plan-Review-Check (Gate)

- [x] Architektur mit klaren Boundaries (Hexagonal Architecture)
- [x] API-Verträge definiert (Interfaces für alle Hauptkomponenten)
- [x] Datenmodell spezifiziert (Entities mit Beziehungen)
- [x] NFR-Budget konkretisiert (Performance, Memory, Storage)
- [x] Security-Konzept (Input-Validation, Save-Security, Error Handling)
- [x] Observability vollständig (Logs, Metriken, Traces mit Beispielen)
- [x] Teststrategie auf allen Ebenen (Unit, Integration, Component, E2E)
- [x] Performance & Resilienz (Caching, Error Recovery, Resource Management)
- [x] Tooling/CI/CD (Dependencies, Quality Gates, Pipeline)
- [x] Risiken identifiziert mit Mitigations

**Status**: ✅ APPROVED

---

**Nächste Aktion**: `/tasks` für detaillierte Implementierungsaufgaben mit DoD und Tests.