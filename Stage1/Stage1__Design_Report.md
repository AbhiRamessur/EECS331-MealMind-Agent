# EECS 3311 — Stage 1 Design Report
## MealMind: AI Smart Recipe & Meal Planning Agent

| | |
|---|---|
| **Student** | _Abhishek Ramessur_ |
| **Student ID** | _219433564_ |
| **Repository** | _https://github.com/AbhiRamessur/EECS3311-MealPlanning-Agent/_ |

---

## 1.1 Project Description

### Problem and Motivation
People struggle to decide what to eat each week while respecting allergies, diets, budgets, and what is already in their kitchen. This leads to food waste, repeated takeout, and duplicate grocery purchases. Recipe websites do not know the user's pantry, and a plain chatbot may ignore allergies or invent nutrition numbers. MealMind combines deterministic software components (pantry, scaling, shopping lists, constraint checking) with an LLM-based agent that plans, uses tools, verifies its own output, and revises it.

### Target Users
Primary users are students and busy individuals planning weekly meals on a budget, people with dietary restrictions or allergies, and small households that want to reduce food waste.

### Agent Capabilities & Tools
MealMind combines generative reasoning with deterministic execution. The agent utilizes the following primary tool interfaces:
- `PantryTool`: Queries available ingredients, quantities, and upcoming expiration dates.
- `ConstraintValidator`: Deterministically verifies plans against user dietary rules and allergen blacklists.
- `RecipeSearchTool`: Retrieves verified structured recipes from local storage or external APIs.
- `MealPlanEditorTool`: Generates and updates multi-day meal plan structures with full undo/redo capabilities.
- `ShoppingListTool`: Aggregates missing ingredients and consolidates quantities.

**AI / deterministic boundary:** the LLM handles language understanding and generation (recipes, plan choices, substitutions, edit interpretation, commentary). Deterministic code handles storage, pantry arithmetic, unit conversion, scaling, shopping-list consolidation, nutrition numbers, and all safety validation. The LLM never accesses data or the UI directly.

### Why an agent is appropriate
| System Requirement | Why an Agent (Not a Static LLM Prompt) |
|---|---|
| **Live User Context** | Retrieves real-time pantry and profile data via deterministic tools instead of guessing. |
| **Multi-Constraint Reasoning** | Executes multi-step workflows: gather inventory → draft plan → validate constraints → revise. |
| **Safety & Verification** | Output is verified by `ConstraintValidator`. If a violation occurs, the agent receives structural feedback and retries automatically. |
| **Natural Language Editing** | Converts ambiguous user text (*"Make Friday's dinner lower calorie"*) into deterministic, undoable command execution. |
| **Conversational Memory** | Maintains session context across multiple turns to handle follow-up edits and preferences. |

Demonstrated Agent Behaviors: **Tool invocation, context retrieval, multi-step execution (ReAct loop), stateful memory, automated self-correction, and planning.**

### AI/LLM Model Selection
* **Primary Model**: **OpenAI GPT-4o** accessed via REST API. Chosen for its strict adherence to JSON schema function calling, high reasoning capacity, and low latency for complex multi-step planning.
* **Secondary / Offline Fallback Model**: **DeepSeek-R1 (8B)** or **Llama 3 (8B)** executed locally via **Ollama** (`http://localhost:11434`).
* **Architectural Flexibility**: The system abstracts LLM interaction behind an `LLMClient` interface using the **Strategy Pattern**. This allows runtime switching between cloud APIs and local Ollama instances based on network availability or user configuration without modifying domain logic.

### AI Model Interaction Mechanism
The AI model interacts with the rest of the software system through a structured, multi-step Agent-Tool loop rather than direct UI or data access:

1. **Tool Schema Registration**: At startup, `ToolManager` inspects available Java tools (`PantryTool`, `ConstraintValidator`, `RecipeSearchTool`, `MealPlanEditorTool`, `ShoppingListTool`) and exposes their signatures as standardized JSON schemas specifying required parameters and method descriptions.
2. **Context Assembly and Prompt Dispatch**: When a user enters a request through either the JavaFX GUI or CLI interface, `AgentController` gathers the user's active dietary profile, pantry state, conversation history, and registered tool schemas into a structured prompt payload. This payload is dispatched to `LLMClient`.
3. **Structured Tool Execution**: Rather than generating raw unstructured text, the LLM emits a JSON payload containing specific tool invocation requests. `ToolManager` intercepts these payloads, parses the arguments, and reflectively invokes the corresponding methods on the deterministic Java domain models.
4. **Deterministic Validation and Retry Loop**: Data generated or retrieved by tools passes through `ConstraintValidator` to enforce allergen safety and macro constraints. If safety rules are violated, the validation failure details are sent directly back into the LLM's conversation context, prompting the model to automatically re-evaluate and self-correct its output.
5. **Decoupled State and UI Synchronization**: Once a valid response is generated, `AgentController` updates the underlying domain objects (`MealPlan`, `Pantry`). These state updates trigger notifications through the Observer pattern, automatically updating the JavaFX GUI components or displaying formatted output in the CLI console without giving the LLM direct access to presentation or persistence layers.

## 1.2 Feature Specification

**Legend:** **D** = Deterministic | **AI** = AI-Based | **H** = Hybrid

---

### F01 — Preference & Dietary Profile Management
* **Type**: Hybrid (**H**)
* **Description**: Captures and maintains user dietary preferences, allergies, macro targets, calorie limits, budget constraints, and household serving sizes. Users can enter data via form fields or plain text.
* **User Interaction**:
  * **GUI**: *Profile Tab* containing form inputs alongside a natural language text box (*"Describe your diet..."*).
  * **CLI**: `profile set --text "I am vegetarian and allergic to peanuts"` or `profile update --calories 2200 --budget 150`.
* **Input**: Structured form values or natural language string.
* **Output**: Confirmed `UserProfile` object displayed in GUI/CLI.
* **AI Involvement**: The LLM parses free-text descriptions into structured JSON fields. Validation and file persistence are strictly deterministic.
* **Workflow**: User submits input → `AgentController.updateProfile()` → If free-text, `LLMClient` parses input into structured fields → `ConstraintValidator` validates bounds → `ProfileRepository.save()` updates state → **Observer Pattern** notifies active views.
* **Errors & Exceptions**: If `LLMClient` is unreachable, falls back to manual form input with an alert. If parsed numbers are invalid (e.g., negative budget), rejects entry and displays specific error bounds.

---

### F02 — Pantry Inventory Management
* **Type**: Deterministic (**D**)
* **Description**: Full CRUD management of kitchen stock, including item names, quantities, units, and expiration dates.
* **User Interaction**:
  * **GUI**: *Pantry Tab* featuring an interactive table, "Add Item" modal dialog, unit dropdowns, and red highlighting for near-expiry items.
  * **CLI**: `pantry add "Chicken Breast" 500 g 2026-10-10`, `pantry list`, `pantry remove "Milk"`.
* **Input**: Item name, numerical quantity, unit type, optional expiry date.
* **Output**: Updated `Pantry` state with consolidated quantities and color-coded expiration flags.
* **AI Involvement**: None (100% Deterministic).
* **Workflow**: User issues command → `AgentController.addPantryItem()` creates an `AddPantryItemCommand` (**Command Pattern**) → Executed on `Pantry` domain model → Quantities merged via `UnitConverter` → **Observer Pattern** triggers UI and shopping list refreshes.
* **Errors & Exceptions**: Negative quantities or incompatible units (e.g., merging `grams` and `liters` without density metrics) trigger a validation error without changing state.

---

### F03 — Smart Pantry Recipe Recommendation
* **Type**: Hybrid (**H**)
* **Description**: Analyzes current pantry inventory and user dietary rules to rank and recommend matching recipes with natural-language reasoning.
* **User Interaction**:
  * **GUI**: *Recipes Tab* "Recommend from Pantry" button; results render as recipe cards showing pantry coverage percentage and AI explanation badges.
  * **CLI**: `recipe recommend --pantry-first --max-time 30`.
* **Input**: Current `Pantry` snapshot, `UserProfile`, optional time/cuisine filters.
* **Output**: Ranked list of `Recipe` objects with match percentage and textual justification.
* **AI Involvement**: `RecipeSearchTool` deterministically scores inventory overlap and filters candidates; LLM re-ranks candidates and generates concise explanations.
* **Workflow**: User clicks recommend → `AgentController.recommendRecipes()` → `RecipeSearchTool` calculates ingredient overlap → `ConstraintValidator` eliminates allergen matches → LLM generates textual justifications for top candidates → Returned to view.
* **Errors & Exceptions**: If the pantry is empty, recommends purely based on user profile. If zero recipes match strict constraints, automatically relaxes non-safety constraints (e.g., prep time) and alerts the user.

---

### F04 — AI Recipe Generation
* **Type**: AI-Based (**AI**)
* **Description**: Synthesizes custom recipes from free-form user prompts while strictly adhering to user dietary safety constraints.
* **User Interaction**:
  * **GUI**: *Recipes Tab* prompt bar (*"Create a high-protein vegan pasta dinner under 20 mins"*) with a preview/save modal.
  * **CLI**: `recipe generate "high-protein vegan pasta under 20 mins"`.
* **Input**: Free-text prompt, active `UserProfile`.
* **Output**: A fully structured `Recipe` object (title, scaled ingredients, step-by-step instructions, macros).
* **AI Involvement**: LLM generates recipe details into a strict JSON schema; deterministic code validates ingredients against allergen blacklists.
* **Workflow**: User submits prompt → `AgentController.generateRecipe()` → `LLMClient` generates structured recipe → `ConstraintValidator` checks against `UserProfile` → If violations occur, error feedback is passed back to LLM for automated retry (max 3 loops) → Displayed in preview dialog.
* **Errors & Exceptions**: If JSON is malformed, triggers an internal repair prompt. If validation fails after 3 retries, displays the candidate recipe flagged with explicit safety warnings (never saved automatically).

---

### F05 — Weekly Meal Plan Generation
* **Type**: Hybrid (**H**)
* **Description**: Generates multi-day meal calendars (1–7 days) balancing dietary rules, budget targets, pantry utilization, and chosen strategy.
* **User Interaction**:
  * **GUI**: *Plan Tab* with duration controls, strategy selector (*Pantry-First*, *Budget*, *High-Protein*), "Generate Plan" button, and an *Agent Activity* live log panel.
  * **CLI**: `plan generate --days 7 --strategy pantry-first --budget 120`.
* **Input**: `PlanRequest` (day count, strategy enum, budget ceiling, natural-language notes).
* **Output**: Populated `MealPlan` displayed on a calendar grid with daily cost and macro totals.
* **AI Involvement**: The LLM selects recipe combinations using tool calls; financial totals, unit scaling, and constraint enforcement are deterministic.
* **Workflow**: `AgentController.generatePlan()` instantiates selected `PlanningStrategy` (**Strategy Pattern**) → ReAct agent loop calls `PantryTool` and `RecipeSearchTool` → Assembles draft `MealPlan` (**Composite Pattern**) → `ConstraintValidator` verifies macros/allergens → **Observer Pattern** renders updated plan on GUI/CLI.
* **Errors & Exceptions**: Network timeout during LLM execution prompts user to retry or switch to the local rule-based fallback generator using cached recipes.

---

### F06 — Natural-Language Plan Editing
* **Type**: Hybrid (**H**)
* **Description**: Modifies existing meal plans via plain English instructions with full undo/redo capabilities.
* **User Interaction**:
  * **GUI**: Chat sidebar below calendar grid; interactive *Undo* / *Redo* toolbar buttons.
  * **CLI**: `plan edit "Swap Tuesday dinner for a 15-minute quick meal"` or `plan undo`.
* **Input**: Natural-language instruction string, active `MealPlan`.
* **Output**: Updated `MealPlan` and an execution entry pushed to `CommandHistory`.
* **AI Involvement**: LLM interprets natural language intent into a structured command payload; modifying the plan object and managing history are deterministic.
* **Workflow**: User enters edit command → `AgentController.modifyPlan()` passes context to `LLMClient` → LLM returns target slot and action → `ReplaceMealCommand` (**Command Pattern**) created → `ConstraintValidator` checks new meal → Command executed and pushed to `CommandHistory` → Calendar view updates.
* **Errors & Exceptions**: Ambiguous targets (*"Change dinner"*) prompt a clarifying question. If the proposed edit violates safety rules, command execution is aborted and the plan remains untouched.

---

### F07 — AI Ingredient Substitution
* **Type**: AI-Based (**AI**)
* **Description**: Recommends context-aware ingredient replacements based on dietary restrictions, missing pantry items, or personal dislikes.
* **User Interaction**:
  * **GUI**: Right-click context menu on any recipe ingredient → *Suggest Substitute...* dialog.
  * **CLI**: `recipe substitute --id R101 --ingredient "Peanut Butter" --reason allergy`.
* **Input**: `Recipe`, target ingredient name, replacement reason (*Allergy*, *Missing*, *Preference*).
* **Output**: Top 3 replacement options with conversion ratios, culinary impact notes, and pantry availability flags.
* **AI Involvement**: LLM analyzes flavor profiles and functionality to suggest substitutes; `PantryTool` cross-references available inventory.
* **Workflow**: User requests substitution → `AgentController.suggestSubstitution()` queries `PantryTool` for owned alternatives → LLM generates candidate substitutes with scaled ratios → `ConstraintValidator` filters out candidates containing allergens → User selects option → `SubstituteIngredientCommand` (**Command Pattern**) updates recipe.
* **Errors & Exceptions**: If no safe substitute exists, the system advises replacing the entire meal and suggests alternative recipes.

---

### F08 — Deterministic Recipe & Meal Scaling
* **Type**: Deterministic (**D**)
* **Description**: Dynamically rescales ingredient quantities and macronutrient totals based on target serving sizes.
* **User Interaction**:
  * **GUI**: Numerical serving size spinner on recipe detail and meal plan views.
  * **CLI**: `recipe scale --id R101 --servings 6`.
* **Input**: Target serving count, `Recipe` or `Meal` instance.
* **Output**: Rescaled ingredient quantities rounded to standard culinary measurements.
* **AI Involvement**: None (100% Deterministic).
* **Workflow**: User adjusts spinner → `AgentController.scaleRecipe()` invokes `RecipeScaler` → Linear scaling factor applied to quantitative ingredients → Non-scalable items (*"pinch of salt"*) preserved via regular expressions → `ScaleRecipeCommand` (**Command Pattern**) updates entity → UI refreshes.
* **Errors & Exceptions**: Values $\le 0$ are rejected with an inline validation message.

---

### F09 — Multi-Level Nutrition Analysis & AI Commentary
* **Type**: Hybrid (**H**)
* **Description**: Aggregates macro and micronutrient totals across meals, days, and weeks, providing plain-language health insights against user targets.
* **User Interaction**:
  * **GUI**: *Nutrition Tab* rendering daily progress bars alongside an "AI Nutrition Assessment" panel.
  * **CLI**: `plan nutrition` or `plan nutrition --explain`.
* **Input**: Active `MealPlan`, `UserProfile` targets.
* **Output**: `NutritionSummary` data object and generated natural-language commentary.
* **AI Involvement**: Calculations are 100% deterministic using local database lookups; LLM only interprets the computed figures to generate commentary.
* **Workflow**: `AgentController.getNutritionSummary()` recursively sums macros across `MealPlan` items using `NutritionTool` (**Composite Pattern**) → `NutritionSummary` computed → Summary payload forwarded to LLM for dietary commentary → Results displayed on view.
* **Errors & Exceptions**: Unrecognized ingredients are flagged as "Nutritional Data Unavailable" and excluded from exact sums with a warning badge.

---

### F10 — Expiring Inventory Leftover Utilization
* **Type**: AI-Based (**AI**)
* **Description**: Identifies near-expiry pantry items and constructs meal plan proposals designed to eliminate food waste.
* **User Interaction**:
  * **GUI**: *Pantry Tab* "Clear Expiring Items" button; previews a proposed meal insertion into open plan slots.
  * **CLI**: `plan leftovers --days 3`.
* **Input**: Expiring `PantryItem` list, unallocated `MealPlan` slots.
* **Output**: Proposed meal additions or modifications targeting leftover consumption.
* **AI Involvement**: LLM synthesizes recipes or selects existing database meals prioritizing expiring ingredients; validation is deterministic.
* **Workflow**: `AgentController.planLeftovers()` fetches items expiring within $N$ days via `PantryTool` → Instantiates `LeftoverStrategy` (**Strategy Pattern**) → LLM selects or drafts meals incorporating items → `ConstraintValidator` checks safety → Proposal modal shown → On confirmation, `ReplaceMealCommand` (**Command Pattern**) executes.
* **Errors & Exceptions**: If no items are near expiry, informs the user no action is required. If no open slots exist in the plan, offers to overwrite existing non-leftover meals.

---

### F11 — Consolidated Shopping List Generation
* **Type**: Deterministic (**D**)
* **Description**: Compiles a consolidated shopping list by subtracting current pantry inventory from total recipe requirements in the meal plan.
* **User Interaction**:
  * **GUI**: *Shopping List Tab* auto-updating table with check-off boxes, store section groupings, and text export.
  * **CLI**: `shopping list`, `shopping export --format txt`.
* **Input**: Active `MealPlan`, current `Pantry` state.
* **Output**: Consolidated `ShoppingList` grouped by aisle with estimated cost totals.
* **AI Involvement**: None (100% Deterministic).
* **Workflow**: `MealPlan` or `Pantry` modifications trigger **Observer Pattern** → `ShoppingListGenerator.generate()` aggregates ingredient demand → `UnitConverter` normalizes units → Subtracts available pantry stock → Groups by category → Renders to GUI/CLI.
* **Errors & Exceptions**: Incompatible ingredient unit conversions (e.g., `count` vs `grams`) keep items as separate list entries with an explanatory note.

---

### F12 — Recipe Rating, Favorites & Persistent Memory
* **Type**: Deterministic (**D**)
* **Description**: Allows users to rate recipes (1–5 stars) and favorite items. Preference history persists locally and influences future AI planning choices.
* **User Interaction**:
  * **GUI**: Interactive star rating bar and heart favorite icon on recipe cards; "Favorites" filter toggle.
  * **CLI**: `recipe rate --id R101 --stars 5`, `recipe favorite --id R101`.
* **Input**: Recipe ID, numerical rating (1–5), boolean favorite flag.
* **Output**: Updated rating state saved to disk; updated `AgentMemory` context.
* **AI Involvement**: Deterministic storage. Stored ratings are loaded into `AgentMemory` to bias future LLM prompts toward highly rated recipes.
* **Workflow**: User rates recipe → `AgentController.rateRecipe()` updates `RecipeRepository` → Event recorded in `AgentMemory` → Future agent system prompts automatically receive top-rated recipes as preferred context.
* **Errors & Exceptions**: Out-of-bounds ratings ($<1$ or $>5$) are rejected by domain validation.

## 2.1 Class Diagram


## Section 3: System Design & Architectural Modeling

### 3.1 Design Pattern Specifications

The `MealMind` architecture integrates 5 object-oriented design patterns to solve specific design challenges, ensuring loose coupling, high extensibility, and maintainability.

| Pattern & Design Problem Addressed | Participating Classes | Class Roles | Why Pattern is Appropriate | Difficulty Without Pattern |
|---|---|---|---|---|
| **Strategy Pattern**<br><br>**Problem**: Dynamically switching meal planning algorithms (e.g., Pantry-First, Budget-Conscious, High-Protein) at runtime based on user goals without cluttering agent logic. | • `PlanningStrategy` *(Interface)*<br>• `PantryFirstStrategy` *(Concrete Strategy)*<br>• `BudgetConsciousStrategy` *(Concrete Strategy)*<br>• `HighProteinStrategy` *(Concrete Strategy)*<br>• `MealPlanningAgent` *(Context)* | • **`PlanningStrategy`**: Defines unified interface for system prompt construction and constraint weighting.<br>• **`PantryFirst...` / `Budget...` / `HighProtein...`**: Encapsulate specific prompt strategies and recipe filtering rules.<br>• **`MealPlanningAgent`**: Holds reference to active strategy and delegates plan execution. | Encapsulates planning logic into standalone strategy objects. Allows adding new planning modes (e.g., Keto, Low-Carb) without modifying the core agent execution loop. | Core agent code would be polluted with conditional `if-else` or `switch` statements. Adding new strategies would require modifying existing, tested code, violating the Open/Closed Principle. |
| **Command Pattern**<br><br>**Problem**: Encapsulating plan modifications, recipe scaling, and pantry updates into standalone command objects to support full undo/redo history and execution logging. | • `Command` *(Interface)*<br>• `ReplaceMealCommand` *(Concrete)*<br>• `AddPantryItemCommand` *(Concrete)*<br>• `ScaleRecipeCommand` *(Concrete)*<br>• `CommandHistory` *(Invoker)*<br>• `MealPlan` / `Pantry` *(Receivers)* | • **`Command`**: Declares `execute()` and `undo()` signatures.<br>• **`ReplaceMeal...` / `AddPantry...` / `ScaleRecipe...`**: Store target parameters and execute actions on target receivers.<br>• **`CommandHistory`**: Maintains execution stacks for undo/redo.<br>• **`MealPlan` / `Pantry`**: Receivers containing underlying domain state. | Decouples UI/LLM request interpretation from actual domain state mutation. Manages action state history in a centralized, reusable invoker. | State changes would be applied directly to domain models with no record. Supporting multi-step undo/redo would require manual state-snapshot hacks or complex reverse-mutation logic scattered throughout UI controllers. |
| **Observer Pattern**<br><br>**Problem**: Keeping dual presentation interfaces (JavaFX GUI and CLI Console) and derived domain objects (`ShoppingList`) synchronized whenever `Pantry` or `MealPlan` state changes. | • `ObservableSubject` *(Base Class)*<br>• `Pantry` *(Concrete Subject)*<br>• `MealPlan` *(Concrete Subject)*<br>• `ModelObserver` *(Interface)*<br>• `CalendarView` *(Concrete Observer)*<br>• `CLIConsoleView` *(Concrete Observer)*<br>• `ShoppingListGenerator` *(Concrete Observer)* | • **`ObservableSubject`**: Manages observer registration and notification emission.<br>• **`Pantry` / `MealPlan`**: Emit change events upon state mutation.<br>• **`ModelObserver`**: Defines `onStateChanged()` contract.<br>• **`CalendarView` / `CLIConsole...` / `ShoppingList...`**: React to state notifications by re-rendering UI or updating dependent entities. | Ensures loose coupling between domain models and presentation/derived layers. Model classes remain completely uncoupled from GUI/CLI details. | Domain models would need direct references to GUI components and shopping list managers, requiring explicit manual update calls after every single mutation, creating tight coupling and fragile UI code. |
| **Composite Pattern**<br><br>**Problem**: Treating individual ingredients, single recipes, daily meals, and multi-day plans uniformly when recursively calculating macronutrients and total costs. | • `MealComponent` *(Component Interface)*<br>• `Ingredient` *(Leaf)*<br>• `Recipe` *(Composite)*<br>• `Meal` *(Composite)*<br>• `DayPlan` *(Composite)*<br>• `MealPlan` *(Composite)* | • **`MealComponent`**: Declares common operations (`getCalories()`, `getCost()`, `getProtein()`).<br>• **`Ingredient`**: Returns leaf-level quantitative values.<br>• **`Recipe` / `Meal` / `DayPlan` / `MealPlan`**: Maintain children lists and recursively aggregate values from nested components. | Allows client code to treat single ingredients and full 7-day meal plans identically using a single polymorphic interface call (`getCalories()`). | Aggregating nutrients across different levels would require nested loops, type checking (`instanceof`), and explicit branching logic scattered across UI and controller code. |
| **Facade Pattern**<br><br>**Problem**: Hiding the complex interaction mechanics of the agent subsystem (LLM API REST calls, JSON schema reflection, tool execution, and retry/validation loops) behind a simplified entry point for UI controllers. | • `AgentFacade` *(Facade)*<br>• `AgentController` *(Subsystem Class)*<br>• `LLMClient` *(Subsystem Class)*<br>• `ToolManager` *(Subsystem Class)*<br>• `ConstraintValidator` *(Subsystem Class)*<br>• `AgentMemory` *(Subsystem Class)*<br>• `MainController` / `CLIHandler` *(Clients)* | • **`AgentFacade`**: Provides clean, high-level methods (`generatePlan()`, `modifyPlan()`, `recommendFromPantry()`).<br>• **`AgentController` / `LLMClient` / `ToolManager` / `ConstraintValidator`**: Subsystem components executing tool loops, validation retries, and network calls.<br>• **`MainController` / `CLIHandler`**: Presentation controllers interacting strictly via the facade. | Shields presentation layers (JavaFX / CLI) from the low-level details of LLM schema parsing, tool reflection, and constraint feedback loops. | UI controllers would be directly responsible for managing HTTP network clients, parsing JSON tool definitions, executing reflection calls, and managing re-try validation loops, resulting in bloated, unmaintainable UI code. |
