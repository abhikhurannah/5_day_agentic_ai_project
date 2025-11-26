## 🧠 How PantryChef’s Agents Work

PantryChef is a **multi-agent system** built around a single LLM-powered root agent plus two specialized tool-agents:

- **Root Agent (PantryChef ADK Agent)** – brain / coordinator
- **Meal Planner Tool-Agent** – plans recipes from pantry
- **Shopping Tool-Agent** – generates and “places” grocery orders

Even though the meal planner and shopping logic are implemented as Python functions (`meal_planner_tool`, `mock_place_order`), we treat them as **tools used by the root agent**. Together, they behave like a small multi-agent team.

---

### 1. Overall Flow

High-level flow for a typical request:

1. **User**: “Plan dinner using my pantry and give me a shopping list.”
2. **Root Agent** (LLM via Gemini + ADK):
   - Reads the user message and the current **session state** (pantry items, previous context).
   - Decides to call the **Meal Planner Tool-Agent** with:
     - `pantry_items` – list from `pantry.json`
     - `constraints` – e.g. max cooking time, servings, diet
3. **Meal Planner Tool-Agent** (`meal_planner_tool`):
   - Uses **Gemini** (or mock) to generate `recipes`:
     - `title`
     - `ingredients`
     - `steps`
     - `time_minutes`
     - `servings`
     - `missing_items` (ingredients not in pantry)
   - Returns this structured JSON to the root agent.
4. **Root Agent**:
   - Reads the recipes and identifies `missing_items`.
   - Decides to call the **Shopping Tool-Agent** with:
     - `shopping_list` = all unique missing items.
5. **Shopping Tool-Agent** (`mock_place_order`):
   - Simulates placing an order and returns:
     - `status`
     - `order_id`
     - `eta_minutes`
     - `items`
6. **Root Agent**:
   - Combines recipes + shopping order into a final, human-readable response for the user.

---

### 2. Role of Each Agent

#### 🧠 Root Agent (PantryChef ADK Agent)

- Implemented using **Google ADK** as an `Agent`:
  - Has a **name**, **description**, and **instruction** (system prompt).
  - Uses a **Gemini model** (e.g. `gemini-2.5-flash`).
  - Has access to tools:
    - `adk_meal_planner_tool`
    - `adk_shopping_tool`
- Responsibilities:
  - Understand the user’s request (natural language).
  - Choose **which tool to call** and in **what order**.
  - Interpret tool outputs and synthesize a clean answer.
  - Update / read **session state** via the `Runner` + `InMemorySessionService`.

> Conceptually, this is the “manager agent” that delegates work to more specialized sub-agents (tools).

---

#### 🍳 Meal Planner Tool-Agent

- Python function: `meal_planner_tool(pantry_items, constraints)`.
- In mock mode:
  - Uses `mock_generate_recipes` to return a few example recipes.
- In real mode:
  - Calls **Gemini** with a prompt that includes:
    - Pantry items
    - Constraints (time, diet, servings)
  - Asks for a **JSON response** with:
    - `recipes: [ { title, ingredients, steps, time_minutes, servings, missing_items } ]`
- Behaves like a **specialized “chef” agent**:
  - Focuses only on recipe generation and ingredient analysis.
  - Does **not** worry about ordering or user conversation.

---

#### 🛒 Shopping Tool-Agent

- Python function: `mock_place_order(shopping_list)`.
- Behaves like a **“shopping assistant” agent**:
  - Takes a list of items that are **not** in the pantry.
  - Returns a fake order confirmation:
    - `status`
    - `order_id`
    - `eta_minutes`
    - `items`
- In a future version, this could be swapped with a real **OpenAPI connector** to:
  - Grocery APIs
  - Delivery services
  - Trello/Notion lists (for a shopping board)

---

### 3. Sessions & Memory

PantryChef tracks **state** in two main ways:

1. **Pantry state** (`pantry.json`):
   - Persistent file storing current pantry items:
     ```json
     {
       "items": ["eggs", "rice", "paneer", "spinach", "onion"]
     }
     ```
   - Managed via the `Pantry` class (add, remove, clear, list).

2. **ADK Session State**:
   - Managed by `InMemorySessionService` with:
     - `APP_NAME`
     - `USER_ID`
     - `SESSION_ID`
   - Keeps chat context and any agent-level memory for that user/session.
   - Used by `Runner` to maintain continuity across multiple calls.

Because of this, PantryChef can:

- Remember which ingredients you’ve added.
- Use the same session to refine recipes (e.g. “Less spicy” or “15 minutes only”).

---

### 4. Multi-Agent Pattern

Even though there is **one LLM agent** in code (the root agent), the system still demonstrates **multi-agent behavior**:

- **Root Agent (LLM manager)**
- **Tool-Agent 1: Meal Planner**
- **Tool-Agent 2: Shopping**

The **A2A-like behavior** happens when:

1. Root agent → calls Meal Planner Tool
2. Root agent → takes that output → calls Shopping Tool
3. Tools return structured data → root agent composes a final answer.

This pattern is easy to extend:

- Add a **Nutrition Agent** tool
- Add a **Cook Agent** (with timers & long-running operations)
- Add a **Budget Agent** (optimize costs across shops)

Each new capability is just a new **tool-agent** that the root agent can call.

---

### 5. Where to Look in the Code

- **Pantry & state**:
  - `Pantry` class (pantry.json handling)
- **Meal Planner logic**:
  - `meal_planner_tool`
  - `mock_generate_recipes`
- **Shopping logic**:
  - `aggregate_missing_items`
  - `mock_place_order`
- **ADK Agent & Runner**:
  - `Agent(...)` creation
  - `Runner(...)` and `InMemorySessionService`
  - Optional `runner.run(...)` example at the end

With these pieces together, PantryChef is a **small but complete agentic application**:
- Multi-agent logic
- Tools
- Sessions & memory
- (Optional) ADK runner for real agent orchestration.
