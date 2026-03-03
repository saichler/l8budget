# PRD: L8Budget — Household Budget Management

## Overview

L8Budget is a household budget management application built on the Layer 8 ecosystem. It provides families and individuals with tools to track accounts, categorize transactions, plan monthly budgets, manage recurring expenses, and set savings goals — all with real-time analytics and both desktop and mobile interfaces.

The system follows the same architectural patterns as l8erp: Go backend services with protobuf-defined types, PostgreSQL persistence via the Layer 8 ORM, and a configuration-driven JavaScript frontend using the l8ui component library.

---

## Module Definition

| Property | Value |
|----------|-------|
| Module prefix | `Bud` |
| Proto package | `bud` |
| Go package path | `./types/bud` |
| ServiceArea | `byte(10)` |
| UI namespace | `BUD` |
| Section name | `budget` |

---

## Prime Objects

Applying the Prime Object test (independence, own lifecycle, direct query need, no parent ID dependency):

| # | Prime Object | Service Name | Description |
|---|-------------|-------------|-------------|
| 1 | `BudAccount` | `Account` | Bank accounts, credit cards, cash wallets |
| 2 | `BudCategory` | `Category` | Hierarchical income/expense categories |
| 3 | `BudPayee` | `Payee` | Merchants, employers, vendors |
| 4 | `BudTransaction` | `Txn` | Individual income/expense/transfer records |
| 5 | `BudBudget` | `Budget` | Monthly/weekly budget plan with line allocations |
| 6 | `BudRecurring` | `Recurring` | Recurring transaction templates (rent, subscriptions, salary) |
| 7 | `BudGoal` | `Goal` | Savings targets and debt payoff goals |

All ServiceName values are <= 10 characters per the maintainability rule.

### Child Types (embedded via `repeated` in parent, no own service)

| Child Type | Parent | Description |
|-----------|--------|-------------|
| `BudBudgetLine` | `BudBudget` | Per-category allocation within a budget period |
| `BudGoalMilestone` | `BudGoal` | Tracked milestone/checkpoint within a goal |
| `BudRecurringSplit` | `BudRecurring` | Split allocation when a recurring rule touches multiple categories |

---

## Protobuf Design

### File Organization

| Proto File | Contents |
|-----------|----------|
| `bud-common.proto` | Shared enums: AccountType, CategoryType, TransactionType, BudgetPeriod, GoalStatus, RecurringFrequency |
| `bud-accounts.proto` | BudAccount, BudAccountList |
| `bud-categories.proto` | BudCategory, BudCategoryList |
| `bud-payees.proto` | BudPayee, BudPayeeList |
| `bud-transactions.proto` | BudTransaction, BudTransactionList |
| `bud-budgets.proto` | BudBudget, BudBudgetList, BudBudgetLine (child) |
| `bud-recurring.proto` | BudRecurring, BudRecurringList, BudRecurringSplit (child) |
| `bud-goals.proto` | BudGoal, BudGoalList, BudGoalMilestone (child) |

### Enum Definitions (zero-value convention)

```protobuf
enum BudAccountType {
  BUD_ACCOUNT_TYPE_UNSPECIFIED = 0;
  BUD_ACCOUNT_TYPE_CHECKING = 1;
  BUD_ACCOUNT_TYPE_SAVINGS = 2;
  BUD_ACCOUNT_TYPE_CREDIT_CARD = 3;
  BUD_ACCOUNT_TYPE_CASH = 4;
  BUD_ACCOUNT_TYPE_INVESTMENT = 5;
  BUD_ACCOUNT_TYPE_LOAN = 6;
  BUD_ACCOUNT_TYPE_OTHER = 7;
}

enum BudCategoryType {
  BUD_CATEGORY_TYPE_UNSPECIFIED = 0;
  BUD_CATEGORY_TYPE_EXPENSE = 1;
  BUD_CATEGORY_TYPE_INCOME = 2;
  BUD_CATEGORY_TYPE_TRANSFER = 3;
}

enum BudTransactionType {
  BUD_TRANSACTION_TYPE_UNSPECIFIED = 0;
  BUD_TRANSACTION_TYPE_EXPENSE = 1;
  BUD_TRANSACTION_TYPE_INCOME = 2;
  BUD_TRANSACTION_TYPE_TRANSFER = 3;
  BUD_TRANSACTION_TYPE_REFUND = 4;
  BUD_TRANSACTION_TYPE_ADJUSTMENT = 5;
}

enum BudTransactionStatus {
  BUD_TRANSACTION_STATUS_UNSPECIFIED = 0;
  BUD_TRANSACTION_STATUS_PENDING = 1;
  BUD_TRANSACTION_STATUS_CLEARED = 2;
  BUD_TRANSACTION_STATUS_RECONCILED = 3;
  BUD_TRANSACTION_STATUS_VOID = 4;
}

enum BudBudgetPeriod {
  BUD_BUDGET_PERIOD_UNSPECIFIED = 0;
  BUD_BUDGET_PERIOD_WEEKLY = 1;
  BUD_BUDGET_PERIOD_BIWEEKLY = 2;
  BUD_BUDGET_PERIOD_MONTHLY = 3;
  BUD_BUDGET_PERIOD_QUARTERLY = 4;
  BUD_BUDGET_PERIOD_YEARLY = 5;
}

enum BudRecurringFrequency {
  BUD_RECURRING_FREQUENCY_UNSPECIFIED = 0;
  BUD_RECURRING_FREQUENCY_DAILY = 1;
  BUD_RECURRING_FREQUENCY_WEEKLY = 2;
  BUD_RECURRING_FREQUENCY_BIWEEKLY = 3;
  BUD_RECURRING_FREQUENCY_MONTHLY = 4;
  BUD_RECURRING_FREQUENCY_QUARTERLY = 5;
  BUD_RECURRING_FREQUENCY_YEARLY = 6;
}

enum BudGoalStatus {
  BUD_GOAL_STATUS_UNSPECIFIED = 0;
  BUD_GOAL_STATUS_ACTIVE = 1;
  BUD_GOAL_STATUS_PAUSED = 2;
  BUD_GOAL_STATUS_COMPLETED = 3;
  BUD_GOAL_STATUS_CANCELLED = 4;
}

enum BudGoalType {
  BUD_GOAL_TYPE_UNSPECIFIED = 0;
  BUD_GOAL_TYPE_SAVINGS = 1;
  BUD_GOAL_TYPE_DEBT_PAYOFF = 2;
  BUD_GOAL_TYPE_EMERGENCY_FUND = 3;
  BUD_GOAL_TYPE_PURCHASE = 4;
  BUD_GOAL_TYPE_CUSTOM = 5;
}
```

### Prime Object Messages

#### BudAccount
```protobuf
message BudAccount {
  string account_id = 1;
  string name = 2;
  BudAccountType account_type = 3;
  string institution = 4;         // bank/institution name
  string account_number = 5;      // last 4 digits only, masked
  erp.Money current_balance = 6;
  erp.Money opening_balance = 7;
  string currency_code = 8;       // default currency for this account
  bool is_active = 9;
  string notes = 10;
  string color = 11;              // hex color for UI identification
  string icon = 12;               // icon identifier for UI
  int64 opened_date = 13;         // when the account was opened
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}
```

#### BudCategory
```protobuf
message BudCategory {
  string category_id = 1;
  string name = 2;
  BudCategoryType category_type = 3;
  string parent_category_id = 4;   // for hierarchical categories (self-ref by ID)
  string icon = 5;
  string color = 6;
  bool is_active = 7;
  int32 sort_order = 8;
  string notes = 9;
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}
```

#### BudPayee
```protobuf
message BudPayee {
  string payee_id = 1;
  string name = 2;
  string default_category_id = 3;  // ref to BudCategory by ID
  string notes = 4;
  bool is_active = 5;
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}
```

#### BudTransaction
```protobuf
message BudTransaction {
  string transaction_id = 1;
  string account_id = 2;          // ref to BudAccount by ID
  BudTransactionType transaction_type = 3;
  BudTransactionStatus status = 4;
  int64 transaction_date = 5;
  erp.Money amount = 6;
  string category_id = 7;         // ref to BudCategory by ID
  string payee_id = 8;            // ref to BudPayee by ID
  string description = 9;
  string reference_number = 10;   // check number, confirmation code
  string transfer_account_id = 11; // dest account for transfers (ref to BudAccount by ID)
  string recurring_id = 12;       // ref to BudRecurring by ID (if auto-generated)
  repeated string tags = 13;
  string notes = 14;
  bool is_reconciled = 15;
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}
```

#### BudBudget (with embedded BudBudgetLine child)
```protobuf
message BudBudget {
  string budget_id = 1;
  string name = 2;
  BudBudgetPeriod period = 3;
  int64 start_date = 4;
  int64 end_date = 5;
  erp.Money total_planned = 6;    // sum of all line allocations
  erp.Money total_actual = 7;     // sum of actual spending (computed)
  string notes = 8;
  bool is_active = 9;
  repeated BudBudgetLine lines = 20;  // embedded child
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}

// Child type — embedded in BudBudget, no own service
message BudBudgetLine {
  string line_id = 1;
  string category_id = 2;         // ref to BudCategory by ID
  erp.Money planned_amount = 3;
  erp.Money actual_amount = 4;    // computed from transactions
  string notes = 5;
}
```

#### BudRecurring (with embedded BudRecurringSplit child)
```protobuf
message BudRecurring {
  string recurring_id = 1;
  string name = 2;
  BudRecurringFrequency frequency = 3;
  string account_id = 4;          // ref to BudAccount by ID
  string payee_id = 5;            // ref to BudPayee by ID
  string category_id = 6;         // primary category (ref by ID)
  BudTransactionType transaction_type = 7;
  erp.Money amount = 8;
  int64 start_date = 9;
  int64 end_date = 10;            // 0 = no end
  int64 next_due_date = 11;
  int32 day_of_month = 12;        // for monthly: which day (1-31)
  int32 day_of_week = 13;         // for weekly: which day (0=Sun..6=Sat)
  bool is_active = 14;
  bool auto_post = 15;            // auto-create transaction when due
  string notes = 16;
  repeated BudRecurringSplit splits = 20;  // embedded child for split allocations
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}

// Child type — embedded in BudRecurring, no own service
message BudRecurringSplit {
  string split_id = 1;
  string category_id = 2;         // ref to BudCategory by ID
  erp.Money amount = 3;
  string notes = 4;
}
```

#### BudGoal (with embedded BudGoalMilestone child)
```protobuf
message BudGoal {
  string goal_id = 1;
  string name = 2;
  BudGoalType goal_type = 3;
  BudGoalStatus status = 4;
  erp.Money target_amount = 5;
  erp.Money current_amount = 6;
  string account_id = 7;          // linked savings/debt account (ref by ID)
  int64 target_date = 8;
  int64 start_date = 9;
  string color = 10;
  string icon = 11;
  string notes = 12;
  repeated BudGoalMilestone milestones = 20;  // embedded child
  map<string, string> custom_fields = 30;
  erp.AuditInfo audit_info = 31;
  reserved 32 to 50;
}

// Child type — embedded in BudGoal, no own service
message BudGoalMilestone {
  string milestone_id = 1;
  string name = 2;
  erp.Money amount = 3;
  int64 target_date = 4;
  bool is_reached = 5;
}
```

### List Types (all follow the convention)
```protobuf
message BudAccountList {
  repeated BudAccount list = 1;
  l8api.L8MetaData metadata = 2;
}
// Same pattern for BudCategoryList, BudPayeeList, BudTransactionList,
// BudBudgetList, BudRecurringList, BudGoalList
```

---

## Backend Service Architecture

### Directory Structure

```
go/
  erp/
    bud/
      accounts/
        BudAccountService.go
        BudAccountServiceCallback.go
      categories/
        BudCategoryService.go
        BudCategoryServiceCallback.go
      payees/
        BudPayeeService.go
        BudPayeeServiceCallback.go
      transactions/
        BudTransactionService.go
        BudTransactionServiceCallback.go
      budgets/
        BudBudgetService.go
        BudBudgetServiceCallback.go
      recurring/
        BudRecurringService.go
        BudRecurringServiceCallback.go
      goals/
        BudGoalService.go
        BudGoalServiceCallback.go
    services/
      activate_bud.go
    common/              # reuse from l8erp or extract shared module
    ui/
      main.go            # type registration
      shared.go          # registerBudTypes()
  types/
    bud/                 # generated .pb.go files
```

### Service Pattern (per l8erp conventions)

Each service follows the identical pattern from l8erp:

```go
// BudAccountService.go
package accounts

const (
    ServiceName = "Account"
    ServiceArea = byte(10)
)

func Activate(creds, dbname string, vnic ifs.IVNic) {
    common.ActivateService[bud.BudAccount, bud.BudAccountList](common.ServiceConfig{
        ServiceName: ServiceName, ServiceArea: ServiceArea,
        PrimaryKey: "AccountId", Callback: newBudAccountServiceCallback(),
        Transactional: true,
    }, creds, dbname, vnic)
}
```

```go
// BudAccountServiceCallback.go
package accounts

func newBudAccountServiceCallback() ifs.IServiceCallback {
    return common.NewServiceCallback("BudAccount",
        func(e *bud.BudAccount) { common.GenerateID(&e.AccountId) },
        validate)
}

func validate(account *bud.BudAccount, vnic ifs.IVNic) error {
    common.ValidateRequired(account.Name, "Name")
    common.ValidateEnum(int32(account.AccountType), "AccountType",
        int32(bud.BudAccountType_BUD_ACCOUNT_TYPE_UNSPECIFIED))
    return nil
}
```

### Service Activation

```go
// activate_bud.go
package services

func ActivateBudServices(creds, dbname string, vnic ifs.IVNic) {
    accounts.Activate(creds, dbname, vnic)
    categories.Activate(creds, dbname, vnic)
    payees.Activate(creds, dbname, vnic)
    transactions.Activate(creds, dbname, vnic)
    budgets.Activate(creds, dbname, vnic)
    recurring.Activate(creds, dbname, vnic)
    goals.Activate(creds, dbname, vnic)
}
```

### Type Registration

```go
// shared.go
func registerBudTypes(resources ifs.IResources) {
    common.RegisterType[bud.BudAccount, bud.BudAccountList](resources, "AccountId")
    common.RegisterType[bud.BudCategory, bud.BudCategoryList](resources, "CategoryId")
    common.RegisterType[bud.BudPayee, bud.BudPayeeList](resources, "PayeeId")
    common.RegisterType[bud.BudTransaction, bud.BudTransactionList](resources, "TransactionId")
    common.RegisterType[bud.BudBudget, bud.BudBudgetList](resources, "BudgetId")
    common.RegisterType[bud.BudRecurring, bud.BudRecurringList](resources, "RecurringId")
    common.RegisterType[bud.BudGoal, bud.BudGoalList](resources, "GoalId")
}
```

---

## UI Architecture

### Sub-module Organization

| Sub-module | Services | Description |
|-----------|----------|-------------|
| `accounts` | Account, Payee | Account and payee management |
| `transactions` | Transaction | Transaction register and tracking |
| `budgets` | Budget, Category | Budget planning and category setup |
| `planning` | Recurring, Goal | Recurring rules and savings goals |

### File Structure

```
go/erp/ui/web/
  budget/
    budget-config.js          # Layer8ModuleConfigFactory.create()
    budget-section-config.js  # Layer8SectionConfigs.register('budget', {...})
    budget-init.js            # Layer8DModuleFactory.create()
    budget.css                # module accent color and styles
    accounts/
      accounts-enums.js       # BUD_ACCOUNT_TYPE, BUD_ACCOUNT_TYPE_CLASSES
      accounts-columns.js     # BudAccount, BudPayee columns
      accounts-forms.js       # BudAccount, BudPayee forms
    transactions/
      transactions-enums.js   # BUD_TRANSACTION_TYPE, BUD_TRANSACTION_STATUS, etc.
      transactions-columns.js # BudTransaction columns
      transactions-forms.js   # BudTransaction form
    budgets/
      budgets-enums.js        # BUD_BUDGET_PERIOD, BUD_CATEGORY_TYPE
      budgets-columns.js      # BudBudget, BudCategory columns
      budgets-forms.js        # BudBudget form (with inline BudBudgetLine table), BudCategory form
    planning/
      planning-enums.js       # BUD_RECURRING_FREQUENCY, BUD_GOAL_STATUS, BUD_GOAL_TYPE
      planning-columns.js     # BudRecurring, BudGoal columns
      planning-forms.js       # BudRecurring form (with inline split table), BudGoal form (with inline milestones)
  sections/
    budget.html               # section placeholder -> Layer8SectionGenerator
  js/
    reference-registry-bud.js # all BudXxx reference entries
```

### Module Config (`budget-config.js`)

```javascript
window.BudgetConfig = Layer8ModuleConfigFactory.create({
    namespace: 'BUD',
    serviceArea: 10,
    subModules: {
        accounts: {
            services: [
                { key: 'accounts', label: 'Accounts', model: 'BudAccount', endpoint: 'Account' },
                { key: 'payees', label: 'Payees', model: 'BudPayee', endpoint: 'Payee' }
            ]
        },
        transactions: {
            services: [
                { key: 'transactions', label: 'Transactions', model: 'BudTransaction', endpoint: 'Txn' }
            ]
        },
        budgets: {
            services: [
                { key: 'budgets', label: 'Budgets', model: 'BudBudget', endpoint: 'Budget' },
                { key: 'categories', label: 'Categories', model: 'BudCategory', endpoint: 'Category' }
            ]
        },
        planning: {
            services: [
                { key: 'recurring', label: 'Recurring', model: 'BudRecurring', endpoint: 'Recurring' },
                { key: 'goals', label: 'Goals', model: 'BudGoal', endpoint: 'Goal' }
            ]
        }
    }
});
```

### Module Init (`budget-init.js`)

```javascript
(function() {
    'use strict';
    Layer8DModuleFactory.create({
        namespace: 'BUD',
        defaultModule: 'accounts',
        defaultService: 'accounts',
        sectionSelector: 'accounts',   // MUST match defaultModule
        initializerName: 'initializeBUD',
        requiredNamespaces: ['Accounts', 'Transactions', 'Budgets', 'Planning']
    });
})();
```

### Section Initializer (in `sections.js`)

```javascript
budget: () => {
    if (typeof initializeBUD === 'function') {
        initializeBUD();
    }
},
```

### Reference Registry (`reference-registry-bud.js`)

```javascript
(function() {
    'use strict';
    const ref = Layer8RefFactory;
    Layer8DReferenceRegistry.register({
        ...ref.simple('BudAccount', 'accountId', 'name', 'Account'),
        ...ref.simple('BudCategory', 'categoryId', 'name', 'Category'),
        ...ref.simple('BudPayee', 'payeeId', 'name', 'Payee'),
        ...ref.simple('BudRecurring', 'recurringId', 'name', 'Recurring Rule'),
    });
})();
```

### app.html Additions

```html
<!-- In <head> -->
<link rel="stylesheet" href="budget/budget.css">

<!-- After other reference registries -->
<script src="js/reference-registry-bud.js"></script>

<!-- BUD Module -->
<script src="budget/budget-config.js"></script>
<script src="budget/budget-section-config.js"></script>
<script src="budget/accounts/accounts-enums.js"></script>
<script src="budget/accounts/accounts-columns.js"></script>
<script src="budget/accounts/accounts-forms.js"></script>
<script src="budget/transactions/transactions-enums.js"></script>
<script src="budget/transactions/transactions-columns.js"></script>
<script src="budget/transactions/transactions-forms.js"></script>
<script src="budget/budgets/budgets-enums.js"></script>
<script src="budget/budgets/budgets-columns.js"></script>
<script src="budget/budgets/budgets-forms.js"></script>
<script src="budget/planning/planning-enums.js"></script>
<script src="budget/planning/planning-columns.js"></script>
<script src="budget/planning/planning-forms.js"></script>
<script src="budget/budget-init.js"></script>
```

---

## l8ui View Types Per Service

Each service's default view and available alternative views:

| Service | Default View | Alt Views | Notes |
|---------|-------------|-----------|-------|
| BudAccount | `table` | `widget` | Widget cards showing balance per account |
| BudPayee | `table` | — | Simple list |
| BudTransaction | `table` | `chart`, `calendar` | Chart for spending trends; calendar for transaction dates |
| BudBudget | `table` | `chart` | Bar chart: planned vs actual per category |
| BudCategory | `tree` | — | TreeGrid for hierarchical display via `parentCategoryId` |
| BudRecurring | `table` | `calendar` | Calendar showing upcoming due dates |
| BudGoal | `table` | `widget`, `chart` | Widget cards showing progress; chart for goal tracking |

These are configured in `budget-section-config.js` via the `viewConfig` property on each service definition, leveraging `Layer8DViewFactory` and `Layer8DViewSwitcher`.

---

## Budget vs Actual Analytics

The budget sub-module's chart view provides the core budgeting insight: planned allocation vs actual spend per category. This uses `Layer8DChart` with:

- **Type**: Grouped bar chart
- **X-axis**: Category names (from budget lines)
- **Y-axis**: Dollar amounts
- **Series**: Planned (theme primary color) vs Actual (green/red based on over/under)
- **Colors**: `Layer8DChart.getThemePalette()` for theme compliance

---

## Mobile Parity

Every desktop feature has a mobile equivalent per the mobile-parity rule:

### Mobile File Structure

```
go/erp/ui/web/m/
  sections/
    budget.html
  js/
    budget/
      accounts/
        accounts-enums.js
        accounts-columns.js
        accounts-forms.js
      transactions/
        transactions-enums.js
        transactions-columns.js
        transactions-forms.js
      budgets/
        budgets-enums.js
        budgets-columns.js
        budgets-forms.js
      planning/
        planning-enums.js
        planning-columns.js
        planning-forms.js
    layer8m-reference-registry-bud.js
    layer8m-nav-config-bud.js
```

### Mobile Navigation Config

```javascript
LAYER8M_NAV_CONFIG.budget = {
    label: 'Budget',
    icon: 'wallet',
    modules: {
        accounts:     { label: 'Accounts',     services: ['accounts', 'payees'] },
        transactions: { label: 'Transactions',  services: ['transactions'] },
        budgets:      { label: 'Budgets',       services: ['budgets', 'categories'] },
        planning:     { label: 'Planning',      services: ['recurring', 'goals'] }
    }
};
```

---

## CSS Theme (`budget.css`)

Module accent color using `--layer8d-*` tokens per the l8ui theme compliance rule:

```css
:root {
    --bud-accent: #10b981;
    --bud-accent-dark: #059669;
}

.budget-header {
    background: linear-gradient(135deg, var(--bud-accent), var(--bud-accent-dark));
}

.budget-module-tab.active {
    color: var(--bud-accent);
    border-bottom-color: var(--bud-accent);
}

/* Status colors for goal/transaction states */
.bud-status-active { color: var(--layer8d-success); }
.bud-status-pending { color: var(--layer8d-warning); }
.bud-status-void { color: var(--layer8d-error); }
.bud-status-completed { color: var(--layer8d-primary); }
.bud-status-paused { color: var(--layer8d-text-muted); }
```

No `[data-theme="dark"]` blocks — dark mode is handled centrally by `layer8d-theme.css`.

---

## Mock Data

### Store Additions (`store.go`)

```go
// BUD Module - Phase 1: Foundation
BudAccountIDs   []string
BudCategoryIDs  []string
BudPayeeIDs     []string

// BUD Module - Phase 2: Transactions & Budgets
BudTransactionIDs []string
BudBudgetIDs      []string

// BUD Module - Phase 3: Planning
BudRecurringIDs []string
BudGoalIDs      []string
```

### Generator Files

| File | Generates | Count |
|------|----------|-------|
| `gen_bud_foundation.go` | BudAccount, BudCategory, BudPayee | 8, 25, 30 |
| `gen_bud_transactions.go` | BudTransaction | 200 |
| `gen_bud_budgets.go` | BudBudget (with BudBudgetLine children) | 6 |
| `gen_bud_planning.go` | BudRecurring (with splits), BudGoal (with milestones) | 15, 5 |

### Phase Ordering

```
Phase 1: Foundation (no deps)
  - BudAccount (8 accounts: 2 checking, 2 savings, 2 credit cards, 1 cash, 1 investment)
  - BudCategory (25 categories in hierarchy: Housing, Food, Transport, etc.)
  - BudPayee (30 payees: grocery stores, utilities, employers, etc.)

Phase 2: Transactions & Budgets (depend on Phase 1)
  - BudBudget + BudBudgetLine (6 monthly budgets with ~15 lines each referencing categories)
  - BudTransaction (200 transactions across 6 months referencing accounts, categories, payees)

Phase 3: Planning (depend on Phase 1)
  - BudRecurring + BudRecurringSplit (15 rules: rent, utilities, subscriptions, salary)
  - BudGoal + BudGoalMilestone (5 goals: emergency fund, vacation, car, holiday, debt payoff)
```

### Realistic Data Arrays (`data.go`)

```go
// Account names
var BudAccountNames = []string{"Primary Checking", "Joint Checking", "Emergency Savings",
    "Vacation Savings", "Visa Rewards", "Mastercard", "Cash Wallet", "Investment Account"}

// Category names (hierarchical)
var BudExpenseCategories = []string{"Housing", "Rent/Mortgage", "Utilities", "Insurance",
    "Food", "Groceries", "Dining Out", "Coffee", "Transportation", "Gas", "Car Payment",
    "Public Transit", "Entertainment", "Streaming", "Movies", "Health", "Medical",
    "Pharmacy", "Clothing", "Personal Care", "Education", "Gifts", "Charity",
    "Subscriptions", "Miscellaneous"}

var BudIncomeCategories = []string{"Salary", "Side Income", "Interest", "Dividends", "Refunds"}

// Payee names
var BudPayeeNames = []string{"Whole Foods", "Costco", "Target", "Amazon", "Netflix",
    "Spotify", "PG&E", "AT&T", "State Farm", "Shell Gas", "Starbucks", "Uber",
    "ACME Corp", "City Water", "Comcast", "Planet Fitness", "CVS Pharmacy",
    "Home Depot", "Trader Joe's", "Chipotle", "DoorDash", "Verizon", "Chase Bank",
    "Wells Fargo", "Fidelity", "IRS", "DMV", "Apple", "Google", "Microsoft"}
```

---

## Validation Rules

### BudAccount
- `Name`: required
- `AccountType`: required, not UNSPECIFIED
- `CurrencyCode`: required, default "USD"

### BudCategory
- `Name`: required
- `CategoryType`: required, not UNSPECIFIED
- `ParentCategoryId`: if set, must reference valid BudCategory

### BudPayee
- `Name`: required

### BudTransaction
- `AccountId`: required, must reference valid BudAccount
- `TransactionType`: required, not UNSPECIFIED
- `TransactionDate`: required
- `Amount`: required
- `TransferAccountId`: required when TransactionType is TRANSFER

### BudBudget
- `Name`: required
- `Period`: required, not UNSPECIFIED
- `StartDate`: required
- `EndDate`: required, must be after StartDate

### BudRecurring
- `Name`: required
- `Frequency`: required, not UNSPECIFIED
- `AccountId`: required
- `Amount`: required
- `StartDate`: required

### BudGoal
- `Name`: required
- `GoalType`: required, not UNSPECIFIED
- `TargetAmount`: required, must be > 0
- `StartDate`: required

---

## Implementation Phases

### Phase 1: Foundation
1. Create proto files and generate bindings
2. Implement Account, Category, Payee services + callbacks
3. Type registration in `ui/shared.go`
4. Service activation in `activate_bud.go`

### Phase 2: Core Services
5. Implement Transaction service + callback
6. Implement Budget service + callback (with BudBudgetLine child handling)
7. Implement Recurring service + callback (with BudRecurringSplit child handling)
8. Implement Goal service + callback (with BudGoalMilestone child handling)

### Phase 3: Desktop UI
9. Create `budget-config.js` (module config)
10. Create `budget-section-config.js` (section HTML generator)
11. Create sub-module enums, columns, forms (4 sub-modules)
12. Create `reference-registry-bud.js`
13. Create `budget-init.js`
14. Create `budget.css`
15. Create `sections/budget.html`
16. Update `sections.js` with initializer
17. Update `app.html` with all includes

### Phase 4: Mobile UI
18. Create mobile section HTML
19. Create mobile enums, columns, forms (mirror desktop)
20. Create mobile reference registry and nav config
21. Update `m/app.html` with all includes

### Phase 5: Mock Data
22. Add store ID slices
23. Add data arrays
24. Create generator files (foundation, transactions, budgets, planning)
25. Create phase orchestration files
26. Update `main.go` with phase calls
27. Build and verify: `go build ./tests/mocks/ && go vet ./tests/mocks/`

### Phase 6: Integration Tests
28. Create CRUD integration tests in `go/tests/` for all 7 services
29. Tests exercise POST/GET/PUT/DELETE via IVNic (system API, not internal functions)
30. Verify validation rules: required fields, enum-not-UNSPECIFIED checks, conditional requirements (e.g., `TransferAccountId` required for TRANSFER type)
31. Verify child type round-trip: POST a BudBudget with BudBudgetLine children, GET it back, assert children are present
32. Build and run: `go test ./tests/...`

### Phase 7: View Enhancements
33. Configure chart views for budgets (planned vs actual)
34. Configure widget views for accounts (balance cards) and goals (progress cards)
35. Configure calendar view for recurring (due dates) and transactions
36. Configure tree view for categories (hierarchical display)

---

## Cross-Reference Summary

All cross-references between Prime Objects use ID strings only (never embedded structs):

| From | Field | References |
|------|-------|-----------|
| BudTransaction | `accountId` | BudAccount |
| BudTransaction | `categoryId` | BudCategory |
| BudTransaction | `payeeId` | BudPayee |
| BudTransaction | `transferAccountId` | BudAccount |
| BudTransaction | `recurringId` | BudRecurring |
| BudBudgetLine | `categoryId` | BudCategory |
| BudRecurring | `accountId` | BudAccount |
| BudRecurring | `payeeId` | BudPayee |
| BudRecurring | `categoryId` | BudCategory |
| BudRecurringSplit | `categoryId` | BudCategory |
| BudGoal | `accountId` | BudAccount |
| BudPayee | `defaultCategoryId` | BudCategory |
| BudCategory | `parentCategoryId` | BudCategory (self-ref) |

---

## Demo Directory Sync

If a `go/demo/web/` directory is used for local development/testing, all source web files must be synced after any modification per the `demo-directory-sync` rule:

```bash
# After editing any file under the source web directory
cp go/erp/ui/web/<path> go/demo/web/<path>

# Verify sync
diff -rq go/erp/ui/web/budget/ go/demo/web/budget/
```

If this project does not use a demo directory, this section is not applicable and can be removed during implementation.

---

## Compliance Checklist

- [x] Proto enum zero values: all enums have `*_UNSPECIFIED = 0`
- [x] Proto list convention: all lists use `repeated X list = 1` + `l8api.L8MetaData metadata = 2`
- [x] Prime Object test: all 7 services pass the 4-criteria test
- [x] Child types embedded: BudBudgetLine, BudGoalMilestone, BudRecurringSplit as `repeated` in parent
- [x] Cross-references by ID only: no embedded Prime Object structs
- [x] ServiceName <= 10 chars: Account(7), Category(8), Payee(5), Txn(3), Budget(6), Recurring(9), Goal(4)
- [x] UI module integration checklist: section HTML, initializer, CSS, reference registry, app.html updates
- [x] Mobile parity: full mobile UI structure mirroring desktop
- [x] l8ui theme compliance: `--bud-accent` CSS variable for module accent, `--layer8d-*` tokens for status colors, no dark mode blocks
- [x] JS field names from protobuf: all field references verified against proto definitions
- [x] Mock data completeness: all 7 services have generators
- [x] File size < 500 lines: generators split by functional area
- [x] Reference registry completeness: all models used as `lookupModel` in forms are registered (BudAccount, BudCategory, BudPayee, BudRecurring)
- [x] Integration tests: CRUD tests in `go/tests/` for all 7 services via system API
- [x] Demo directory sync: documented for applicability
- [x] Plan written to `./plans/` directory per plan-approval-workflow rule
