# EECS 3311 — Stage 1 Design Report
## MealMind: AI Smart Recipe & Meal Planning Agent

| | |
|---|---|
| **Student** | _Abhishek Ramessur_ |
| **Student ID** | _219433564_ |
| **Repository** | _[https://github.com/AbhiRamessur/EECS3311-MealMind-Agent/](https://github.com/AbhiRamessur/EECS331-MealMind-Agent/)_ |

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

#### Class Diagram: [`diagrams/MealMind_Class_Diagram.svg`](diagrams/MealMind_Class_Diagram.svg)

### Design Pattern Specifications

The `MealMind` architecture integrates 5 object-oriented design patterns to solve specific design challenges, ensuring loose coupling, high extensibility, and maintainability.

| Pattern & Design Problem Addressed | Participating Classes | Class Roles | Why Pattern is Appropriate | Difficulty Without Pattern |
|---|---|---|---|---|
| **Strategy Pattern**<br><br>**Problem**: Dynamically switching meal planning algorithms (e.g., Pantry-First, Budget-Conscious, High-Protein) at runtime based on user goals without cluttering agent logic. | • `PlanningStrategy` *(Interface)*<br>• `PantryFirstStrategy` *(Concrete Strategy)*<br>• `BudgetConsciousStrategy` *(Concrete Strategy)*<br>• `HighProteinStrategy` *(Concrete Strategy)*<br>• `MealPlanningAgent` *(Context)* | • **`PlanningStrategy`**: Defines unified interface for system prompt construction and constraint weighting.<br>• **`PantryFirst...` / `Budget...` / `HighProtein...`**: Encapsulate specific prompt strategies and recipe filtering rules.<br>• **`MealPlanningAgent`**: Holds reference to active strategy and delegates plan execution. | Encapsulates planning logic into standalone strategy objects. Allows adding new planning modes (e.g., Keto, Low-Carb) without modifying the core agent execution loop. | Core agent code would be polluted with conditional `if-else` or `switch` statements. Adding new strategies would require modifying existing, tested code, violating the Open/Closed Principle. |
| **Command Pattern**<br><br>**Problem**: Encapsulating plan modifications, recipe scaling, and pantry updates into standalone command objects to support full undo/redo history and execution logging. | • `Command` *(Interface)*<br>• `ReplaceMealCommand` *(Concrete)*<br>• `AddPantryItemCommand` *(Concrete)*<br>• `ScaleRecipeCommand` *(Concrete)*<br>• `CommandHistory` *(Invoker)*<br>• `MealPlan` / `Pantry` *(Receivers)* | • **`Command`**: Declares `execute()` and `undo()` signatures.<br>• **`ReplaceMeal...` / `AddPantry...` / `ScaleRecipe...`**: Store target parameters and execute actions on target receivers.<br>• **`CommandHistory`**: Maintains execution stacks for undo/redo.<br>• **`MealPlan` / `Pantry`**: Receivers containing underlying domain state. | Decouples UI/LLM request interpretation from actual domain state mutation. Manages action state history in a centralized, reusable invoker. | State changes would be applied directly to domain models with no record. Supporting multi-step undo/redo would require manual state-snapshot hacks or complex reverse-mutation logic scattered throughout UI controllers. |
| **Observer Pattern**<br><br>**Problem**: Keeping dual presentation interfaces (JavaFX GUI and CLI Console) and derived domain objects (`ShoppingList`) synchronized whenever `Pantry` or `MealPlan` state changes. | • `ObservableSubject` *(Base Class)*<br>• `Pantry` *(Concrete Subject)*<br>• `MealPlan` *(Concrete Subject)*<br>• `ModelObserver` *(Interface)*<br>• `CalendarView` *(Concrete Observer)*<br>• `CLIConsoleView` *(Concrete Observer)*<br>• `ShoppingListGenerator` *(Concrete Observer)* | • **`ObservableSubject`**: Manages observer registration and notification emission.<br>• **`Pantry` / `MealPlan`**: Emit change events upon state mutation.<br>• **`ModelObserver`**: Defines `onStateChanged()` contract.<br>• **`CalendarView` / `CLIConsole...` / `ShoppingList...`**: React to state notifications by re-rendering UI or updating dependent entities. | Ensures loose coupling between domain models and presentation/derived layers. Model classes remain completely uncoupled from GUI/CLI details. | Domain models would need direct references to GUI components and shopping list managers, requiring explicit manual update calls after every single mutation, creating tight coupling and fragile UI code. |
| **Composite Pattern**<br><br>**Problem**: Treating individual ingredients, single recipes, daily meals, and multi-day plans uniformly when recursively calculating macronutrients and total costs. | • `MealComponent` *(Component Interface)*<br>• `Ingredient` *(Leaf)*<br>• `Recipe` *(Composite)*<br>• `Meal` *(Composite)*<br>• `DayPlan` *(Composite)*<br>• `MealPlan` *(Composite)* | • **`MealComponent`**: Declares common operations (`getCalories()`, `getCost()`, `getProtein()`).<br>• **`Ingredient`**: Returns leaf-level quantitative values.<br>• **`Recipe` / `Meal` / `DayPlan` / `MealPlan`**: Maintain children lists and recursively aggregate values from nested components. | Allows client code to treat single ingredients and full 7-day meal plans identically using a single polymorphic interface call (`getCalories()`). | Aggregating nutrients across different levels would require nested loops, type checking (`instanceof`), and explicit branching logic scattered across UI and controller code. |
| **Facade Pattern**<br><br>**Problem**: Hiding the complex interaction mechanics of the agent subsystem (LLM API REST calls, JSON schema reflection, tool execution, and retry/validation loops) behind a simplified entry point for UI controllers. | • `AgentFacade` *(Facade)*<br>• `AgentController` *(Subsystem Class)*<br>• `LLMClient` *(Subsystem Class)*<br>• `ToolManager` *(Subsystem Class)*<br>• `ConstraintValidator` *(Subsystem Class)*<br>• `AgentMemory` *(Subsystem Class)*<br>• `MainController` / `CLIHandler` *(Clients)* | • **`AgentFacade`**: Provides clean, high-level methods (`generatePlan()`, `modifyPlan()`, `recommendFromPantry()`).<br>• **`AgentController` / `LLMClient` / `ToolManager` / `ConstraintValidator`**: Subsystem components executing tool loops, validation retries, and network calls.<br>• **`MainController` / `CLIHandler`**: Presentation controllers interacting strictly via the facade. | Shields presentation layers (JavaFX / CLI) from the low-level details of LLM schema parsing, tool reflection, and constraint feedback loops. | UI controllers would be directly responsible for managing HTTP network clients, parsing JSON tool definitions, executing reflection calls, and managing re-try validation loops, resulting in bloated, unmaintainable UI code. |

## 2.2 Use Case Diagram and Descriptions

#### Use Case Diagram: [`diagrams/MealMind_Use_Case_Diagram.svg`](diagrams/MealMind_Use_Case_Diagram.svg) 

### Major Use Case Descriptions:

#### UC02 — Manage Pantry Inventory
* **Use Case ID**: UC02
* **Use Case Name**: Manage Pantry Inventory
* **Actor(s)**: User, Pantry Model, CommandHistory
* **Goal**: Add, update, or remove ingredients in the user's local pantry inventory to drive smart recipe recommendations.
* **Preconditions**: Application is running with an active user session.
* **Trigger**: User navigates to the Pantry Tab and clicks "Add Item".
* **Main Success Scenario**:
  1. User inputs item name, quantity, unit type, and expiry date.
  2. `AgentController.addPantryItem()` creates an `AddPantryItemCommand` (**Command Pattern**).
  3. `Pantry` model receives the item; measurement units are normalized, and duplicate entries are merged.
  4. **Observer Pattern** notifies active views and triggers shopping list regeneration.
* **Alternative/Exception Flows**:
  * *2a. Invalid Quantity (<= 0) or Incompatible Unit*: Validation rejects the input immediately, displaying an inline error message without modifying state.
* **Postconditions**: Pantry inventory state is updated, merged, and saved locally.
* **Related Feature(s)**: F02 (Manage Pantry Inventory), F11 (Generate Shopping List)

---

#### UC04 — Generate AI Recipe
* **Use Case ID**: UC04
* **Use Case Name**: Generate AI Recipe
* **Actor(s)**: User, AI Model Service, ConstraintValidator
* **Goal**: Create a custom, allergen-safe recipe on-the-fly using generative AI prompts.
* **Preconditions**: User profile is configured with active allergen blacklists.
* **Trigger**: User enters a custom recipe prompt in the generator bar and clicks submit.
* **Main Success Scenario**:
  1. User enters a prompt (*"High-protein vegan pasta under 20 mins"*).
  2. `AgentController.generateRecipe()` dispatches prompt and profile constraints to `LLMClient`.
  3. LLM returns a structured JSON recipe payload matching the strict schema.
  4. `ConstraintValidator` checks ingredients against user allergen blacklists.
  5. Recipe preview dialog opens, allowing the user to save the recipe to local storage.
* **Alternative/Exception Flows**:
  * *3a. Malformed JSON*: Triggers an internal automated repair prompt to the LLM (max 1 retry).
  * *4a. Constraint Violation*: Error feedback is looped back into the agent to regenerate a safe variant (max 3 tries).
* **Postconditions**: A custom recipe is generated, validated, and displayed in a preview modal.
* **Related Feature(s)**: F04 (Generate AI Recipe), F01 (Manage Dietary Profile)

---

#### UC05 — Generate Weekly Meal Plan
* **Use Case ID**: UC05
* **Use Case Name**: Generate Weekly Meal Plan
* **Actor(s)**: User (Student / Busy Professional), AI Model Service (OpenAI GPT-4o / Ollama), ConstraintValidator
* **Goal**: Automatically generate a balanced, personalized weekly meal plan that respects budget limits, dietary restrictions, and utilizes current pantry items.
* **Preconditions**: 
  * User profile exists with dietary restrictions, calorie targets, and weekly budget limits.
  * Pantry inventory contains current stock items.
* **Trigger**: User clicks the "Generate Plan" button in the Plan Tab after configuring parameters.
* **Main Success Scenario**:
  1. User selects planning parameters (duration, meals per day, planning strategy) and clicks *Generate Plan*.
  2. `AgentController` calls `AgentFacade.generatePlan()`, instantiating the selected algorithm via the **Strategy Pattern**.
  3. The ReAct agent execution loop invokes `PantryTool` and `RecipeSearchTool` to query inventory and local recipes.
  4. The LLM selects recipes via tool calls, constructing a multi-day candidate `MealPlan` using the **Composite Pattern**.
  5. `ConstraintValidator` runs deterministic checks against macro totals and allergen blacklists.
  6. The validated plan is saved, and registered observers (**Observer Pattern**) trigger UI updates on `CalendarView`.
* **Alternative/Exception Flows**:
  * *4a. LLM Network Timeout / Ollama Unreachable*: System attempts one automatic retry, then prompts the user to switch to the local rule-based fallback generator.
  * *5a. Constraint Violation Detected*: Validation failure details are fed back into the agent loop for automated self-correction before rendering.
* **Postconditions**: 
  * A structured `MealPlan` is created, validated, persisted, and rendered onto the weekly calendar UI.
  * Shopping list and nutrition totals are automatically recalculated.
* **Related Feature(s)**: F05 (Generate Weekly Meal Plan), F03 (Recommend Recipes from Pantry)

---

#### UC06 — Modify Plan via Natural Language
* **Use Case ID**: UC06
* **Use Case Name**: Modify Plan via Natural Language
* **Actor(s)**: User, AI Model Service, CommandHistory
* **Goal**: Modify an existing meal slot dynamically using natural language chat commands while maintaining undo/redo capabilities.
* **Preconditions**: An active `MealPlan` currently exists on the calendar view.
* **Trigger**: User types a text instruction into the agent chat box and hits enter.
* **Main Success Scenario**:
  1. User types a modification instruction (*"Swap Tuesday dinner for a quick 15-minute meal"*).
  2. `AgentController.modifyPlan()` passes the instruction and current plan context to `LLMClient`.
  3. The LLM parses the natural language request and returns a structured replacement command payload.
  4. System instantiates a `ReplaceMealCommand` (**Command Pattern**).
  5. `ConstraintValidator` verifies the new meal satisfies all user safety rules.
  6. `CommandHistory` executes the command, updates the `MealPlan`, and observers automatically refresh the calendar view.
* **Alternative/Exception Flows**:
  * *3a. Ambiguous Instruction*: The system prompts the user with a clarifying question (*"Which day's dinner would you like to swap?"*).
  * *5a. Safety Violation*: Command execution is aborted, the meal plan remains unchanged, and an explanatory refusal message is displayed.
* **Postconditions**: 
  * The target meal slot is successfully modified.
  * The action is pushed to `CommandHistory` to support undo/redo.
* **Related Feature(s)**: F06 (Modify Plan via Natural Language), F08 (Scale Recipe Servings)

---

#### UC11 — Generate Shopping List
* **Use Case ID**: UC11
* **Use Case Name**: Generate Shopping List
* **Actor(s)**: User, ShoppingListGenerator (Observer), MealPlan, Pantry
* **Goal**: Automatically compute missing ingredients by comparing active meal plan requirements against current pantry inventory using the Observer pattern.
* **Preconditions**: 
  * An active `MealPlan` exists for the current week.
  * Current `Pantry` inventory stock is populated.
* **Trigger**: User opens the Shopping List Tab and clicks "Generate Shopping List".
* **Main Success Scenario**:
  1. User clicks *Generate Shopping List*.
  2. `AgentController` calls `AgentFacade.generateShoppingList()`.
  3. `ShoppingListGenerator` (acting as an **Observer**) receives notification state change from the active `MealPlan` and `Pantry` subjects.
  4. Generator computes the difference between required meal ingredients and available pantry stock using the **Composite Pattern** aggregation.
  5. Missing quantities are merged by category, unit-normalized, and estimated costs are calculated.
  6. Consolidated shopping list is rendered on the UI.
* **Alternative/Exception Flows**:
  * *4a. Pantry Inventory is Empty*: System treats all meal plan ingredients as missing and generates a full list of required items with an advisory warning.
* **Postconditions**: A consolidated, priced `ShoppingList` is generated, persisted locally, and displayed on screen.
* **Related Feature(s)**: F11 (Generate Shopping List), F05 (Generate Weekly Meal Plan), F02 (Manage Pantry Inventory)

--- 

### Summary of Supporting Use Cases

While detailed step-by-step specifications are documented above for the five major system-driving use cases (`UC02`, `UC04`, `UC05`, `UC06`, and `UC11`), the remaining system interactions are summarized in the table below. 

> **Note**: The following use cases represent supporting, auxiliary, or CRUD-based features that complement the core architecture. They are implemented using standard domain logic or straightforward UI-to-model data bindings without invoking complex multi-step ReAct agent loops.

| Use Case ID | Use Case Name | Primary Actor | Primary Purpose / Goal | Related Feature(s) |
| :--- | :--- | :--- | :--- | :--- |
| **UC01** | Manage Dietary Profile | User | Updates user allergen blacklists, macro targets, and dietary preferences. | F01 |
| **UC03** | Recommend Recipes from Pantry | User, AI Model | Suggests high-affinity recipes utilizing items currently tracked in inventory. | F03 |
| **UC07** | Suggest Ingredient Substitution | User, AI Model | Recommends safe, context-aware alternative ingredients for active recipes. | F07 |
| **UC08** | Scale Recipe Servings | User, CommandHistory | Dynamically recalculates ingredient quantities based on user serving size adjustments. | F08 |
| **UC09** | Analyze Nutrition & Commentary | User, AI Model | Aggregates weekly macros via the Composite pattern and displays AI commentary. | F09 |
| **UC10** | Plan Expiring Leftovers | User, AI Model | Creates targeted zero-waste meal snippets for inventory items nearing expiry. | F10 |
| **UC12** | Rate & Favorite Recipes | User | Saves user recipe ratings and favorites locally for quick filtering. | F12 |

 ## 2.3 Sequence Diagrams

### 3.3 System Sequence Diagrams

The matrix below summarizes the dynamic message passing, design pattern collaborations, and core feature mappings for the system's major use cases.

| Sequence ID | Use Case Target | Core Collaboration & Design Pattern | Mapped Feature(s) | Sequence Diagram |
| :--- | :--- | :--- | :--- | :--- |
| **SD02** | **UC02**<br>(Manage Pantry) | `MainController` -> `AgentFacade` -> `AddPantryItemCommand` -> `Pantry` model.<br><br>*Pattern: Command Pattern* | **F02** | [`diagrams/MealMind_Seq_UC02`](diagrams/MealMind_Seq_UC02.png) |
| **SD04** | **UC04**<br>(Generate AI Recipe) | `AgentController` -> `LLMClient` -> `ConstraintValidator`. Handles automated JSON repair and safety checks.<br><br>*Pattern: Strategy / Validation Loop* | **F04** | [`diagrams/MealMind_Seq_UC04`](diagrams/MealMind_Seq_UC04.png) |
| **SD05** | **UC05**<br>(Generate Meal Plan) | `AgentFacade` -> `MealPlanningAgent` -> `PlanningStrategy` -> `Composite` plan hierarchy -> `ConstraintValidator`.<br><br>*Pattern: Strategy & Composite Patterns* | **F05**, **F03** | [`diagrams/MealMind_Seq_UC05`](diagrams/MealMind_Seq_UC05.png) |
| **SD06** | **UC06**<br>(Modify via NL) | `AgentController` -> `LLMClient` -> `ReplaceMealCommand` -> `CommandHistory` with undo/redo support.<br><br>*Pattern: Command Pattern* | **F06**, **F08** | [`diagrams/MealMind_Seq_UC06`](diagrams/MealMind_Seq_UC06.png) |
| **SD11** | **UC11**<br>(Generate Shopping List) | `MealPlan` & `Pantry` notify `ShoppingListGenerator` via observer subscription to compute missing items.<br><br>*Pattern: Observer Pattern* | **F11**, **F05**, **F02** | [`diagrams/MealMind_Seq_UC11`](diagrams/MealMind_Seq_UC11.png) |
