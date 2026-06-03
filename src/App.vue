<template>
  <div class="app-container">
    <header class="header">
      <h1>Personal Finance Tracker</h1>
      <p>Track income, expenses, and budget in real time.</p>
    </header>

    <section class="panel form-panel">
      <h2>Add Transaction</h2>
      <form @submit.prevent="submitTransaction" class="transaction-form">
        <div class="form-row">
          <label for="desc">Description</label>
          <input id="desc" v-model="newDesc" placeholder="e.g. Salary, Coffee" />
        </div>
        <div class="form-row">
          <label for="amount">Amount</label>
          <input id="amount" type="number" step="0.01" v-model.number="newAmount" placeholder="0.00" />
        </div>
        <div class="form-row">
          <label for="type">Type</label>
          <select id="type" v-model="newType">
            <option value="income">Income</option>
            <option value="expense">Expense</option>
          </select>
        </div>
        <button type="submit" class="btn-primary">Add Transaction</button>
      </form>
    </section>

    <section class="panel summary-panel">
      <h2>Overview</h2>
      <div class="summary-grid">
        <div class="summary-card income-card">
          <span>Income</span>
          <strong>${{ totalIncome.toFixed(2) }}</strong>
        </div>
        <div class="summary-card expense-card">
          <span>Expenses</span>
          <strong>${{ totalExpenses.toFixed(2) }}</strong>
        </div>
        <div class="summary-card balance-card">
          <span>Balance</span>
          <strong>${{ balance.toFixed(2) }}</strong>
        </div>
        <div class="summary-card budget-card">
          <span>Budget Limit</span>
          <strong>${{ budgetLimit.toFixed(2) }}</strong>
        </div>
      </div>

      <div class="budget-bar">
        <div class="budget-track">
          <div class="budget-progress" :style="{
            width: expensePercentage + '%',
            backgroundColor: isOverBudget ? '#dc2626' : '#22c55e'
          }" />
        </div>
        <div class="budget-meta">
          <span>{{ expensePercentage }}% of budget used</span>
          <span>{{ budgetStatus }}</span>
        </div>
      </div>
    </section>

    <section class="panel filter-panel">
      <div class="filter-row">
        <label for="filterType">Filter</label>
        <select id="filterType" v-model="filterType">
          <option value="all">All</option>
          <option value="income">Income</option>
          <option value="expense">Expense</option>
        </select>
        <label for="budgetLimit">Budget Limit</label>
        <input id="budgetLimit" type="number" step="1" v-model.number="budgetLimit" min="0" />
      </div>
    </section>

    <section class="panel transactions-panel">
      <div class="transactions-header">
        <h2>Transactions</h2>
        <button class="btn-secondary" @click="clearAll" :disabled="transactions.length === 0">
          Clear All
        </button>
      </div>
      <div v-if="filteredTransactions.length === 0" class="empty-state">
        No transactions to display.
      </div>
      <ul class="transaction-list">
        <li v-for="transaction in filteredTransactions" :key="transaction.id" class="transaction-item">
          <div>
            <strong>{{ transaction.desc }}</strong>
            <p>{{ transaction.date }}</p>
          </div>
          <div class="transaction-right">
            <span :class="transaction.type === 'income' ? 'tag-income' : 'tag-expense'">
              {{ transaction.type }}
            </span>
            <strong>${{ transaction.amount.toFixed(2) }}</strong>
            <button class="icon-btn" @click="deleteTransaction(transaction.id)">Delete</button>
          </div>
        </li>
      </ul>
    </section>

    <section class="panel category-panel">
      <h2>Category Summary</h2>
      <div class="category-grid">
        <div v-for="item in categorySummary" :key="item.category" class="category-card">
          <span>{{ item.category }}</span>
          <strong>${{ item.total.toFixed(2) }}</strong>
          <small>{{ item.count }} transaction(s)</small>
        </div>
      </div>
    </section>

    <section class="panel notification-panel">
      <h2>Notifications</h2>
      <div v-if="warning" class="warning-box">{{ warning }}</div>
      <ul class="notification-log">
        <li v-for="(note, idx) in notifications" :key="idx">{{ note }}</li>
      </ul>
    </section>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'

interface Transaction {
  id: string
  desc: string
  amount: number
  type: 'income' | 'expense'
  date: string
}

const transactions = ref<Transaction[]>([])
const filterType = ref<'all' | 'income' | 'expense'>('all')
const budgetLimit = ref<number>(1000)
const newDesc = ref('')
const newAmount = ref<number | null>(null)
const newType = ref<'income' | 'expense'>('expense')
const notifications = ref<string[]>([])
const warning = ref('')

const saved = localStorage.getItem('finance-tracker-transactions')
if (saved) {
  try {
    const parsed = JSON.parse(saved) as Transaction[]
    transactions.value = Array.isArray(parsed) ? parsed : []
  } catch {
    transactions.value = []
  }
}

const filteredTransactions = computed(() => {
  if (filterType.value === 'all') {
    return transactions.value
  }
  return transactions.value.filter((tx) => tx.type === filterType.value)
})

const totalIncome = computed(() =>
  transactions.value.reduce((sum, tx) => (tx.type === 'income' ? sum + tx.amount : sum), 0)
)

const totalExpenses = computed(() =>
  transactions.value.reduce((sum, tx) => (tx.type === 'expense' ? sum + tx.amount : sum), 0)
)

const balance = computed(() => totalIncome.value - totalExpenses.value)

const isOverBudget = computed(() => totalExpenses.value > budgetLimit.value)

const expensePercentage = computed(() => {
  if (budgetLimit.value <= 0) {
    return 100
  }
  const percent = Math.round((totalExpenses.value / budgetLimit.value) * 100)
  return Math.min(100, Math.max(0, percent))
})

const categorySummary = computed(() => {
  const grouped = new Map<string, { total: number; count: number }>()
  transactions.value.forEach((tx) => {
    const label = tx.type === 'income' ? 'Income' : 'Expense'
    const current = grouped.get(label) || { total: 0, count: 0 }
    current.total += tx.amount
    current.count += 1
    grouped.set(label, current)
  })
  return Array.from(grouped, ([category, data]) => ({ category, ...data }))
})

const budgetStatus = computed(() => {
  if (totalExpenses.value === 0) {
    return 'No expenses recorded yet.'
  }
  if (isOverBudget.value) {
    const overBy = totalExpenses.value - budgetLimit.value
    return `Over budget by $${overBy.toFixed(2)}`
  }
  return `Within budget (${expensePercentage.value}% used)`
})

watch(
  transactions,
  (value) => {
    localStorage.setItem('finance-tracker-transactions', JSON.stringify(value))
  },
  { deep: true }
)

watch(balance, (current) => {
  if (current < 0) {
    warning.value = 'Warning: Your balance is negative. Review your expenses.'
  } else {
    warning.value = ''
  }
})

watch(isOverBudget, (current) => {
  if (current) {
    const message = `${new Date().toLocaleString()}: Budget exceeded. Expenses are $${totalExpenses.value.toFixed(2)} of $${budgetLimit.value.toFixed(2)}.`
    notifications.value.unshift(message)
    if (notifications.value.length > 6) {
      notifications.value.splice(6)
    }
  }
})

function addTransaction(desc: string, amount: number, type: 'income' | 'expense') {
  if (!desc.trim() || Number.isNaN(amount) || amount <= 0) {
    return
  }

  const newTransaction: Transaction = {
    id: `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
    desc: desc.trim(),
    amount: Number(amount),
    type,
    date: new Date().toLocaleDateString(),
  }

  transactions.value.push(newTransaction)
  newDesc.value = ''
  newAmount.value = null
  newType.value = 'expense'
}

function deleteTransaction(id: string) {
  transactions.value = transactions.value.filter((tx) => tx.id !== id)
}

function clearAll() {
  transactions.value = []
}

function submitTransaction() {
  if (newAmount.value === null) {
    return
  }
  addTransaction(newDesc.value, newAmount.value, newType.value)
}
</script>

<style scoped>
.app-container {
  max-width: 980px;
  margin: 0 auto;
  padding: 2rem;
  font-family: Inter, system-ui, sans-serif;
  color: #111827;
  background: #f8fafc;
}

.header {
  text-align: center;
  margin-bottom: 2rem;
}

.header h1 {
  font-size: clamp(2rem, 4vw, 2.8rem);
  margin-bottom: 0.4rem;
}

.panel {
  background: #ffffff;
  border-radius: 1rem;
  box-shadow: 0 16px 30px rgba(15, 23, 42, 0.08);
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.transaction-form,
.filter-row,
.transactions-header,
.budget-meta,
.category-grid,
.summary-grid {
  display: grid;
  gap: 1rem;
}

.transaction-form {
  grid-template-columns: 1fr;
}

.form-row {
  display: grid;
  gap: 0.5rem;
}

.form-row label {
  font-weight: 600;
}

.form-row input,
.form-row select {
  border: 1px solid #d1d5db;
  border-radius: 0.75rem;
  padding: 0.75rem 1rem;
  font-size: 1rem;
  background: #f8fafc;
}

.btn-primary,
.btn-secondary,
.icon-btn {
  border: none;
  border-radius: 999px;
  cursor: pointer;
}

.btn-primary {
  padding: 0.85rem 1.25rem;
  background: #2563eb;
  color: #ffffff;
  font-weight: 600;
}

.btn-secondary {
  padding: 0.75rem 1rem;
  background: #e2e8f0;
  color: #111827;
}

.icon-btn {
  margin-left: 1rem;
  padding: 0.45rem 0.75rem;
  background: #f1f5f9;
  color: #1f2937;
}

.summary-grid {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.summary-card {
  padding: 1rem;
  border-radius: 1rem;
  background: #f8fafc;
}

.income-card {
  border-left: 4px solid #22c55e;
}

.expense-card {
  border-left: 4px solid #ef4444;
}

.balance-card {
  border-left: 4px solid #2563eb;
}

.budget-card {
  border-left: 4px solid #f59e0b;
}

.budget-bar {
  margin-top: 1.5rem;
}

.budget-track {
  height: 1rem;
  background: #e2e8f0;
  border-radius: 999px;
  overflow: hidden;
}

.budget-progress {
  height: 100%;
  transition: width 0.3s ease;
}

.budget-meta {
  display: flex;
  justify-content: space-between;
  margin-top: 0.75rem;
  font-size: 0.95rem;
  color: #475569;
}

.filter-row {
  grid-template-columns: repeat(4, minmax(0, 1fr));
  align-items: end;
}

.transactions-panel .transactions-header {
  grid-template-columns: 1fr auto;
  align-items: center;
  margin-bottom: 1rem;
}

.transaction-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.transaction-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  border-bottom: 1px solid #e2e8f0;
}

.transaction-item:last-child {
  border-bottom: none;
}

.transaction-right {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.tag-income,
.tag-expense {
  padding: 0.25rem 0.65rem;
  border-radius: 999px;
  font-size: 0.85rem;
  font-weight: 700;
  text-transform: uppercase;
}

.tag-income {
  background: #dcfce7;
  color: #166534;
}

.tag-expense {
  background: #fee2e2;
  color: #991b1b;
}

.category-grid {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.category-card {
  padding: 1rem;
  background: #f8fafc;
  border-radius: 1rem;
}

.notification-panel {
  min-height: 6rem;
}

.warning-box {
  background: #fef2f2;
  color: #991b1b;
  padding: 1rem;
  border-radius: 0.85rem;
  margin-bottom: 1rem;
}

.notification-log {
  list-style: none;
  padding: 0;
  margin: 0;
}

.notification-log li {
  padding: 0.75rem 0;
  border-bottom: 1px solid #e2e8f0;
}

.empty-state {
  padding: 1rem;
  color: #64748b;
}

@media (max-width: 768px) {

  .summary-grid,
  .filter-row,
  .category-grid {
    grid-template-columns: 1fr;
  }
}
</style>
